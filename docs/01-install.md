# 01. Установка рабочего стенда

[Назад: глоссарий](00-glossary.md) · [К содержанию](../README.md) · [Далее: обслуживание](02-operations.md)

Этот документ описывает первый запуск рабочего стенда EspoCRM на сервере. После выполнения инструкции должны работать четыре контейнера: база данных, веб-контейнер EspoCRM, фоновый обработчик и WebSocket.

Если незнакомы термины «контейнер», «образ», «том» — загляните в [`00-glossary.md`](00-glossary.md), там короткие определения.

Рабочий стенд по умолчанию открывается по адресу:

```text
http://10.0.2.7
```

## Подготовка сервера

Этот раздел нужен, если сервер ещё не готов. Если Docker, Docker Compose и Git уже установлены (это проверяется в следующем разделе), переходите сразу к нему.

Для работы нужен Linux-сервер (инструкция рассчитана на Ubuntu или Debian), доступ к нему по SSH и возможность выполнять команды через `sudo`.

Установить Docker вместе с плагином Docker Compose проще всего официальным скриптом. Скрипт стоит сначала посмотреть, прежде чем запускать:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

Включить автозапуск Docker при загрузке сервера:

```bash
sudo systemctl enable --now docker
```

Вся остальная документация выполняет команды `docker` и `docker compose` без `sudo`. Чтобы это работало, добавьте своего пользователя в группу `docker` и перезайдите в сессию (выйти и снова зайти по SSH):

```bash
sudo usermod -aG docker "$USER"
```

Установить Git:

```bash
sudo apt update
sudo apt install -y git
```

Для других дистрибутивов и для установки по официальному репозиторию (рекомендуется для постоянной эксплуатации) смотрите официальную документацию: `https://docs.docker.com/engine/install/`.

## Перед началом

Сначала нужно убедиться, что сервер готов к запуску Docker-стенда. Это лучше сделать до копирования настроек, чтобы не искать проблему уже после запуска.

| Что проверить | Команда | Что должно быть |
|---|---|---|
| Docker | `docker --version` | выводится версия Docker |
| Docker Compose plugin | `docker compose version` | выводится версия Compose |
| Git | `git --version` | выводится версия Git |
| Docker без sudo | `docker ps` | команда работает без `sudo` и без ошибки прав |
| Место на диске | `df -h` | достаточно места на разделе с Docker |
| Порты `80` и `8081` | `sudo ss -lntp \| grep -E ':80\|:8081' \|\| true` | порты свободны или понятно, кто их занял |

Если какая-то из проверок не проходит, вернитесь к разделу «Подготовка сервера». Если `docker ps` требует `sudo` или выдаёт ошибку прав, значит пользователь ещё не в группе `docker` или сессия не переоткрыта после добавления.

Если порт `80` занят, нужно либо освободить его, либо поменять `ESPOCRM_HTTP_PORT` в `.env.prod`. Если порт `8081` занят, аналогично поменять `ESPOCRM_WEBSOCKET_PORT`.

## 1. Получить проект из GitHub

Проект размещается в `/opt/espocrm`. Этот путь используется во всех дальнейших инструкциях.

```bash
sudo mkdir -p /opt/espocrm
sudo chown -R "$USER:$USER" /opt/espocrm
cd /opt/espocrm
git clone https://github.com/lazuale/espops.git .
```

Проверьте, что каталог действительно связан с нужным репозиторием:

```bash
git remote -v
```

Ожидаемый результат:

```text
origin  https://github.com/lazuale/espops.git (fetch)
origin  https://github.com/lazuale/espops.git (push)
```

Если на сервере настроен SSH-доступ к GitHub, можно использовать SSH-адрес (из того же каталога `/opt/espocrm`):

```bash
git clone git@github.com:lazuale/espops.git .
```

## 2. Создать рабочий файл настроек

Файл `.env.prod.example` — это шаблон из репозитория. Реальный `.env.prod` создаётся на сервере и не попадает в Git.

```bash
cp .env.prod.example .env.prod
nano .env.prod
chmod 600 .env.prod
```

В `.env.prod` нужно проверить адреса и порты:

| Переменная | Значение по умолчанию | Назначение |
|---|---|---|
| `ESPOCRM_HTTP_PORT` | `80` | порт веб-интерфейса |
| `ESPOCRM_WEBSOCKET_PORT` | `8081` | порт WebSocket |
| `ESPOCRM_SITE_URL` | `http://10.0.2.7` | адрес CRM для пользователей |
| `ESPOCRM_WEB_SOCKET_URL` | `ws://10.0.2.7:8081` | адрес WebSocket для браузера |

## 3. Заменить пароли

Пароли в `.env.prod` обязательно заменить. Писать их нужно в одинарных кавычках.

```env
ESPOCRM_DATABASE_PASSWORD='CHANGE_ME_LONG_RANDOM_DB_PASSWORD'
MARIADB_ROOT_PASSWORD='CHANGE_ME_LONG_RANDOM_ROOT_PASSWORD'
ESPOCRM_ADMIN_PASSWORD='CHANGE_ME_LONG_RANDOM_ADMIN_PASSWORD'
```

Одинарные кавычки нужны, чтобы символ `$` и другие спецсимволы не сломали подстановку переменных Docker Compose.

Внутри самого пароля одинарную кавычку использовать нельзя — она закроет строку и сломает значение. Если пароль генерируется, выбирайте символы без одинарной кавычки.

Правильно:

```env
ESPOCRM_DATABASE_PASSWORD='P@ss$word2026'
```

Неправильно:

```env
ESPOCRM_DATABASE_PASSWORD=P@ss$word2026
```

Если нужен простой безопасный вариант без спецсимволов, можно сгенерировать пароль так:

```bash
openssl rand -hex 32
```

`ESPOCRM_ADMIN_PASSWORD` используется только при первой установке на пустой Docker-том. Если EspoCRM уже установлена, изменение этого значения в `.env.prod` не меняет пароль существующего администратора.

## 4. Проверить compose-конфигурацию

Перед запуском нужно проверить, что Docker Compose видит все переменные и может собрать итоговую конфигурацию.

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml config
```

Если команда завершилась без ошибок, можно запускать стенд.

Если команда завершилась с ошибкой, запускать нельзя. Нужно исправить `.env.prod` или compose-файл и повторить проверку.

## 5. Запустить рабочий стенд

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

Первый запуск идёт дольше обычного. Сначала инициализируется база MariaDB, затем контейнер `espocrm` выполняет установку и создаёт `data/config.php`. Только после этого стартуют `espocrm-daemon` и `espocrm-websocket`, потому что они ждут готовности `espocrm` (в compose настроен healthcheck с запасом до двух минут). Это нормально: если сразу после `up -d` сайт ещё не открывается, нужно подождать и посмотреть журнал.

После запуска проверьте состояние контейнеров:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
```

В колонке `STATUS` у `espocrm` и `espocrm-db` сначала может быть `health: starting`, затем должно стать `healthy`. Дождитесь готовности `espocrm`. Если запуск затянулся, посмотрите журнал:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml logs -f espocrm
```

Ожидаемые контейнеры:

| Контейнер | Назначение |
|---|---|
| `espocrm-db` | база данных MariaDB |
| `espocrm` | основной веб-контейнер EspoCRM |
| `espocrm-daemon` | фоновые задания EspoCRM |
| `espocrm-websocket` | WebSocket для уведомлений и обновлений интерфейса |

## 6. Проверить доступ

С рабочего компьютера открыть:

```text
http://10.0.2.7
```

Проверить порты с Windows можно так:

```powershell
Test-NetConnection 10.0.2.7 -Port 80
Test-NetConnection 10.0.2.7 -Port 8081
```

В обоих случаях ожидается:

```text
TcpTestSucceeded : True
```

После того как сайт открылся, войдите под администратором. Логин задан в `.env.prod` в переменной `ESPOCRM_ADMIN_USERNAME` (по умолчанию `admin`), пароль — в `ESPOCRM_ADMIN_PASSWORD`. Эти значения используются только при первой установке на пустой том; сменить пароль позже можно из интерфейса EspoCRM.

Если сайт не открывается или открывается с ошибкой, не нужно повторно запускать установку. Сначала загляните в [`07-troubleshooting.md`](07-troubleshooting.md) — там разобраны типичные причины.

## После установки

Сразу после первого успешного запуска нужно сделать первую резервную копию:

```bash
cd /opt/espocrm
mkdir -p backups
bash ./scripts/backup-docker-container.sh espocrm ./backups
```

Подробно резервное копирование описано в [`03-backup-restore.md`](03-backup-restore.md).

[Назад: глоссарий](00-glossary.md) · [К содержанию](../README.md) · [Далее: обслуживание](02-operations.md)
