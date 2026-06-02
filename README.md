# EspoCRM Docker Ops

Репозиторий содержит минимальный набор файлов для запуска EspoCRM в Docker во внутренней сети.

Рабочий адрес по умолчанию:

```text
http://10.0.2.7
```

Стенд рассчитан на работу во внутренней сети по обычному `http`.

Репозиторий:

```text
https://github.com/lazuale/espops
```

## Что входит в проект

| Файл или каталог | Назначение |
|---|---|
| `docker-compose.prod.yml` | рабочий стенд EspoCRM |
| `docker-compose.dev.yml` | тестовый стенд EspoCRM |
| `.env.prod.example` | пример настроек рабочего стенда |
| `.env.dev.example` | пример настроек тестового стенда |
| `scripts/backup-docker-container.sh` | официальный скрипт резервного копирования EspoCRM для Docker |
| `docs/` | подробные инструкции администратора |
| `.gitignore` | защита от случайного добавления секретов и резервных копий |
| `.gitattributes` | правила переносов строк для Linux-файлов |

В Git не должны попадать реальные настройки и резервные копии:

```text
.env.prod
.env.dev
backups/
restore-tmp/
*.tar.gz
*.sql
*.log
```

## Быстрый запуск рабочего стенда

Этот раздел нужен только для первого старта. Подробная установка с пояснениями описана в [`docs/01-install.md`](docs/01-install.md).

Раздел предполагает, что на сервере уже установлены Docker, Docker Compose и Git, а команды `docker` выполняются без `sudo`. Если сервер ещё не готов, начните с раздела «Подготовка сервера» в [`docs/01-install.md`](docs/01-install.md).

На сервере подготовить каталог и забрать проект:

```bash
sudo mkdir -p /opt/espocrm
sudo chown -R "$USER:$USER" /opt/espocrm
cd /opt/espocrm
git clone https://github.com/lazuale/espops.git .
```

Создать рабочие настройки:

```bash
cp .env.prod.example .env.prod
nano .env.prod
chmod 600 .env.prod
```

В `.env.prod` обязательно заменить пароли. Пароли писать в одинарных кавычках:

```env
ESPOCRM_DATABASE_PASSWORD='CHANGE_ME_LONG_RANDOM_DB_PASSWORD'
MARIADB_ROOT_PASSWORD='CHANGE_ME_LONG_RANDOM_ROOT_PASSWORD'
ESPOCRM_ADMIN_PASSWORD='CHANGE_ME_LONG_RANDOM_ADMIN_PASSWORD'
```

Проверить и запустить:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml config
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
```

Открыть в браузере:

```text
http://10.0.2.7
```

Если `docker compose config` завершился с ошибкой, запускать стенд нельзя. Сначала нужно исправить `.env.prod` или compose-файл.

## Документация

| Документ | Когда использовать |
|---|---|
| [`docs/00-glossary.md`](docs/00-glossary.md) | словарь терминов для тех, кто не работал с Docker |
| [`docs/01-install.md`](docs/01-install.md) | первый запуск рабочего стенда |
| [`docs/02-operations.md`](docs/02-operations.md) | повседневное обслуживание, логи, остановка и запуск |
| [`docs/03-backup-restore.md`](docs/03-backup-restore.md) | резервные копии и восстановление рабочего стенда |
| [`docs/04-dev-stand.md`](docs/04-dev-stand.md) | тестовый стенд и перенос рабочей копии в тестовый стенд |
| [`docs/05-update.md`](docs/05-update.md) | обновление файлов из GitHub и обновление версий образов |
| [`docs/06-checklists.md`](docs/06-checklists.md) | короткие контрольные списки перед важными действиями |
| [`docs/07-troubleshooting.md`](docs/07-troubleshooting.md) | типичные проблемы и порядок диагностики |

## Главное правило по данным

Данные EspoCRM хранятся не в каталоге проекта, а в Docker-томах.

| Том | Что хранит |
|---|---|
| `espocrm-prod_espocrm-db` | база данных MariaDB |
| `espocrm-prod_espocrm` | файлы EspoCRM, загрузки, настройки и доработки |

На рабочем стенде нельзя выполнять команды, которые удаляют тома:

```bash
docker compose down -v
docker volume prune
docker system prune --volumes
```

## Резервная копия

Минимальная команда для ручной резервной копии:

```bash
cd /opt/espocrm
mkdir -p backups
bash ./scripts/backup-docker-container.sh espocrm ./backups
```

Резервная копия EspoCRM не содержит `.env.prod`. Этот файл нужно хранить отдельно в защищённом месте.

Подробная инструкция по резервному копированию и восстановлению находится в [`docs/03-backup-restore.md`](docs/03-backup-restore.md).
