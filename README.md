# EspoCRM Docker Ops

Репозиторий для запуска EspoCRM в Docker во внутренней сети.

Рабочий адрес по умолчанию:

```text
http://10.0.2.7
```

Репозиторий:

```text
https://github.com/lazuale/espops
```

## Состав

```text
docker-compose.prod.yml       # рабочий стенд
docker-compose.dev.yml        # тестовый стенд
.env.prod.example             # пример настроек рабочего стенда
.env.dev.example              # пример настроек тестового стенда
.gitignore                    # исключения для Git
.gitattributes                # правила переносов строк
README.md                     # быстрый старт
ADMIN_GUIDE.md                # инструкция администратора
scripts/backup-docker-container.sh
```

В Git не должны попадать реальные настройки, резервные копии и временные файлы:

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

## Быстрый запуск рабочего стенда

На сервере:

```bash
sudo mkdir -p /opt
sudo chown -R "$USER:$USER" /opt
cd /opt
```

Клонировать репозиторий:

```bash
git clone https://github.com/lazuale/espops.git espocrm
cd /opt/espocrm
```

Создать рабочий файл настроек:

```bash
cp .env.prod.example .env.prod
nano .env.prod
chmod 600 .env.prod
```

Обязательно заменить пароли:

```env
ESPOCRM_DATABASE_PASSWORD=CHANGE_ME_LONG_RANDOM_DB_PASSWORD
MARIADB_ROOT_PASSWORD=CHANGE_ME_LONG_RANDOM_ROOT_PASSWORD
ESPOCRM_ADMIN_PASSWORD=CHANGE_ME_LONG_RANDOM_ADMIN_PASSWORD
```

Проверить конфигурацию:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml config
```

Запустить:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

Проверить контейнеры:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
```

Открыть в браузере:

```text
http://10.0.2.7
```

## Резервная копия

Официальный скрипт резервного копирования EspoCRM запускается с сервера из папки проекта:

```bash
cd /opt/espocrm
mkdir -p backups
bash ./scripts/backup-docker-container.sh espocrm ./backups
```

Результат:

```text
backups/YYYY-MM-DD_HHMMSS.tar.gz
```

Проверить состав архива:

```bash
tar tzf backups/YYYY-MM-DD_HHMMSS.tar.gz
```

Внутри должны быть:

```text
./
./db.tar.gz
./files.tar.gz
```

## Обновление файлов на сервере из GitHub

Перед обновлением сделать резервную копию:

```bash
cd /opt/espocrm
bash ./scripts/backup-docker-container.sh espocrm ./backups
```

Проверить, что на сервере нет ручных изменений файлов проекта:

```bash
git status --short
```

Забрать изменения:

```bash
git pull --ff-only
```

Проверить и применить:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml config
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
```

## Где лежат данные

Данные хранятся в томах Docker:

```text
espocrm-prod_espocrm-db  -> база MariaDB
espocrm-prod_espocrm     -> файлы EspoCRM, загрузки, настройки и доработки
```

Обычная остановка не удаляет данные:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml down
```

На рабочем стенде не выполнять:

```bash
docker compose down -v
docker volume prune
docker system prune --volumes
```

Подробная инструкция администратора: [`ADMIN_GUIDE.md`](ADMIN_GUIDE.md).
