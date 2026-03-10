# DEVELOPMENT Guide (Draft)

**状态**: Final (P0-DEV-001)  
**维护者**: writer / coder / tester  
**标准**: `docs/standards/DOCUMENTATION_STANDARDS.md`  
**最后更新**: 2026-03-09 (self-validated)

> 本文档是开发侧唯一合并入口，聚焦本地开发与验证。部署细节请见 `docs/deployment/`，系统架构设计请见 `docs/architecture/`。

## 1. Scope and Non-Goals

- 目标: 新开发者可在 15 分钟内完成环境准备并启动核心服务。
- 包含: 可执行命令、环境变量、快速启动、开发验证、DataSync 新参数使用。
- 不包含:
  - 生产部署拓扑/运维策略（放在 `docs/deployment/`）
  - 架构决策与技术选型解释（放在 `docs/architecture/`）

## 2. Prerequisites

### 2.1 Required Tools

- Git 2.40+
- Docker + Docker Compose v2
- Python 3.11+
- Node.js 18+ (前端开发需要)

### 2.2 Verify

```bash
git --version
docker --version
docker compose version
python --version
node --version
npm --version
```

## 3. Core Commands (P0 Priority)

### 3.1 Project Bootstrap

```bash
cd /home/ubuntu/.openclaw/workspace/projects/TraderMate

# 最新代码（如适用）
git pull origin main

# 复制环境变量模板
cp .env.example .env
```

### 3.2 Start Core Services (Docker)

```bash
# 启动基础服务（从项目根目录）
cd tradermate
docker compose up -d mysql redis

# 启动 API / worker（portal 为前端服务，见下方）
docker compose up -d api worker

# 查看状态
docker compose ps
```

### 3.3 API and Frontend (Local Helper Scripts)

```bash
# API lifecycle（推荐，从 tradermate/ 目录）
cd tradermate
./scripts/api_service.sh start
./scripts/api_service.sh status

# Frontend lifecycle（从 tradermate-portal/ 目录）
cd ../tradermate-portal
./scripts/portal_service.sh start
./scripts/portal_service.sh status
```

### 3.4 Health Checks

```bash
curl -s http://localhost:8000/health
curl -s http://localhost:8000/docs | head -20
curl -s http://localhost:5173 | head -20
```

## 4. Environment Configuration (P0 Priority)

> 本节仅列开发必需项。完整变量说明见 `docs/development/ENV_VARIABLES_REFERENCE.md`。

### 4.1 Required .env Fields

```bash
# core
DEBUG=true
SECRET_KEY=<generate_with_openssl_rand_hex_32>

# database
MYSQL_PASSWORD=<strong_password>
MYSQL_HOST=localhost
MYSQL_PORT=3306

# optional data source
TUSHARE_TOKEN=<token_or_empty_for_partial_features>

# CORS (dev)
CORS_ORIGINS=["http://localhost:5173","http://localhost:3000"]
```

### 4.2 Generate Secret

```bash
openssl rand -hex 32
```

### 4.3 Confirmed Environment Variables (as of 2026-03-09)

Based on `tradermate/.env.example` and `app/infrastructure/config/config.py`:

```bash
# core
DEBUG=true
SECRET_KEY=<generate_with_openssl_rand_hex_32>

# database (MySQL)
MYSQL_HOST=mysql
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=<strong_password>
MYSQL_DATABASE=tradermate

# optional data source
TUSHARE_TOKEN=<token_or_empty_for_partial_features>

# CORS (dev) - JSON array string
CORS_ORIGINS='["http://localhost:5173","http://localhost:3000","http://127.0.0.1:5173"]'

# sync
SYNC_INTERVAL_HOURS=24
BATCH_SIZE=100
MAX_RETRIES=3

# DataSync限流 (Tushare API rate limit)
TUSHARE_CALLS_PER_MIN=50

# logging
LOG_LEVEL=INFO
```

## 5. Quick Start (15 min)

1. 准备 `.env`
2. 启动 `mysql + redis + api + portal`
3. 执行数据库初始化 SQL
4. 验证 API / Docs / Frontend 页面

### 5.1 Database Init

```bash
# Docker 场景（从项目根目录执行）
cd tradermate
docker compose exec -T mysql mysql -u root -p${MYSQL_PASSWORD} < mysql/init/tradermate.sql
docker compose exec -T mysql mysql -u root -p${MYSQL_PASSWORD} < mysql/init/tushare.sql
docker compose exec -T mysql mysql -u root -p${MYSQL_PASSWORD} < mysql/init/vnpy.sql
docker compose exec -T mysql mysql -u root -p${MYSQL_PASSWORD} < mysql/init/akshare.sql
```

### 5.2 Smoke Test

```bash
# API
curl http://localhost:8000/health

# Optional: run minimal tests
pytest -q tests/ || true
```

### 5.3 Expected Outcome

- `docker compose ps` 中核心服务为 `Up`/`healthy`
- `http://localhost:8000/docs` 可访问
- `http://localhost:5173` 可访问

## 6. DataSync Init Resume and Recovery

> 变更背景与设计见 `docs/development/DATASYNC_RESUME_RATELIMIT_PLAN.md`。

### 6.1 New Flags

- `--resume` (默认开启): 从 `init_progress` checkpoint 续跑
- `--reset-progress`: 清空进度后全量重跑

### 6.2 Usage Examples

```bash
# 默认续跑（--resume 默认为 true，可省略）
PYTHONPATH=tradermate python tradermate/scripts/init_market_data.py

# 明确指定续跑
PYTHONPATH=tradermate python tradermate/scripts/init_market_data.py --resume

# 重置进度后全量重跑
PYTHONPATH=tradermate python tradermate/scripts/init_market_data.py --reset-progress

# 帮助信息
PYTHONPATH=tradermate python tradermate/scripts/init_market_data.py --help
```

**Note**: `--resume` is default ON. Use `--reset-progress` to clear `init_progress` and start fresh.

### 6.3 Checkpoint Recovery Workflow (SOP)

1. 任务中断后，先检查 `init_progress` 最新 phase/cursor。
2. 直接使用 `--resume` 重启任务。
3. 若进度损坏或需全量重跑，执行 `--reset-progress`。
4. 重跑后观察限流日志和错误重试行为。

### 6.4 Rate-Limit Hit Troubleshooting

- 症状: 日志出现 Tushare 限流错误/等待提示。
- 预期行为: 优先解析错误中的等待时长，无法解析时使用指数退避。
- 操作:
  - 保持进程运行，观察自动恢复
  - 若长时间停滞，重启 DataSync 服务: `./scripts/datasync_service.sh restart`
  - 必要时降低并发/批量大小: 修改 `.env` 中的 `BATCH_SIZE`

## 7. Daily Developer Workflow

```bash
# before coding
git checkout main
git pull origin main

# create branch
git checkout -b feat/<topic>

# run tests (minimal smoke)
cd tradermate
pytest -q tests/
```

## 8. Testing Entry

- 快速测试入口: `docs/testing/README.md`
- 验证报告归档: `docs/testing/validation-reports/`

### 8.1 Minimal Dev Test

```bash
cd tradermate
pytest -q tests/
```

**Note**: As of 2026-03-09, `tradermate/tests/` contains `test_config_security.py`. Additional tests may be present; run without markers for general smoke.

## 9. Boundaries to Avoid Duplication

- 本文档不重复维护:
  - 生产部署拓扑、回滚演练细则、值班SOP（`docs/deployment/`）
  - 架构图、模块职责、设计决策（`docs/architecture/`）
- 本文档仅保留“开发可执行入口 + 必要链接”。

## 10. Changelog

- 2026-03-06: 新增 `DEVELOPMENT.md` 合并入口文档（Draft）
- 2026-03-06: 加入 DataSync 断点续跑参数说明（`--resume`/`--reset-progress`）
- 2026-03-06: 增加限流命中与 checkpoint 恢复故障排查
- 2026-03-09: 自验证并定稿（Final），修正路径（`tradermate/mysql/init/*.sql`，`tradermate-portal/scripts/portal_service.sh`），确认 `--resume` 默认开启、CORS 格式
- 2026-03-09: 补充 `TUSHARE_CALLS_PER_MIN=50` 限流配置说明 (非阻塞优化，响应 QA 反馈)

---

如需补充部署恢复流程，请在 `docs/deployment/DEPLOYMENT.md` 更新并在此文档增加链接，不要复制全文。
