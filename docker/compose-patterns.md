# Docker Compose Patterns

[🏠 Home](../README.md) > [🐳 Docker](./README.md)

## Essential Commands (Compose V2)

```bash
# Start all services in background
docker compose up -d

# Rebuild and start services
docker compose up -d --build

# Stop and remove containers (keeps named volumes)
docker compose down

# Stop and remove everything INCLUDING volumes (wipes DB data)
docker compose down -v

# Follow logs for a specific service
docker compose logs -f db
```

## Local Dev Setup (PostgreSQL + Redis)

Clean starter `docker-compose.yml` for local development with auto-init SQL scripts and persistent storage.

```yaml
services:
  db:
    image: postgres:17-alpine
    container_name: postgres_db
    restart: always
    environment:
      POSTGRES_USER: <user>
      POSTGRES_PASSWORD: <password>
      POSTGRES_DB: <db>
    ports:
      - "5432:5432"
    volumes:
      - ./database/init:/docker-entrypoint-initdb.d
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    container_name: redis_cache
    restart: always
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

## Quick Notes

* **Auto-init SQL:** Any `.sql` or `.sh` files inside `./database/init` run automatically **only once** when `pgdata` volume is first created. To re-run them, wipe the volume with `docker compose down -v`.
* **Redis Persistence:** `--appendonly yes` enables AOF persistence so cached data survives container restarts. Add `--requirepass yourpassword` if authentication is needed.
* **Service Hostnames:** Other containers in the same compose file connect using `db:5432` and `redis:6379`.
