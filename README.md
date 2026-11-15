# Xray2VPS — Скрипты автоматической установки Xray-core на VPS [^1]

Набор Bash-скриптов для быстрой автоматической установки и настройки прокси-сервера **Xray-core** (протокол **VLESS**) на чистый VPS под управлением Ubuntu / Debian.

Все скрипты автоматически включают алгоритм управления перегрузкой TCP **BBR**, генерируют ключи и идентификаторы, настраивают маршрутизацию (с блокировкой рекламы `geosite:category-ads-all`), создают консольные утилиты для управления пользователями и сразу после установки выводят готовую ссылку `vless://` и **QR-код** для подключения.

---

## Содержание

1. [Требования к серверу](#требования-к-серверу)
2. [Варианты установки (описание скриптов)](#варианты-установки-описание-скриптов)
   - [1. VLESS + TCP + REALITY (`install-tcp-xray`)](#1-vless--tcp--reality-install-tcp-xray)
   - [2. VLESS + XHTTP + REALITY (`install-xhttp-xray`)](#2-vless--xhttp--reality-install-xhttp-xray)
   - [3. VLESS + TCP + TLS Self-Steal с собственным доменом (`install-self-steal-xray`)](#3-vless--tcp--tls-self-steal-с-собственным-доменом-install-self-steal-xray)
3. [Сравнение вариантов](#сравнение-вариантов)
4. [Быстрая установка (в одну строку)](#быстрая-установка-в-одну-строку)
5. [Управление пользователями после установки](#управление-пользователями-после-установки)
6. [Расположение файлов и служб](#расположение-файлов-и-служб)

---

## Требования к серверу

- **ОС:** Ubuntu 20.04 / 22.04 / 24.04 или Debian 11 / 12 (используется пакетный менеджер `apt` и `systemd`).
- **Права доступа:** `root` (скрипты устанавливают пакеты, изменяют `/etc/sysctl.conf`, управляют службами `systemctl` и пишут в `/usr/local/`).
- **Свободные порты:**
  - Порт `443/tcp` (для всех вариантов).
  - Порт `80/tcp` и локальный порт `8080/tcp` (дополнительно для варианта `install-self-steal-xray` — для выпуска сертификата Let's Encrypt и работы Nginx).
- **Переменная окружения `domain`:** перед запуском любого из скриптов **обязательно** должна быть задана переменная `domain` (`export domain=...`), иначе скрипт завершит работу с ошибкой валидации.

---

## Варианты установки (описание скриптов)

### 1. VLESS + TCP + REALITY (`install-tcp-xray`)

Классическая и наиболее производительная конфигурация с маскировкой под чужой сайт через технологию **REALITY** (собственный домен и SSL-сертификаты не требуются).

- **Протокол:** `vless` (порт `443`)
- **Транспорт (network):** `tcp`
- **Поток (flow):** `xtls-rprx-vision`
- **Безопасность:** `reality` (отпечаток TLS в ссылках клиента: `fp=firefox`)
- **Роль переменной `domain`:** домен внешнего сайта-маскировки (SNI / `dest`), поддерживающего TLS 1.3 и H2 (например, `yahoo.com`, `www.microsoft.com` и т. п.).
- **Маршрутизация:** блокировка рекламных доменов (`geosite:category-ads-all`).

---

### 2. VLESS + XHTTP + REALITY (`install-xhttp-xray`)

Конфигурация с использованием современного транспорта **XHTTP (SplitHTTP)** поверх **REALITY**. Помогает обходить продвинутые системы DPI, которые ограничивают или отслеживают длительные одиночные TCP-соединения.

- **Протокол:** `vless` (порт `443`)
- **Транспорт (network):** `xhttp` (`path: "/"`, `mode: auto`)
- **Поток (flow):** не используется (`""`)
- **Безопасность:** `reality` (отпечаток TLS в ссылках клиента: `fp=firefox`)
- **Роль переменной `domain`:** домен внешнего сайта-маскировки (SNI / `target`).
- **Сниффинг:** `http`, `tls`, `quic`.
- **Маршрутизация:** блокировка рекламных доменов (`geosite:category-ads-all`) и IP-адресов Китая (`geoip:cn`).

---

### 3. VLESS + TCP + TLS Self-Steal с собственным доменом (`install-self-steal-xray`)

Конфигурация со **своим реальным доменом**, собственным TLS-сертификатом Let's Encrypt и локальным веб-сервером **Nginx** в качестве fallback-сайта. При обращении браузера или активного зондирования цензора к серверу на порт `443` без валидного VLESS-ключа Xray прозрачно перенаправляет запрос на локальный Nginx (`127.0.0.1:8080`), который отдает настоящий сайт из `/var/www/html`.

- **Протокол:** `vless` (порт `443`)
- **Транспорт (network):** `tcp`
- **Поток (flow):** `xtls-rprx-vision`
- **Безопасность:** `tls` (сертификат ECC `ec-256` от Let's Encrypt, `alpn: http/1.1`, `fingerprint: chrome`)
- **Fallback:** перенаправление не-VLESS трафика на порт `8080` (Nginx), а также редирект с `80` порта на `https://`.
- **DNS:** встроенный DoH-резолвер (`https+local://1.1.1.1/dns-query`, `localhost`).
- **Автопродление сертификата:** устанавливается `acme.sh` и создается скрипт `/usr/local/etc/xray/xray_cert/xray-cert-renew`, который автоматически добавляется в `crontab` (`0 1 1 * *` — каждое 1-е число месяца в 01:00).
- **Роль переменной `domain`:** **ваш собственный домен**, A-запись которого уже направлена на IP-адрес вашего VPS (без проксирования Cloudflare).

---

## Сравнение вариантов

| Параметр | `install-tcp-xray` | `install-xhttp-xray` | `install-self-steal-xray` |
| :--- | :--- | :--- | :--- |
| **Протокол / Поток** | VLESS (`xtls-rprx-vision`) | VLESS (без flow) | VLESS (`xtls-rprx-vision`) |
| **Транспорт** | `tcp` | `xhttp` (`path: /`) | `tcp` |
| **Шифрование / Маскировка** | `reality` (`fp=firefox`) | `reality` (`fp=firefox`) | `tls` Let's Encrypt (`fp=chrome`) + Fallback |
| **Нужен ли свой домен?** | Нет (указывается чужой SNI) | Нет (указывается чужой SNI) | **Да** (с A-записью на IP сервера) |
| **Доп. ПО** | `qrencode`, `curl`, `jq` | `qrencode`, `curl`, `jq` | `nginx`, `acme.sh`, `qrencode`, `curl`, `wget`, `jq` |

---

## Быстрая установка (в одну строку)

> **Важно:** Перед запуском команды обязательно замените значение в `export domain=...` на нужное доменное имя!

### Вариант 1: Установка VLESS + TCP + REALITY (`install-tcp-xray`)

Замените `example.com` на выбранный SNI-домен для маскировки:

```bash
export domain=example.com && wget -qO- https://raw.githubusercontent.com/vinser/xray2vps/refs/heads/main/install-tcp-xray | bash
```

*(Если ваш репозиторий на GitHub назван `hray2vps`, используйте соответствующий URL)*:
```bash
export domain=example.com && wget -qO- https://raw.githubusercontent.com/vinser/hray2vps/refs/heads/main/install-tcp-xray | bash
```

---

### Вариант 2: Установка VLESS + XHTTP + REALITY (`install-xhttp-xray`)

Замените `example.com` на выбранный SNI-домен для маскировки:

```bash
export domain=example.com && wget -qO- https://raw.githubusercontent.com/vinser/xray2vps/refs/heads/main/install-xhttp-xray | bash
```

*(Для репозитория `hray2vps`)*:
```bash
export domain=example.com && wget -qO- https://raw.githubusercontent.com/vinser/hray2vps/refs/heads/main/install-xhttp-xray | bash
```

---

### Вариант 3: Установка VLESS + TCP + TLS Self-Steal (`install-self-steal-xray`)

Предварительно направьте DNS A-запись вашего домена на IP-адрес VPS. Замените `my-domain.com` на **ваш реальный домен**:

```bash
export domain=my-domain.com && wget -qO- https://raw.githubusercontent.com/vinser/xray2vps/refs/heads/main/install-self-steal-xray | bash
```

*(Для репозитория `hray2vps`)*:
```bash
export domain=my-domain.com && wget -qO- https://raw.githubusercontent.com/vinser/hray2vps/refs/heads/main/install-self-steal-xray | bash
```

---

## Управление пользователями после установки

Все три скрипта создают в `/usr/local/bin/` набор глобальных CLI-команд, которые можно вызывать из любого места в терминале:

| Команда | Описание |
| :--- | :--- |
| `mainuser` | Выводит ссылку `vless://` и QR-код для основного пользователя (`main`), созданного при установке. |
| `newuser` | Интерактивно создает нового пользователя (запрашивает имя без пробелов), перезапускает Xray и сразу выводит ссылку и QR-код. |
| `rmuser` | Выводит нумерованный список клиентов и позволяет удалить выбранного пользователя по номеру. |
| `sharelink` | Выводит список существующих клиентов и генерирует ссылку подключения + QR-код для выбранного по номеру пользователя. |
| `userlist` | Выводит нумерованный список всех клиентов из конфигурации Xray. |

Памятка по этим командам также автоматически сохраняется в домашней директории пользователя: `cat ~/help`.

---

## Расположение файлов и служб

- **Конфигурация Xray:** `/usr/local/etc/xray/config.json`
- **Сгенерированные ключи, UUID, ShortID и домен:** `/usr/local/etc/xray/.keys`
- **TLS-сертификаты и скрипт обновления (только в `install-self-steal-xray`):**
  - Сертификат и ключ: `/usr/local/etc/xray/xray_cert/xray.crt` и `/usr/local/etc/xray/xray_cert/xray.key`
  - Скрипт автообновления: `/usr/local/etc/xray/xray_cert/xray-cert-renew`
  - Директория сайта-заглушки Nginx: `/var/www/html`
  - Конфигурация Nginx: `/etc/nginx/sites-available/default`
- **Управление службами:**
  - Перезапуск Xray: `systemctl restart xray`
  - Статус Xray: `systemctl status xray`
  - Перезапуск Nginx (для `install-self-steal-xray`): `systemctl restart nginx`

---

[^1]: Скрипты представляют собой модификацию оригинальных скриптов из репозиториев https://github.com/ServerTechnologies