# Инструкция администратора EspoCRM Docker Ops

## 1. Назначение

Документ описывает установку, запуск и обслуживание EspoCRM на сервере из GitHub-репозитория:

```text
https://github.com/lazuale/espops
```

Рабочий адрес по умолчанию:

```text
http://10.0.2.7
```

## 2. Состав проекта

В корне репозитория находятся файлы:

```text
docker-compose.prod.yml
docker-compose.dev.yml
.env.prod.example
.env.dev.example
.gitignore
.gitattributes
README.md
ADMIN_GUIDE.md
scripts/backup-docker-container.sh
```

Назначение файлов:

| Файл | Назначение |
|---|---|
| `docker-compose.prod.yml` | рабочий стенд EspoCRM |
| `docker-compose.dev.yml` | тестовый стенд EspoCRM |
| `.env.prod.example` | пример настроек рабочего стенда |
| `.env.dev.example` | пример настроек тестового стенда |
| `.gitignore` | защита от добавления секретов и резервных копий в Git |
| `.gitattributes` | правила переносов строк для Linux-скриптов и compose-файлов |
| `README.md` | быстрый старт |
| `ADMIN_GUIDE.md` | инструкция администратора |
| `scripts/backup-docker-container.sh` | официальный скрипт резервного копирования EspoCRM для Docker |

В Git не добавлять:

```text
.env.prod
.env.dev
backups/
backups-dev/
restore-tmp/
*.tar.gz
*.sql
*.log
```

## 3. Требования к серверу

На сервере должны быть установлены:

```text
Docker
Docker Compose plugin
Git
```

Проверка:

```bash
docker --version
docker compose version
git --version
```

Проверить свободное место:

```bash
df -h
```

Проверить, что нужные порты не заняты:

```bash
sudo ss -lntp | grep -E ':80|:8081' || true
```

По умолчанию рабочий стенд использует:

| Назначение | Порт |
|---|---:|
| Веб-интерфейс EspoCRM | `80` |
| WebSocket EspoCRM | `8081` |

## 4. Установка рабочего стенда из GitHub

### 4.1. Подготовить каталог

```bash
sudo mkdir -p /opt
sudo chown -R "$USER:$USER" /opt
cd /opt
```

### 4.2. Клонировать репозиторий

Основной вариант через HTTPS:

```bash
git clone https://github.com/lazuale/espops.git espocrm
```

Если на сервере настроен SSH-доступ к GitHub:

```bash
git clone git@github.com:lazuale/espops.git espocrm
```

Перейти в проект:

```bash
cd /opt/espocrm
```

Проверить подключённый репозиторий:

```bash
git remote -v
```

Ожидаемо:

```text
origin  https://github.com/lazuale/espops.git (fetch)
origin  https://github.com/lazuale/espops.git (push)
```

или:

```text
origin  git@github.com:lazuale/espops.git (fetch)
origin  git@github.com:lazuale/espops.git (push)
```

### 4.3. Создать рабочие настройки

```bash
cp .env.prod.example .env.prod
nano .env.prod
chmod 600 .env.prod
```

Проверить адреса:

```env
ESPOCRM_HTTP_PORT=80
ESPOCRM_WEBSOCKET_PORT=8081
ESPOCRM_SITE_URL=http://10.0.2.7
ESPOCRM_WEB_SOCKET_URL=ws://10.0.2.7:8081
```

Обязательно заменить пароли. Пароли писать в одинарных кавычках:

```env
ESPOCRM_DATABASE_PASSWORD='CHANGE_ME_LONG_RANDOM_DB_PASSWORD'
MARIADB_ROOT_PASSWORD='CHANGE_ME_LONG_RANDOM_ROOT_PASSWORD'
ESPOCRM_ADMIN_PASSWORD='CHANGE_ME_LONG_RANDOM_ADMIN_PASSWORD'
```

Почему так: Docker Compose применяет подстановку переменных к значениям без кавычек и к значениям в двойных кавычках. Символ `$` в пароле может быть воспринят как начало переменной. Одинарные кавычки передают значение буквально.

Нельзя:

```env
ESPOCRM_DATABASE_PASSWORD=P@ss$word2026
```

Можно:

```env
ESPOCRM_DATABASE_PASSWORD='P@ss$word2026'
```

Для простого и надёжного варианта можно генерировать пароли без спецсимволов:

```bash
openssl rand -hex 32
```

`ESPOCRM_ADMIN_PASSWORD` используется только при первой установке на пустой том Docker. Если EspoCRM уже установлена, изменение этого значения в `.env.prod` не меняет пароль существующего администратора.

### 4.4. Проверить конфигурацию

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml config
```

Если команда завершилась без ошибок, можно запускать.

### 4.5. Запустить рабочий стенд

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

Проверить состояние:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
```

Ожидаемые контейнеры:

```text
espocrm-db
espocrm
espocrm-daemon
espocrm-websocket
```

Открыть в браузере:

```text
http://10.0.2.7
```

Проверить порты с рабочего компьютера Windows:

```powershell
Test-NetConnection 10.0.2.7 -Port 80
Test-NetConnection 10.0.2.7 -Port 8081
```

В обоих случаях ожидается:

```text
TcpTestSucceeded : True
```

## 5. Как устроен рабочий стенд

Рабочий стенд состоит из четырёх контейнеров:

| Контейнер | Назначение |
|---|---|
| `espocrm-db` | база данных MariaDB |
| `espocrm` | основной веб-контейнер EspoCRM |
| `espocrm-daemon` | фоновые задания EspoCRM |
| `espocrm-websocket` | WebSocket для уведомлений и обновлений интерфейса |

Данные хранятся в двух томах Docker:

| Том Docker | Путь внутри контейнера | Что хранится |
|---|---|---|
| `espocrm-prod_espocrm-db` | `/var/lib/mysql` | база данных |
| `espocrm-prod_espocrm` | `/var/www/html` | файлы EspoCRM, загрузки, настройки, доработки |

Важные каталоги внутри `/var/www/html`:

```text
data/config.php
data/config-internal.php
data/upload/
custom/
client/custom/
```

Для полного восстановления нужны оба компонента:

1. база данных;
2. файлы `/var/www/html`.

Официальный скрипт резервного копирования сохраняет оба компонента в один архив.

## 6. Повседневное обслуживание

Все команды выполнять из каталога проекта:

```bash
cd /opt/espocrm
```

Посмотреть состояние:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
```

Посмотреть журналы основного контейнера:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml logs --tail=200 espocrm
```

Посмотреть журналы базы:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml logs --tail=200 espocrm-db
```

Посмотреть журналы фоновых заданий:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml logs --tail=200 espocrm-daemon
```

Посмотреть журналы WebSocket:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml logs --tail=200 espocrm-websocket
```

Остановить без удаления данных:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml down
```

Запустить после остановки:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

Пересоздать контейнеры без удаления данных:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d --force-recreate
```

## 7. Резервное копирование

### 7.1. Сделать резервную копию вручную

```bash
cd /opt/espocrm
mkdir -p backups
bash ./scripts/backup-docker-container.sh espocrm ./backups
```

Результат:

```text
backups/YYYY-MM-DD_HHMMSS.tar.gz
```

Проверить состав:

```bash
tar tzf backups/YYYY-MM-DD_HHMMSS.tar.gz
```

Ожидаемо:

```text
./
./db.tar.gz
./files.tar.gz
```

`db.tar.gz` содержит дамп базы. `files.tar.gz` содержит файлы EspoCRM из `/var/www/html`.

### 7.2. Автоматическая резервная копия через cron

Открыть cron текущего пользователя:

```bash
crontab -e
```

Добавить ежедневный запуск в 01:00:

```cron
0 1 * * * cd /opt/espocrm && bash ./scripts/backup-docker-container.sh espocrm ./backups >> ./backups/backup.log 2>&1
```

Проверить список заданий:

```bash
crontab -l
```

### 7.3. Очистка старых локальных копий

Пример удаления архивов старше 30 дней:

```bash
find /opt/espocrm/backups -type f -name "*.tar.gz" -mtime +30 -delete
```

Если нужно добавить очистку в cron:

```cron
30 1 * * * find /opt/espocrm/backups -type f -name "*.tar.gz" -mtime +30 -delete
```

Локальная копия на том же сервере не защищает от потери самого сервера. Важные архивы нужно копировать во внешнее защищённое место.

## 8. Восстановление из резервной копии

Восстановление выполнять в окно обслуживания. На время восстановления пользователи не должны работать в CRM.

### 8.1. Подготовить архив

```bash
cd /opt/espocrm
BACKUP=./backups/YYYY-MM-DD_HHMMSS.tar.gz
```

Распаковать служебные файлы:

```bash
rm -rf restore-tmp
mkdir restore-tmp
tar xzf "$BACKUP" -C restore-tmp
tar xzf restore-tmp/db.tar.gz -C restore-tmp
```

После этого должны быть файлы:

```text
restore-tmp/db.sql
restore-tmp/files.tar.gz
```

Проверить:

```bash
ls -lh restore-tmp
```

### 8.2. Считать параметры базы до остановки CRM

```bash
DB_CONTAINER=$(docker exec espocrm printenv ESPOCRM_DATABASE_HOST)
DB_NAME=$(docker exec espocrm printenv ESPOCRM_DATABASE_NAME)
DB_USER=$(docker exec espocrm printenv ESPOCRM_DATABASE_USER)
DB_PASS=$(docker exec espocrm printenv ESPOCRM_DATABASE_PASSWORD)
```

Проверить, что значения не пустые:

```bash
printf 'DB_CONTAINER=%s\nDB_NAME=%s\nDB_USER=%s\n' "$DB_CONTAINER" "$DB_NAME" "$DB_USER"
test -n "$DB_CONTAINER" && test -n "$DB_NAME" && test -n "$DB_USER" && test -n "$DB_PASS"
```

### 8.3. Остановить EspoCRM, но оставить базу

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml stop espocrm-daemon espocrm-websocket espocrm
```

Контейнер `espocrm-db` должен продолжать работать.

Проверить:

```bash
docker ps --format '{{.Names}}'
```

В списке должен быть:

```text
espocrm-db
```

### 8.4. Восстановить базу

```bash
cat restore-tmp/db.sql | docker exec -i "$DB_CONTAINER" \
  mariadb --user="$DB_USER" --password="$DB_PASS" "$DB_NAME"
```

### 8.5. Восстановить файлы

Очистить `/var/www/html` в томе EspoCRM:

```bash
docker run --rm --volumes-from espocrm alpine \
  sh -c 'rm -rf /var/www/html/* /var/www/html/.[!.]* /var/www/html/..?*'
```

Распаковать файлы из резервной копии:

```bash
cat restore-tmp/files.tar.gz | docker run --rm -i --volumes-from espocrm alpine \
  tar xzf - -C /var/www/html
```

### 8.6. Запустить CRM

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

Проверить:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
```

Открыть:

```text
http://10.0.2.7
```

### 8.7. Убрать временные файлы

```bash
rm -rf restore-tmp
```

## 9. Восстановление на новом сервере

Порядок:

1. Установить Docker, Docker Compose plugin и Git.
2. Клонировать репозиторий в `/opt/espocrm`.
3. Создать `.env.prod` с теми же параметрами базы, которые были на старом сервере.
4. Запустить пустой стенд, чтобы создались контейнеры и тома.
5. Выполнить восстановление по разделу 8.

Команды:

```bash
sudo mkdir -p /opt
sudo chown -R "$USER:$USER" /opt
cd /opt
git clone https://github.com/lazuale/espops.git espocrm
cd /opt/espocrm
cp .env.prod.example .env.prod
nano .env.prod
chmod 600 .env.prod
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

После запуска выполнить восстановление из раздела 8.

Важно: сохранить отдельно рабочий `.env.prod`. Обычная резервная копия EspoCRM не содержит этот файл.

## 10. Тестовый стенд

Тестовый стенд нужен для проверки запуска, обновлений и восстановления без риска для рабочего стенда.

Создать настройки:

```bash
cp .env.dev.example .env.dev
nano .env.dev
chmod 600 .env.dev
```

Запустить:

```bash
docker compose --env-file .env.dev -f docker-compose.dev.yml up -d
```

Открыть:

```text
http://localhost:8080
```

Если тестовый стенд нужен с другого компьютера, в `.env.dev` заменить `localhost` на IP сервера.

Остановить:

```bash
docker compose --env-file .env.dev -f docker-compose.dev.yml down
```

## 11. Проверка резервной копии на тестовом стенде

Резервная копия считается надёжной только после проверки восстановления.

Общий порядок:

1. Поднять тестовый стенд.
2. Убедиться, что он открывается.
3. Восстановить архив в тестовый стенд, используя те же шаги, что для рабочего стенда, но с контейнерами тестового стенда.
4. Проверить вход, карточки, вложения и основные разделы.

Для тестового стенда имена контейнеров:

| Рабочий стенд | Тестовый стенд |
|---|---|
| `espocrm` | `espocrm-dev` |
| `espocrm-db` | `espocrm-db-dev` |
| `espocrm-daemon` | `espocrm-daemon-dev` |
| `espocrm-websocket` | `espocrm-websocket-dev` |

При проверке восстановления на тестовом стенде заменить в командах:

```text
espocrm -> espocrm-dev
docker-compose.prod.yml -> docker-compose.dev.yml
.env.prod -> .env.dev
```

## 12. Обновление сервера из GitHub

Перед обновлением:

1. Сделать резервную копию.
2. Убедиться, что на сервере нет ручных изменений tracked-файлов.
3. Забрать изменения из GitHub.
4. Проверить compose-конфигурацию.
5. Применить изменения.

Команды:

```bash
cd /opt/espocrm
bash ./scripts/backup-docker-container.sh espocrm ./backups
git status --short
git pull --ff-only
docker compose --env-file .env.prod -f docker-compose.prod.yml config
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
```

Если `git status --short` показывает изменения в tracked-файлах, сначала разобраться с ними. На рабочем сервере не править `docker-compose.*.yml`, `README.md`, `ADMIN_GUIDE.md` напрямую без последующего commit в репозиторий.

## 13. Обновление версий EspoCRM или MariaDB

Перед обновлением версий:

1. Сделать резервную копию.
2. Проверить резервную копию на тестовом стенде.
3. Изменить версию образа сначала на тестовом стенде.
4. Проверить запуск и основные функции.
5. Только после этого менять рабочий стенд.

Не обновлять одновременно EspoCRM и MariaDB без необходимости.

Текущие версии задаются в `.env.prod` и `.env.dev`:

```env
ESPOCRM_IMAGE=espocrm/espocrm:9.3.7-apache-trixie
MARIADB_IMAGE=mariadb:11.4.11-noble
```

После изменения версии:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml pull
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

Проверить:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
```

## 14. Частые проверки

Проверить свободное место:

```bash
df -h
```

Проверить размеры Docker:

```bash
docker system df -v
```

Проверить список контейнеров:

```bash
docker ps
```

Проверить сайт с сервера:

```bash
curl -I http://10.0.2.7
```

Проверить WebSocket-порт с Windows:

```powershell
Test-NetConnection 10.0.2.7 -Port 8081
```

Проверить, куда подключён Git:

```bash
git remote -v
```

Проверить текущую ветку:

```bash
git branch --show-current
```

## 15. Что нельзя делать на рабочем стенде

Не выполнять:

```bash
docker compose down -v
docker volume prune
docker system prune --volumes
docker volume rm espocrm-prod_espocrm-db
docker volume rm espocrm-prod_espocrm
```

Не коммитить в Git:

```text
.env.prod
.env.dev
backups/
backups-dev/
restore-tmp/
*.tar.gz
*.sql
```

Не использовать в рабочем стенде плавающие версии образов:

```text
latest
lts
9
11
```

Использовать только точные теги образов.

Не вставлять GitHub token в команды вида:

```text
https://TOKEN@github.com/lazuale/espops.git
```

Token может попасть в историю shell или в Git-конфиг.

## 16. Чек-лист первого запуска

Перед первым запуском сервера:

- [ ] репозиторий склонирован в `/opt/espocrm`;
- [ ] создан `.env.prod`;
- [ ] заменены пароли;
- [ ] права на `.env.prod`: `600`;
- [ ] `docker compose config` проходит без ошибок;
- [ ] порты `80` и `8081` свободны.

После запуска:

- [ ] контейнеры запущены;
- [ ] сайт открывается по `http://10.0.2.7`;
- [ ] порт `8081` доступен;
- [ ] вход администратора работает;
- [ ] сделана первая резервная копия;
- [ ] копия вынесена за пределы сервера.

Перед обновлением из GitHub:

- [ ] сделана резервная копия;
- [ ] `git status --short` не показывает неожиданных изменений;
- [ ] выполнен `git pull --ff-only`;
- [ ] `docker compose config` проходит без ошибок;
- [ ] контейнеры перезапущены.
