# Immich Docker Compose Stack

Self-contained Docker Compose deployment for [Immich](https://immich.app) — a self-hosted photo and video management solution.

Based on the official Immich release **v2.7.5**.

## What's in the stack

| Container              | Purpose                                              |
|------------------------|------------------------------------------------------|
| `immich_server`        | Main Immich server (web UI + API)                    |
| `immich_machine_learning` | ML models for facial recognition, search, etc.    |
| `immich_redis`         | In-memory cache (Valkey, Redis-compatible)           |
| `immich_postgres`      | PostgreSQL 14 with vector extensions for smart search|

> **Note on Redis:** Immich requires a Redis-compatible cache internally. This stack uses Valkey (a lightweight Redis fork). It runs automatically — no configuration needed on your end.

## Prerequisites

- Ubuntu 22.04+ (or any Linux with Docker support)
- Docker Engine 24+ with Docker Compose v2
- At least 4 GB RAM (ML features use ~2 GB)
- Sufficient disk space for your photo library

## Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/anirudhatalmale6-alt/immich-docker.git
cd immich-docker

# 2. Create your .env from the example
cp .env.example .env

# 3. Edit .env — at minimum, change DB_PASSWORD
nano .env

# 4. Start the stack
docker compose up -d

# 5. Open the web UI
# http://<your-server-ip>:2283
```

On first launch, Immich will ask you to create an admin account.

## Configuration

All configurable values are in `.env`. The important ones:

| Variable            | What it does                               | Default       |
|---------------------|--------------------------------------------|---------------|
| `UPLOAD_LOCATION`   | Host path for photo/video storage          | `./library`   |
| `DB_DATA_LOCATION`  | Host path for PostgreSQL data              | `./postgres`  |
| `DB_PASSWORD`       | PostgreSQL password (alphanumeric only)    | —             |
| `IMMICH_VERSION`    | Immich version tag                         | `v2`          |
| `IMMICH_PORT`       | Host port for the web UI                   | `2283`        |
| `TZ`                | Timezone                                   | Server default|

For production, use absolute paths for `UPLOAD_LOCATION` and `DB_DATA_LOCATION` (e.g., `/mnt/data/immich/library`).

## Stop / Start / Restart

```bash
# Stop all containers (data is preserved)
docker compose down

# Start again
docker compose up -d

# Restart a specific service
docker compose restart immich-server

# View logs
docker compose logs -f              # all services
docker compose logs -f immich-server # just the server
```

## Backup

### What to back up

1. **Photo library** — the directory at `UPLOAD_LOCATION`
2. **Database** — PostgreSQL data

### Database backup (recommended method)

```bash
# Dump the database to a SQL file
docker exec -t immich_postgres pg_dumpall -c -U postgres > immich-db-backup.sql
```

### Database restore

```bash
# Stop Immich first
docker compose down

# Remove existing database data
rm -rf ./postgres

# Start only the database
docker compose up -d database

# Wait a few seconds for Postgres to initialize, then restore
sleep 5
docker exec -i immich_postgres psql -U postgres -d immich < immich-db-backup.sql

# Start everything
docker compose up -d
```

### Full backup script (cron-friendly)

```bash
#!/bin/bash
BACKUP_DIR="/mnt/backups/immich/$(date +%Y-%m-%d)"
mkdir -p "$BACKUP_DIR"

# Database
docker exec -t immich_postgres pg_dumpall -c -U postgres > "$BACKUP_DIR/db.sql"

# Library (rsync for incremental)
rsync -a ./library/ "$BACKUP_DIR/library/"

echo "Backup complete: $BACKUP_DIR"
```

## Upgrade

```bash
# Pull latest images
docker compose pull

# Recreate containers with new images
docker compose up -d

# Verify
docker compose ps
docker compose logs -f immich-server
```

If you pinned `IMMICH_VERSION` to a specific tag (e.g., `v2.7.5`), update the value in `.env` before pulling.

To roll back, change `IMMICH_VERSION` to the previous version and run the same commands.

## Hardware Acceleration (Optional)

For faster video transcoding or ML inference, Immich supports NVIDIA, Intel QuickSync, AMD ROCm, and more. See the official docs:

- [Transcoding acceleration](https://docs.immich.app/features/hardware-transcoding)
- [ML acceleration](https://docs.immich.app/features/ml-hardware-acceleration)

## Troubleshooting

```bash
# Check if all containers are healthy
docker compose ps

# Check a specific container's logs
docker compose logs immich-server --tail 50

# Restart everything fresh (keeps data)
docker compose down && docker compose up -d

# Nuclear option: rebuild from scratch (DELETES ALL DATA)
# docker compose down -v && rm -rf ./library ./postgres
```

## Resources

- [Immich Documentation](https://docs.immich.app)
- [Immich GitHub](https://github.com/immich-app/immich)
- [Environment Variables Reference](https://docs.immich.app/install/environment-variables)
