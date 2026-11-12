# 💾 Backup & Restore

[🏠 Home](../README.md) > [🐘 PostgreSQL](./README.md)

## 📤 Export (`pg_dump`)

```bash
# Custom compressed format (Recommended — supports parallel restore & selective tables)
pg_dump -U postgres -h localhost -F c -d my_db -f backup.dump

# Plain SQL file (human-readable)
pg_dump -U postgres -h localhost -d my_db > backup.sql

# Schema only (no data)
pg_dump -U postgres -s -d my_db > schema.sql

# Dump from a running Docker container
docker exec -t postgres_container pg_dump -U postgres -F c my_db > backup.dump
```

## 📥 Import (`pg_restore` & `psql`)

```bash
# Restore from custom .dump file (--clean drops existing objects first)
pg_restore -U postgres -h localhost -d my_db --clean --if-exists backup.dump

# Restore from plain .sql file
psql -U postgres -h localhost -d my_db < backup.sql

# Restore into a Docker container
cat backup.dump | docker exec -i postgres_container pg_restore -U postgres -d my_db --clean --if-exists
```
