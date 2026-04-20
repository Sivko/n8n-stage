# n8n (Docker Compose)

Стек: PostgreSQL, Redis, MongoDB, n8n и n8n-worker в режиме очереди. Конфигурация — `docker-compose.yml`, секреты — в `.env` (файл не коммитится).

## Имена томов

Docker Compose создаёт **именованные тома** вида `<имя_проекта>_<имя_тома_в_compose>`. Имя проекта по умолчанию совпадает с именем каталога (например, `n8n`), поэтому тома часто называются:

- `n8n_postgres_data`
- `n8n_n8n_data`
- `n8n_redis_data`
- `n8n_mongo_data`

Проверить фактические имена:

```bash
docker volume ls
```

Если задана переменная `COMPOSE_PROJECT_NAME`, префикс будет другим.

---

## 1. Создание резервных копий

Рекомендуется **остановить контейнеры**, чтобы снимок файловых данных БД был согласованным:

```bash
docker compose down
```

Создайте каталог для архивов (в репозитории по умолчанию игнорируется Git — см. `.gitignore`):

```text
volume-backups/
```

Дальше — архивация каждого тома через временный контейнер с `alpine`. Подставьте **свои** имена томов из `docker volume ls` и путь к проекту на вашей машине.

**Windows (PowerShell), пример:**

```powershell
$backup = "c:/Users/limbo/Documents/projects/n8n/volume-backups"

docker run --rm -v n8n_postgres_data:/source:ro -v "${backup}:/backup" alpine tar czf /backup/postgres_data.tar.gz -C /source .
docker run --rm -v n8n_n8n_data:/source:ro -v "${backup}:/backup" alpine tar czf /backup/n8n_data.tar.gz -C /source .
docker run --rm -v n8n_redis_data:/source:ro -v "${backup}:/backup" alpine tar czf /backup/redis_data.tar.gz -C /source .
docker run --rm -v n8n_mongo_data:/source:ro -v "${backup}:/backup" alpine tar czf /backup/mongo_data.tar.gz -C /source .
```

**Linux / macOS (bash):**

```bash
BACKUP="$(pwd)/volume-backups"
mkdir -p "$BACKUP"

docker run --rm -v n8n_postgres_data:/source:ro -v "$BACKUP:/backup" alpine tar czf /backup/postgres_data.tar.gz -C /source .
docker run --rm -v n8n_n8n_data:/source:ro -v "$BACKUP:/backup" alpine tar czf /backup/n8n_data.tar.gz -C /source .
docker run --rm -v n8n_redis_data:/source:ro -v "$BACKUP:/backup" alpine tar czf /backup/redis_data.tar.gz -C /source .
docker run --rm -v n8n_mongo_data:/source:ro -v "$BACKUP:/backup" alpine tar czf /backup/mongo_data.tar.gz -C /source .
```

В каталоге `volume-backups/` появятся четыре файла `*.tar.gz`.

Дополнительно сохраните вне репозитория или в защищённом хранилище:

- копию `.env` (в том числе **`N8N_ENCRYPTION_KEY`** — без него зашифрованные данные n8n не восстановить);
- при необходимости — `docker-compose.yml` и инструкции по домену/прокси.

После бэкапа снова запустите стек:

```bash
docker compose up -d
```

---

## 2. Разворачивание (восстановление из архивов)

### Требования

- Установлены Docker и Docker Compose.
- В каталоге проекта лежат актуальные `docker-compose.yml` и `.env` с **теми же** учётными данными БД/Redis/Mongo и **`N8N_ENCRYPTION_KEY`**, что использовались при создании бэкапа (иначе n8n и приложения не смогут корректно прочитать данные).

### Шаги

1. Остановите и удалите контейнеры. Чтобы **заменить** данные в томах, старые тома нужно удалить (это уничтожит текущее содержимое томов на машине):

   ```bash
   docker compose down
   docker volume rm n8n_postgres_data n8n_n8n_data n8n_redis_data n8n_mongo_data
   ```

   Имена томов должны совпадать с теми, что вы видите в `docker volume ls` для этого проекта.

2. Создайте пустые тома с теми же именами:

   ```bash
   docker volume create n8n_postgres_data
   docker volume create n8n_n8n_data
   docker volume create n8n_redis_data
   docker volume create n8n_mongo_data
   ```

3. Распакуйте каждый архив в соответствующий том (путь к `volume-backups` замените на свой).

   **PowerShell:**

   ```powershell
   $backup = "c:/Users/limbo/Documents/projects/n8n/volume-backups"

   docker run --rm -v n8n_postgres_data:/target -v "${backup}:/backup" alpine sh -c "cd /target && tar xzf /backup/postgres_data.tar.gz"
   docker run --rm -v n8n_n8n_data:/target -v "${backup}:/backup" alpine sh -c "cd /target && tar xzf /backup/n8n_data.tar.gz"
   docker run --rm -v n8n_redis_data:/target -v "${backup}:/backup" alpine sh -c "cd /target && tar xzf /backup/redis_data.tar.gz"
   docker run --rm -v n8n_mongo_data:/target -v "${backup}:/backup" alpine sh -c "cd /target && tar xzf /backup/mongo_data.tar.gz"
   ```

   **bash:**

   ```bash
   BACKUP="$(pwd)/volume-backups"

   docker run --rm -v n8n_postgres_data:/target -v "$BACKUP:/backup" alpine sh -c "cd /target && tar xzf /backup/postgres_data.tar.gz"
   docker run --rm -v n8n_n8n_data:/target -v "$BACKUP:/backup" alpine sh -c "cd /target && tar xzf /backup/n8n_data.tar.gz"
   docker run --rm -v n8n_redis_data:/target -v "$BACKUP:/backup" alpine sh -c "cd /target && tar xzf /backup/redis_data.tar.gz"
   docker run --rm -v n8n_mongo_data:/target -v "$BACKUP:/backup" alpine sh -c "cd /target && tar xzf /backup/mongo_data.tar.gz"
   ```

4. Запустите сервисы:

   ```bash
   docker compose up -d
   ```

### Перенос на другой компьютер

Скопируйте каталог проекта вместе с `volume-backups/*.tar.gz` и своим `.env`, затем выполните шаги из раздела «Разворачивание» на новой машине. При первом `docker compose up` тома не должны уже существовать с «чужими» данными — либо удалите старые тома и восстановите из архивов, либо используйте новое имя проекта и скорректируйте имена томов в командах.

---

## Краткая справка

| Действие        | Команда (из корня проекта)   |
|----------------|------------------------------|
| Запуск         | `docker compose up -d`       |
| Остановка      | `docker compose down`        |
| Список томов   | `docker volume ls`           |

При сбоях после восстановления проверьте логи: `docker compose logs -f` и логи отдельных сервисов (`n8n`, `postgres` и т.д.).
