# Environment Setup (Local Dev)

This guide matches the current codebase: Docker is used only for MySQL/Redis, while the API and portal run locally.

## Prerequisites

- Git 2.40+
- Docker + Docker Compose v2
- Python 3.11+
- Node.js 18+

## Backend (API + DB)

1) Configure environment

```bash
cd tradermate
cp .env.example .env
```

Set at least:

```bash
MYSQL_PASSWORD=YourDevPassword123!@#
SECRET_KEY=$(openssl rand -hex 32)
TUSHARE_TOKEN=your-tushare-token
```

2) Start MySQL + Redis

```bash
docker compose -f docker-compose.dev.yml up -d mysql redis
docker compose -f docker-compose.dev.yml ps
```

Data is mounted under `tradermate/.data/`.

3) Install deps + start API

```bash
python -m venv .venv
# Windows PowerShell:
. .venv/Scripts/activate
# WSL/Linux:
source .venv/bin/activate
pip install -r requirements.txt

./scripts/api_service.sh start
```

Logs: `tradermate/logs/api.out`

4) Initialize data (idempotent/resumable)

```bash
./scripts/datasync_service.sh init
```

Logs: `tradermate/logs/data_sync.out`

## Frontend (Portal)

```bash
cd tradermate-portal
npm install
npm run dev -- --host 0.0.0.0 --port 5173
```

Portal: http://localhost:5173
API proxy: http://localhost:8000

## Verify

```bash
curl http://localhost:8000/health
curl http://localhost:5173 | head -20
```

First login requires a password change for the admin user.

## Stop / Clean Up

```bash
./scripts/api_service.sh stop
./scripts/datasync_service.sh stop
docker compose -f docker-compose.dev.yml down
```

To remove DB data:

```bash
rm -rf tradermate/.data
```
