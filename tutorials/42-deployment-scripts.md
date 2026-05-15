# Deployment Scripts Reference

## Overview

This directory contains scripts for deploying, managing, and maintaining the ticket-sell system
in a Docker-based production environment.

## Prerequisites

| Tool | Version | Purpose |
| ------ | --------- | --------- |
| **Docker** | 24+ | Container runtime |
| **Docker Compose** | v2.x+ | Multi-container orchestration |
| **Git** | Any | Source code management |
| **Python** | 3.12+ | Only needed for `init_prod_env.sh` (secret generation) |
| **bash** | 4+ | All shell scripts (Linux/macOS/Git Bash on Windows) |

## Quick Start

### First-time deployment

```bash
# 1. Clone the repository (if not already done)
git clone <your-repo-url> && cd ticket-sell

# 2. Initialize production environment (generates secure secrets)
bash scripts/init_prod_env.sh

# 3. Review and update .env (especially CORS_ORIGINS, ALIPAY_*)
#    Edit .env with your text editor

# 4. Deploy
bash scripts/deploy.sh --build
```text

### Subsequent deployments

```bash
# Pull latest code and redeploy
bash scripts/deploy.sh
```

## Scripts Reference

### `deploy.sh` — One-click production deployment

The primary deployment script that handles the full lifecycle:

```bash
bash scripts/deploy.sh [OPTIONS]

Options:
  --build               Force rebuild Docker images from source
  --no-cache            Rebuild with no Docker layer cache (implies --build)
  --skip-migrations     Skip database migrations
  --skip-pull           Skip git pull
  --help                Show help message
```text

**What it does:**

1. **Check prerequisites** — Verifies Docker, Docker Compose, and required files exist
2. **Validate .env** — Checks for required environment variables (`DB_PASSWORD`, `SECRET_KEY`)
3. **Pull latest code** — Git pull from remote repository
4. **Build/pull images** — Builds from source or pulls from registry
5. **Run migrations** — Starts MySQL/Redis, waits for health, runs `alembic upgrade head`
6. **Start services** — Launches all containers via `docker-compose.prod.yml`
7. **Health check** — Waits for nginx and app to respond to HTTP requests

### `deploy.bat` — Windows deployment

Windows batch counterpart with the same functionality:

```cmd
scripts\deploy.bat [OPTIONS]
```

### `init_prod_env.sh` — Environment initialization

Generates a production `.env` file with cryptographically secure random values:

```bash
bash scripts/init_prod_env.sh
bash scripts/init_prod_env.sh --force   # Non-interactive
```text

**Generated values:**

- `SECRET_KEY` —  64-char hex string
- `JWT_SECRET_KEY` — 64-char hex string
- `DB_PASSWORD` — 20-char alphanumeric
- `REDIS_PASSWORD` — 20-char alphanumeric

After running, manually update:

- `CORS_ORIGINS` — Set to your frontend domain(s)
- `ALIPAY_*` — Alipay credentials (if using payments)
- Database connection info if running MySQL/Redis remotely

### `backup.sh` — Database backup

Creates a timestamped MySQL dump:

```bash
bash scripts/backup.sh
bash scripts/backup.sh --no-compress   # Skip gzip
bash scripts/backup.sh --dry-run       # Preview only
```

**Features:**

- Dumps to `./backups/` directory
- Gzip compression by default
- Automatic cleanup of backups older than 7 days
- Integrity verification after backup

### `rollback.sh` — Deployment rollback

Rolls back to a previous Docker image tag:

```bash
bash scripts/rollback.sh                    # Rollback to "previous" tag
bash scripts/rollback.sh v1.2.3             # Rollback to specific tag
bash scripts/rollback.sh --list             # List available tags
```text

**How it works:**

1. Stops all running services (`docker compose down`)
2. Sets `TAG` environment variable to the rollback target
3. Pulls the specified image from registry
4. Restarts services with the older image
5. Runs a health check after rollback

### `healthcheck.py` — Service health check

Used for Docker HEALTHCHECK and manual diagnostics:

```bash
python scripts/healthcheck.py               # Full check (HTTP + DB + Redis)
python scripts/healthcheck.py --quick        # HTTP only
python scripts/healthcheck.py --check-db     # DB + Redis only
```

**Exit codes:** 0 = healthy, 1 = unhealthy

## Environment Variables

| Variable | Required | Default | Description |
| ---------- | ---------- | --------- | ------------- |
| `DB_PASSWORD` | **Yes** | — | MySQL root password |
| `SECRET_KEY` | **Yes** | — | Django-style secret key (min 32 chars) |
| `DB_HOST` | No | `localhost` | MySQL host |
| `DB_PORT` | No | `3306` | MySQL port |
| `DB_USER` | No | `ticket_app` | MySQL application user |
| `DB_NAME` | No | `ticket_prod` | MySQL database name |
| `REDIS_HOST` | No | `localhost` | Redis host |
| `REDIS_PORT` | No | `6379` | Redis port |
| `REDIS_PASSWORD` | No | `redispass123` | Redis password |
| `CORS_ORIGINS` | No | `http://localhost:5173` | Allowed CORS origins (comma-separated) |
| `JWT_SECRET_KEY` | No | (same as SECRET_KEY) | JWT signing key |
| `ALIPAY_APP_ID` | No | — | Alipay application ID |
| `ALIPAY_NOTIFY_URL` | No | — | Alipay async notification URL |
| `ALIPAY_RETURN_URL` | No | — | Alipay return URL |
| `TAG` | No | `latest` | Docker image tag |
| `REGISTRY` | No | `docker.io` | Docker registry URL |
| `IMAGE_NAME` | No | `ticket-api` | Docker image name |
| `FRONTEND_IMAGE_NAME` | No | `ticket-frontend` | Frontend Docker image name |

## Backup and Restore

### Creating a backup

```bash
bash scripts/backup.sh
# Creates: ./backups/ticket_prod_20240501_120000.sql.gz
```text

### Restoring from backup

```bash
# For Docker-based MySQL:
gunzip -c backups/ticket_prod_20240501_120000.sql.gz | docker exec -i ticket-mysql mysql -uroot -p"$DB_PASSWORD"

# For local MySQL:
gunzip -c backups/ticket_prod_20240501_120000.sql.gz | mysql -uroot -p"$DB_PASSWORD"
```

### Automated backups

Add to crontab for daily backups:

```bash
# Run daily at 3 AM
0 3 * * * cd /path/to/ticket-sell && bash scripts/backup.sh >> deploy-logs/backup-cron.log 2>&1
```text

## Monitoring

### Container logs

```bash
# Follow all logs
docker compose -f docker-compose.prod.yml logs -f

# Specific service logs
docker compose -f docker-compose.prod.yml logs -f app
docker compose -f docker-compose.prod.yml logs -f nginx

# Last N lines
docker compose -f docker-compose.prod.yml logs --tail=100 app
```

### Health checks

```bash
# Via nginx (entry point)
curl http://localhost/health

# Direct app health
curl http://localhost:8000/health

# Full diagnostic
python scripts/healthcheck.py
```text

### Container status

```bash
docker compose -f docker-compose.prod.yml ps
docker stats --no-stream
```

## Troubleshooting

### Container won't start

1. Check logs:

   ```bash
   docker compose -f docker-compose.prod.yml logs app
   ```text

2. Verify .env file is complete:

   ```bash
   bash scripts/deploy.sh --skip-pull   # Will validate .env
   ```

3. Check disk space:

   ```bash
   df -h
   docker system df
   ```text

### Database migration fails

1. Check MySQL is running:

   ```bash
   docker compose -f docker-compose.prod.yml ps mysql
   ```

2. Run migration manually:

   ```bash
   docker compose -f docker-compose.prod.yml run --rm app alembic upgrade head
   ```text

3. Check migration history:

   ```bash
   docker compose -f docker-compose.prod.yml run --rm app alembic history
   ```

### Nginx returns 502 Bad Gateway

1. Check app container is running:

   ```bash
   docker compose -f docker-compose.prod.yml ps app
   ```text

2. Check app logs for errors:

   ```bash
   docker compose -f docker-compose.prod.yml logs --tail=50 app
   ```

3. Verify nginx config:

   ```bash
   docker compose -f docker-compose.prod.yml exec nginx nginx -t
   ```text

### Port already in use

```bash
# Find what's using port 80 or 443
sudo lsof -i :80
sudo lsof -i :443

# Or on Windows (as admin):
netstat -ano | findstr :80
```

## Scaling Considerations

### Horizontal scaling (app instances)

To run multiple app replicas, modify `docker-compose.prod.yml`:

```yaml
app:
  deploy:
    replicas: 3
```text

Then update `nginx/conf.d/docker.conf` to use `least_conn` load balancing
(it already has `keepalive` configured). The upstream `api` block will
automatically discover multiple app containers via Docker DNS.

### Resource limits

Adjust resource limits in `docker-compose.prod.yml`:

```yaml
app:
  deploy:
    resources:
      limits:
        memory: 1G    # Increase from 512M
        cpus: "2.0"   # Increase from 1.0
```

### Database

- MySQL data persists in the `mysql_data` volume
- Consider using an external managed database (RDS, Cloud SQL) for production
- Enable slow query logging for performance tuning

### Redis

- AOF persistence is enabled for durability
- Consider Redis Sentinel or Cluster for high availability
- Monitor memory usage: `docker compose exec redis redis-cli INFO memory`
