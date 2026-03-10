# STAGING_LESSONS_2026-03-10.md

**Date (UTC)**: 2026-03-10  
**Scope**: Staging environment on `10.0.0.240` (arm64) + cloudflared entry + Docker Compose  
**Outcome**: Staging reachable via `http://10.0.0.240:80` (nginx), API + Portal healthy, DB uses external `10.0.0.73` (tradermate user), images pulled from GHCR (latest, multi-arch).

---

## 1) Final Staging Topology (SSOT)

- **Host**: `10.0.0.240` (Ubuntu 24.04 arm64)
- **Ingress**: `tradermate_nginx` on **:80**
  - `/api/*`, `/docs`, `/health`, `/metrics` -> `api:8000`
  - everything else -> `portal:80`
- **API**: `tradermate_api` on container port 8000 (host 8000 is still exposed, but SSOT is nginx:80)
- **Portal**: `tradermate_portal` on container port 80 (host 5173 exposed)
- **Redis**: docker container (staging)
- **MySQL**: **external** `10.0.0.73:3306` (not in docker)

**Service entrypoints**:
- Primary: `http://10.0.0.240/` (portal) + `http://10.0.0.240/health` (api)
- API direct (debug): `http://10.0.0.240:8000/health`

---

## 2) What Broke (Root Causes)

1. **No Docker on staging host**
   - `docker: command not found`, `docker.service could not be found` -> all compose actions blocked.

2. **Invalid YAML in `docker-compose.staging.yml`**
   - MySQL `environment` used `MYSQL_DATABASE=tradermate` (not `key: value`) -> compose parse error.

3. **Redis healthcheck / password mismatch**
   - Healthcheck used `redis-cli --raw incr ping` (wrong) and `--requirepass ${REDIS_PASSWORD:-}` with empty password -> unhealthy.

4. **API startup failure due to missing ADMIN_PASSWORD**
   - `RuntimeError: ADMIN_PASSWORD environment variable must be set in production mode` -> `/health` connection reset.

5. **Firewall/iptables blocked ports by default**
   - INPUT policy rejected non-allowed ports; needed explicit allow for 8000/5173 (later +80).

6. **Frontend -> API 404 due to routing (cloudflared pointed to portal only)**
   - `https://tm.xdtech.xyz/health` returned HTML -> traffic was landing on portal.

7. **GHCR pull failed (arm64 platform missing)**
   - `no matching manifest for linux/arm64/v8` -> CI built amd64 only.

8. **Nginx 502 after switching to pull-only compose**
   - Nginx upstream connect refused due to stale DNS/IP mapping after service recreation.
   - Quick fix: `docker compose ... restart nginx`.

---

## 3) Fixes Applied (Actionable)

### 3.1 Install Docker on 10.0.0.240

- Install Docker Engine + Compose plugin (Ubuntu 24.04):
  - `docker-ce`, `docker-compose-plugin`

### 3.2 Add staging nginx reverse proxy (single port 80)

- File: `tradermate/nginx/staging.conf`
- Compose service: `nginx` binds `80:80`

### 3.3 Use external MySQL on 10.0.0.73 + least privilege user

- Staging `.env.staging` uses:
  - `MYSQL_HOST=10.0.0.73`
  - `MYSQL_USER=tradermate` (NOT root)

### 3.4 Switch to GHCR pull-only deployment

- Compose: `tradermate/docker-compose.staging.pull.yml` (no `build:`, no code mounts)
- Workflow:
  - build -> push GHCR
  - staging -> `docker login ghcr.io` + `compose pull` + `compose up -d`

### 3.5 Multi-arch images (amd64 + arm64)

- GH Actions requires QEMU for arm64 builds on amd64 runners.
- Workflows updated:
  - `docker/setup-qemu-action@v3`
  - `platforms: linux/amd64,linux/arm64`

---

## 4) Commands / Evidence Checklist (for future)

On `10.0.0.240`:

1) Ports:
```bash
sudo ss -lntp | egrep ':(80|8000|5173) '
```

2) Containers:
```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

3) Pull-only deploy:
```bash
cd /opt/tradermate
sudo docker login ghcr.io
sudo docker compose --env-file .env.staging -f docker-compose.staging.pull.yml pull
sudo docker compose --env-file .env.staging -f docker-compose.staging.pull.yml up -d --pull always
```

4) Health:
```bash
curl -i http://127.0.0.1/health
curl -i http://10.0.0.240/health
curl -i http://10.0.0.240/api/auth/login   # expect 405 on GET
```

5) If nginx returns 502 after redeploy:
```bash
cd /opt/tradermate
sudo docker compose --env-file .env.staging -f docker-compose.staging.pull.yml restart nginx
```

---

## 5) Recommendations (Prevent Repeat)

- **Always single entrypoint**: keep cloudflared -> `10.0.0.240:80` only.
- **Use immutable tags for real releases**: `latest` is fine for staging, but keep `sha-<commit>` tags for rollback.
- **Avoid copying source to staging**: only pull images; keep only `.env.staging` + compose.
- **Add nginx dynamic DNS resolver** (optional improvement): prevents 502 after container IP changes; reduces need to restart nginx.
- **Secrets hygiene**: never paste PATs into chat; prefer 1Password/secret manager and rotate tokens after use.
