# n8n Suite - Deployment Guide

Self-hosted production ready n8n with PostgreSQL, external Python Task Runners.

## Prerequisites

- Docker and Docker Compose installed
- Git (optional, for version control)

## Project Structure

| File | Purpose |
|---|---|
| `docker-compose.yml` | Defines the PostgreSQL, n8n, and task-runners services |
| `Dockerfile.runners` | Custom runner image that installs Python packages from `requirements.txt` |
| `requirements.txt` | Python packages installed in the task runner (pandas, numpy, requests, etc.) |
| `n8n-task-runners.json` | Runner configuration |
| `sample.env` | Template for environment variables — copy to `.env` and edit |
| `.env` | Your local secrets (created from `sample.env`, not committed) |
| `.gitignore` | Excludes `.env` from version control |

## Initial Deployment

### 1. Configure Secrets

Copy the sample environment file and edit it with your own values:

```bash
cp sample.env .env
```

Then edit `.env` and set strong credentials:

```env
# N8N
N8N_RUNNERS_AUTH_TOKEN=replace-with-a-strong-secret

# PostgreSQL
POSTGRES_DB=n8n
POSTGRES_USER=n8n
POSTGRES_PASSWORD=replace-with-a-strong-password
```

### 2. (Optional) Configure SMTP / Proxy

Uncomment and edit the relevant lines in `docker-compose.yml` if you need email notifications or an HTTP proxy.

### 3. Start the Stack

```bash
docker compose up -d --build
```

This will:
- Pull the `postgres:16` and `n8nio/n8n:stable` images
- Build the custom task-runners image (installs packages from `requirements.txt`)
- Start PostgreSQL, wait for it to be healthy, then start n8n and the runners

### 4. Access n8n

n8n exposes port 5678. Open your browser at:

```
http://<your-server-ip>:5678
```

PostgreSQL is accessible externally on port 15432:

```
Host: <your-server-ip>
Port: 15432
User/DB/Password: values from .env
```

## Updating n8n

### Pull Latest Images and Rebuild

```bash
docker compose pull
docker compose up -d --build
```

This pulls the latest `n8nio/n8n:stable` and `n8nio/runners:stable` images, rebuilds the custom runner image, and recreates the containers.

### Update Only n8n (without rebuilding runners)

```bash
docker compose pull n8n
docker compose up -d n8n
```

### Update Only Task Runners

```bash
docker compose build --no-cache task-runners
docker compose up -d task-runners
```

## Adding Python Packages to Runners

1. Edit `requirements.txt` and add the package name (pin a version if desired):

   ```
   pandas==2.2.3
   numpy==2.2.6
   requests==2.32.3
   sqlalchemy==2.0.48
   new-package
   ```

2. Edit `n8n-task-runners.json` and add the package to the `N8N_RUNNERS_EXTERNAL_ALLOW` list in the Python runner's `env-overrides`:

   ```json
   "N8N_RUNNERS_EXTERNAL_ALLOW": "numpy,pandas,requests,sqlalchemy,new-package"
   ```

3. Rebuild:

   ```bash
   docker compose build --no-cache task-runners
   docker compose up -d task-runners
   ```

## Useful Commands

| Command | Description |
|---|---|
| `docker compose ps` | Show running containers |
| `docker compose logs -f n8n` | Follow n8n logs |
| `docker compose logs -f task-runners` | Follow runner logs |
| `docker compose logs -f postgres` | Follow PostgreSQL logs |
| `docker compose down` | Stop all services |
| `docker compose down -v` | Stop all services **and delete all data volumes** |
| `docker compose restart n8n` | Restart n8n only |

## Backup

### PostgreSQL Database (recommended)

Dump the database:

```bash
docker compose exec postgres pg_dump -U n8n n8n | gzip > n8n-db-backup-$(date +%F).sql.gz
```

Restore the database:

```bash
gunzip -c n8n-db-backup-YYYY-MM-DD.sql.gz | docker compose exec -T postgres psql -U n8n n8n
```

### n8n Files Volume

n8n also stores encryption keys and config in the `n8n_data` volume. Back it up too:

```bash
docker run --rm -v n8n-suite_n8n_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/n8n-files-backup-$(date +%F).tar.gz -C /data .
```

Restore:

```bash
docker run --rm -v n8n-suite_n8n_data:/data -v $(pwd):/backup alpine \
  tar xzf /backup/n8n-files-backup-YYYY-MM-DD.tar.gz -C /data
```

## Security Notes

- **Never commit `.env`** — it is already in `.gitignore`.
- Change `N8N_RUNNERS_AUTH_TOKEN` and `POSTGRES_PASSWORD` from the default placeholders to strong random values.
- Consider restricting access to ports 5678 and 15432 via a reverse proxy or firewall rules.

---

*Created by kgiannakakis*
