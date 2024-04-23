### Пошаговая инструкция

---
#### Структура:

- dumps
- - postgres
- - - 2024-04-21T11:00:00_pmp_postgres.sql
- - - ...
- - - 2024-04-23T11:00:00_pmp_postgres.sql
- .env
- db_dumper.sh

---
#### Файл .env
```dotenv
### Container

POSTGRES_CONTAINER=pmp_postgres
POSTGRES_DUMP_PATH=dumps/postgres


### Database

POSTGRES_HOST=db
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=qwerty
POSTGRES_DB=postgres
```

---
#### Файл docker-compose.yml
```bash
version: '3.9'

services:
  postgres:
    image: postgres
    env_file:
      - ./.env
    container_name: ${POSTGRES_CONTAINER}
    environment:
      POSTGRES_HOST: ${POSTGRES_HOST}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "${POSTGRES_PORT}:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
    restart: always

volumes:
  pg_data: {}
```

---
#### Файл db_dumper.sh [ Сохранение из дампа ]
```bash
# Подключение .env файла при наличии
if [ -f .env ]; then
    source .env
fi

# Определим название файла с будущим дампом
TIMESTAMP=$(date +"%Y-%m-%dT%H-%M-%S")
DUMP_PATH=${POSTGRES_DUMP_PATH}
DUMP_FILE=${TIMESTAMP}_${POSTGRES_CONTAINER}.sql

# Проверим доступность всех необходимых параметров окружения
if [ -z "$POSTGRES_USER" ] || [ -z "$POSTGRES_PASSWORD" ] || [ -z "$POSTGRES_DB" ]; then
    echo "Database credentials not set. Please set $POSTGRES_USER, $POSTGRES_PASSWORD and POSTGRES_DB in .env file."
    exit 1
fi

# Создадим дамп
docker exec -i "${POSTGRES_CONTAINER}" /bin/bash -c "PGPASSWORD=${POSTGRES_PASSWORD} pg_dump -U ${POSTGRES_USER} ${POSTGRES_DB}" > "${DUMP_PATH}"/"${DUMP_FILE}"

# Лог успешного выполнения
echo "Postgres dump file saved as $DUMP_PATH/$DUMP_FILE"
```

---
#### Файл db_recoverer.sh [ Восстановление из дампа ]
```bash
# Подключение .env файла при наличии
if [ -f .env ]; then
    source .env
fi

# Определим полный путь до файла с дампом
DUMP_PATH=${POSTGRES_DUMP_PATH}
DUMP_FILE="$1"

# Файл дампа для восстановления должен передаваться при вызове скрипта
if [ -z "$1" ]; then
    echo "Usage: $0 <dump_file>"
    exit 1
fi

# Проверим, существует ли файл дампа
if [ ! -f "${DUMP_PATH}"/"${DUMP_FILE}" ]; then
    echo "Dump file not found: $DUMP_PATH/$DUMP_FILE"
    exit 1
fi

# Проверим доступность всех необходимых параметров окружения
if [ -z "$POSTGRES_USER" ] || [ -z "$POSTGRES_PASSWORD" ] || [ -z "$POSTGRES_DB" ]; then
    echo "Database credentials not set. Please set $POSTGRES_USER, $POSTGRES_PASSWORD and POSTGRES_DB in .env file."
    exit 1
fi

# Восстановим образ из дампа
docker exec -i "${POSTGRES_CONTAINER}" /bin/bash -c "PGPASSWORD=${POSTGRES_PASSWORD} psql -U ${POSTGRES_USER} ${POSTGRES_DB}" < "$DUMP_PATH/${DUMP_FILE}"

# Лог выполнения
if [ $? -ne 0 ]; then
    echo "Error recovering from dump"
else
    echo "Postgres image was recovered from dump file $DUMP_PATH/$DUMP_FILE"
fi
```

---
#### Настроить права на файловую систему
```bash
# права на выполнение скрипта
chmod +x db_dumper.sh

# права на сохранение дампов
chmod +w -R dumps/postgres
```

---
Команды
```bash
# Запуск докера
docker-compose up -d

# Создание дампа
./db_dumper.sh

# Восстановление из дампа
./db_recoverer.sh {DUMP_NAME}
# Пример:
./db_recoverer.sh 2024-04-23T13-52-14_pmp_postgres.sql
```
---
