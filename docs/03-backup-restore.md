# 03. Резервное копирование и восстановление

[Назад: обслуживание](02-operations.md) · [К содержанию](../README.md) · [Далее: тестовый стенд](04-dev-stand.md)

Резервная копия нужна для отката после ошибки, восстановления на новом сервере и подготовки тестового стенда. В Docker-установке EspoCRM нужно сохранять два компонента: базу данных и файлы `/var/www/html`.

## 1. Что входит в резервную копию

Официальный скрипт EspoCRM создаёт один архив.

| Файл внутри архива | Что содержит |
|---|---|
| `db.tar.gz` | дамп базы данных |
| `files.tar.gz` | файлы EspoCRM из `/var/www/html` |

Файл `.env.prod` в резервную копию EspoCRM не входит. Его нужно хранить отдельно в защищённом месте. Без `.env.prod` восстановить стенд на новом сервере будет сложнее: в нём находятся адреса, имена пользователей и пароли.

## 2. Зависимость от образа Alpine

Официальный скрипт резервного копирования и команды восстановления используют Docker-образ `alpine` для доступа к тому с файлами EspoCRM.

Если сервер имеет доступ к Docker Hub, образ обычно подтянется автоматически. Если сервер работает без доступа в интернет, заранее загрузите образ:

```bash
docker pull alpine
```

Проверить наличие образа:

```bash
docker image ls alpine
```

Если образа нет и сервер не может скачать его из интернета, резервное копирование или восстановление может остановиться на этапе работы с файлами.

## 3. Сделать резервную копию вручную

Эту процедуру можно выполнять без остановки рабочего стенда. Команды запускать на сервере из каталога проекта.

Для работы скрипта оба контейнера должны быть запущены: и `espocrm` (из него читаются параметры базы), и `espocrm-db` (из него снимается дамп). Если стенд остановлен, сначала поднимите его, иначе скрипт завершится с ошибкой «Container is not running».

```bash
cd /opt/espocrm
mkdir -p backups
bash ./scripts/backup-docker-container.sh espocrm ./backups
```

Дамп базы снимается с работающей CRM без блокировки. Для самой согласованной копии запускайте резервное копирование в момент низкой активности пользователей. Перед обновлениями и другими рискованными операциями копию лучше делать в окно обслуживания.

После выполнения появится архив вида:

```text
backups/YYYY-MM-DD_HHMMSS.tar.gz
```

Проверьте, что архив читается:

```bash
tar tzf backups/YYYY-MM-DD_HHMMSS.tar.gz
```

Ожидаемый состав:

```text
./
./db.tar.gz
./files.tar.gz
```

Если архив не читается командой `tar tzf`, такую копию нельзя использовать для восстановления.

## 4. Автоматическая резервная копия через cron

Для ежедневной локальной копии можно использовать cron. Локальная копия полезна для быстрого отката, но она не защищает от потери самого сервера. Важные архивы нужно копировать во внешнее защищённое место.

Перед настройкой проверьте, что `crontab` установлен:

```bash
command -v crontab
```

Если команда ничего не вывела, установите cron:

```bash
sudo apt update
sudo apt install -y cron
sudo systemctl enable --now cron
```

Открыть cron текущего пользователя:

```bash
crontab -e
```

Добавить ежедневный запуск в 01:00:

```cron
0 1 * * * cd /opt/espocrm && mkdir -p backups && bash ./scripts/backup-docker-container.sh espocrm ./backups >> ./backups/backup.log 2>&1
```

Слово `cron` в блоке выше — это только тип подсветки Markdown. В терминал его вводить не нужно.

Проверить список заданий:

```bash
crontab -l
```

## 5. Очистка старых локальных копий

Если локальные архивы не чистить, они постепенно займут диск. Пример удаления архивов старше 30 дней:

```bash
find /opt/espocrm/backups -type f -name "*.tar.gz" -mtime +30 -delete
```

Если нужно добавить очистку в cron:

```cron
30 1 * * * find /opt/espocrm/backups -type f -name "*.tar.gz" -mtime +30 -delete
```

Перед включением автоматической очистки убедитесь, что важные архивы копируются за пределы сервера.

## 6. Восстановление рабочего стенда

Восстановление полностью заменяет текущие данные рабочего стенда данными из архива. Выполнять его нужно в окно обслуживания, когда пользователи не работают в CRM.

Одна важная деталь: вместе с файлами из архива восстанавливается и `data/config-internal.php`, где записан пароль подключения к базе на момент создания копии. Если после той копии пароль базы в `.env.prod` меняли, после восстановления пароль в `config-internal.php` и пароль в `.env.prod` разойдутся, и CRM не подключится к базе. Проще всего восстанавливаться при том же пароле БД, что и в архиве. Если пароль с тех пор меняли, понадобится шаг выравнивания — он описан ниже в Шаге 6.

Перед началом проверьте три вещи:

| Проверка | Зачем |
|---|---|
| Есть архив backup | без него нечего восстанавливать |
| Есть актуальный `.env.prod` | backup EspoCRM его не содержит |
| Есть доступ к образу `alpine` | он нужен для восстановления файлов |

Шаги ниже используют переменные оболочки (`BACKUP`, `DB_CONTAINER` и другие). Выполняйте их в одной сессии терминала и без перерыва: при открытии нового терминала переменные теряются, и шаги нужно начинать сначала.

### Шаг 1. Подготовить архив

```bash
cd /opt/espocrm
BACKUP=./backups/YYYY-MM-DD_HHMMSS.tar.gz
rm -rf restore-tmp
mkdir restore-tmp
tar xzf "$BACKUP" -C restore-tmp
tar xzf restore-tmp/db.tar.gz -C restore-tmp
```

Проверить, что файлы на месте:

```bash
ls -lh restore-tmp
```

Должны быть:

```text
restore-tmp/db.sql
restore-tmp/files.tar.gz
```

Если этих файлов нет, остановиться и не продолжать восстановление.

### Шаг 2. Считать параметры базы до остановки CRM

Параметры берутся из контейнера EspoCRM. Поэтому сначала читаем их, и только потом останавливаем приложение.

В этом проекте адрес базы (`ESPOCRM_DATABASE_HOST`) совпадает с именем контейнера базы (`espocrm-db`), поэтому одну и ту же переменную `DB_CONTAINER` можно использовать и как хост для подключения, и как имя контейнера в `docker exec`.

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

Если проверка не прошла, остановиться. Без параметров БД восстановление продолжать нельзя.

### Шаг 3. Остановить приложение, но оставить базу

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml stop espocrm-daemon espocrm-websocket espocrm
```

База должна продолжать работать:

```bash
docker ps --format '{{.Names}}'
```

В списке должен быть контейнер:

```text
espocrm-db
```

### Шаг 4. Восстановить базу

```bash
cat restore-tmp/db.sql | docker exec -i "$DB_CONTAINER" \
  mariadb --user="$DB_USER" --password="$DB_PASS" "$DB_NAME"
```

Если команда завершилась с ошибкой, не запускать CRM. Сначала разобраться с ошибкой восстановления БД.

### Шаг 5. Восстановить файлы

Сначала очищаем файловый том EspoCRM:

```bash
docker run --rm --volumes-from espocrm alpine \
  sh -c 'rm -rf /var/www/html/* /var/www/html/.[!.]* /var/www/html/..?*'
```

Затем распаковываем файлы из backup:

```bash
cat restore-tmp/files.tar.gz | docker run --rm -i --volumes-from espocrm alpine \
  tar xzf - -C /var/www/html
```

### Шаг 6. Запустить CRM и проверить

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
```

Если в журнале `espocrm` появляются ошибки подключения к базе, скорее всего разошлись пароли: пароль в восстановленном `data/config-internal.php` (из архива) не совпадает с паролем базы. Сравните пароль в `.env.prod` с тем, что записан в восстановленном файле:

```bash
docker exec espocrm sh -c 'grep -i password /var/www/html/data/config-internal.php'
```

Если значение отличается от `ESPOCRM_DATABASE_PASSWORD` в `.env.prod`, приведите их в соответствие. Самый безопасный путь — задать в `.env.prod` тот же пароль, что и в архиве, и пересоздать контейнеры:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d --force-recreate
```

Открыть:

```text
http://10.0.2.7
```

Если после восстановления сайт не открывается, не работает вход или нет данных, разбор причин — в [`07-troubleshooting.md`](07-troubleshooting.md).

После проверки убрать временные файлы:

```bash
rm -rf restore-tmp
```

## 7. Восстановление на новом сервере

На новом сервере порядок такой же, но сначала нужно заново получить проект и создать `.env.prod`.

```bash
sudo mkdir -p /opt/espocrm
sudo chown -R "$USER:$USER" /opt/espocrm
cd /opt/espocrm
git clone https://github.com/lazuale/espops.git .
cp .env.prod.example .env.prod
nano .env.prod
chmod 600 .env.prod
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

После создания пустого стенда выполнить восстановление по разделу выше.

Важно: значения БД в `.env.prod` должны соответствовать тем значениям, которые используются при восстановлении. Сам backup EspoCRM не содержит `.env.prod`.

Причина в том, что имя базы, пользователь и пароль БД при первом старте берутся из `.env.prod` и записываются в файл `data/config-internal.php` внутри тома EspoCRM. Когда вы распаковываете файлы из backup, этот файл заменяется версией из копии и снова содержит «старые» параметры подключения. Поэтому на новом сервере проще всего задать в `.env.prod` те же `ESPOCRM_DATABASE_NAME`, `ESPOCRM_DATABASE_USER` и `ESPOCRM_DATABASE_PASSWORD`, что были на исходном стенде. Если значения отличаются, после восстановления нужно вручную привести их в соответствие в `data/config-internal.php`.

[Назад: обслуживание](02-operations.md) · [К содержанию](../README.md) · [Далее: тестовый стенд](04-dev-stand.md)