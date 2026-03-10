# 环境变量参考手册

本文档汇总TraderMate项目所有环境变量，涵盖后端API和前端Portal。

## 目录
- [概述](#概述)
- [后端环境变量](#后端环境变量)
- [前端环境变量](#前端环境变量)
- [变量优先级](#变量优先级)
- [示例配置](#示例配置)
- [安全建议](#安全建议)

## 概述

环境变量用于配置应用行为，避免硬编码。生产环境务必使用安全的secret管理方式 (如Vault、K8s Secrets、.env加密)。

**通用规则**:
- 布尔值: `true`/`false` (不区分大小写)
- 列表: JSON数组格式 `["a","b"]`
- 整数/浮点数: 直接数值
- 字符串: 引号可省略，除非包含空格或特殊字符

## 后端环境变量

**作用域**: `tradermate/` 目录，由 `app.infrastructure.config.config.Settings` 加载。

### 核心配置

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `DEBUG` | 否 | `false` | 调试模式。生产必须为false。开启后显示详细错误信息。 |
| `APP_NAME` | 否 | `TraderMate API` | 应用名称，显示在Swagger文档中。 |
| `APP_VERSION` | 否 | `1.0.0` | 应用版本号。 |

### 数据库配置

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `MYSQL_HOST` | 否 | `127.0.0.1` | MySQL主机地址。Docker环境通常为 `mysql`。 |
| `MYSQL_PORT` | 否 | `3306` | MySQL端口。 |
| `MYSQL_USER` | 否 | `root` | MySQL用户名。 |
| `MYSQL_PASSWORD` | **是** | - | MySQL root密码。建议16+字符，含大小写字母、数字、特殊字符。 |
| `TRADERMATE_DATABASE` | 否 | `tradermate` | TraderMate主数据库名。 |
| `TUSHARE_DATABASE` | 否 | `tushare` | Tushare数据数据库名。 |
| `TUSHARE_TOKEN` | 推荐 | - | Tushare API token (从 https://tushare.pro/ 获取)。用于获取市场数据。 |
| `VNPY_DATABASE_URL` | 否 | - | vn.py数据库连接URL。如未设置，使用 `MYSQL_*` 组合。格式: `mysql+pymysql://user:pass@host:port/db?charset=utf8mb4` |
| `AKSHARE_DATABASE_URL` | 否 | - | Akshare数据库连接URL。如未设置，使用 `{mysql_url}/akshare`。 |

**示例**:
```bash
MYSQL_HOST=mysql
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=SuperSecure!Pass123
TRADERMATE_DATABASE=tradermate
TUSHARE_DATABASE=tushare
TUSHARE_TOKEN=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### Redis配置

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `REDIS_HOST` | 否 | `127.0.0.1` | Redis主机地址。Docker环境通常为 `redis`。 |
| `REDIS_PORT` | 否 | `6379` | Redis端口。 |
| `REDIS_DB` | 否 | `0` | Redis数据库编号。 |

**示例**:
```bash
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0
```

### JWT认证配置

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `SECRET_KEY` | **是** | - | JWT签名密钥。必须32+字符，使用 `openssl rand -hex 32` 生成。 |
| `JWT_ALGORITHM` | 否 | `HS256` | JWT算法。支持: HS256, HS384, HS512。 |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | 否 | `60` | 访问令牌过期时间(分钟)。默认24小时。 |
| `REFRESH_TOKEN_EXPIRE_DAYS` | 否 | `7` | 刷新令牌过期天数。 |

**示例**:
```bash
SECRET_KEY=$(openssl rand -hex 32)
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=7
```

### 管理员账户配置

| 变量名 | 必需 (生产) | 默认值 | 说明 |
|--------|-------------|--------|------|
| `ADMIN_USERNAME` | 否 | `admin` | 管理员用户名。 |
| `ADMIN_EMAIL` | 否 | `admin@tradermate.local` | 管理员邮箱。 |
| `ADMIN_PASSWORD` | **是** (生产) | - | 管理员初始密码。生产必须设置，dev会自动生成随机密码并记录日志。 |

**示例**:
```bash
ADMIN_USERNAME=admin
ADMIN_EMAIL=admin@tradermate.local
ADMIN_PASSWORD=YourAdminPassword!2025
```

### CORS配置

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `CORS_ORIGINS` | 否 | `["http://localhost:5173","http://localhost:3000","http://127.0.0.1:5173"]` | 允许的CORS源，JSON数组格式。 |

**示例**:
```bash
CORS_ORIGINS=["http://localhost:5173","http://localhost:3000","http://127.0.0.1:5173","https://tradermate.example.com"]
```

### 应用功能配置

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `MAX_CONCURRENT_BACKTESTS` | 否 | `4` | 最大并发回测数量。控制资源消耗。 |
| `BACKTEST_TIMEOUT_SECONDS` | 否 | `600` | 单个回测超时时间(10分钟)。 |
| `RQ_QUEUE_NAME` | 否 | `default` | RQ队列名称。 |
| `SYNC_INTERVAL_HOURS` | 否 | `24` | 数据同步间隔(小时)。 |
| `BATCH_SIZE` | 否 | `100` | 批量操作数量。 |
| `MAX_RETRIES` | 否 | `3` | 操作最大重试次数。 |

### 日志配置

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `LOG_LEVEL` | 否 | `INFO` | 日志级别: DEBUG, INFO, WARNING, ERROR, CRITICAL。 |
| `LOG_FORMAT` | 否 | `json` | 日志格式: `json` (结构化) 或 `text`。 |
| `LOG_FILE` | 否 | - | 日志文件路径。未设置则仅输出到stdout。 |

**示例**:
```bash
LOG_LEVEL=DEBUG
LOG_FORMAT=json
LOG_FILE=/app/logs/app.log
```

### vn.py特定配置

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `VN_DATAFEED_NAME` | 否 | - | 数据源名称。如 `tushare`。 |
| `VN_DATAFEED_USERNAME` | 否 | - | 数据源用户名。 |
| `VN_DATAFEED_PASSWORD` | 否 | - | 数据源密码。 |
| `VN_DATABASE_NAME` | 否 | `mysql` | vn.py数据库驱动。 |
| `VN_DATABASE_HOST` | 否 | - | vn.py数据库主机 (默认使用 `MYSQL_HOST`)。 |
| `VN_DATABASE_PORT` | 否 | - | vn.py数据库端口。 |
| `VN_DATABASE_USER` | 否 | - | vn.py数据库用户。 |
| `VN_DATABASE_PASSWORD` | 否 | - | vn.py数据库密码 (优先于 `MYSQL_PASSWORD`)。 |
| `VN_DATABASE_DB` | 否 | - | vn.py数据库名。 |

**vn.py完整示例**:
```bash
# vn.py连接使用独立配置 (如果与MYSQL_*不同)
VN_DATABASE_HOST=vnpy-db.example.com
VN_DATABASE_PORT=3306
VN_DATABASE_USER=vnpy_user
VN_DATABASE_PASSWORD=vnpy_pass123
VN_DATABASE_DB=vnpy

# 数据源配置
VN_DATAFEED_NAME=tushare
VN_DATAFEED_USERNAME=${TUSHARE_TOKEN}
VN_DATAFEED_PASSWORD=
```

## 前端环境变量

**作用域**: `tradermate-portal/`，Vite构建时注入。

**命名规则**: 必须以 `VITE_` 开头，如 `VITE_API_URL`。

### API配置

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `VITE_API_URL` | 否 | `http://localhost:8000` | 后端API基础URL。生产环境设为 `https://api.tradermate.example.com`。 |
| `VITE_APP_ENV` | 否 | `development` | 应用环境: `development` / `staging` / `production`。 |

**示例** (`.env.local`):
```bash
VITE_API_URL=http://localhost:8000
VITE_APP_ENV=development
```

### 功能开关 (可选)

可根据需要添加:

| 变量名 | 说明 |
|--------|------|
| `VITE_ENABLE_ANALYTICS` | 是否启用Google Analytics等分析 (true/false)。 |
| `VITE_SENTRY_DSN` | Sentry错误监控DSN。 |
| `VITE_GA_ID` | Google Analytics ID。 |

**示例**:
```bash
VITE_ENABLE_ANALYTICS=false
VITE_SENTRY_DSN=https://xxxx@sentry.io/xxxxx
```

## 变量优先级

1. **系统环境变量** (实际系统export的变量) - 最高
2. ** `.env` 文件** (Pydantic加载) - 开发环境中 `.env` 优先级高于 `.env.example`
3. **默认值** (代码中定义) - 最低

**注意**:
- 前端: Vite只在构建时读取 `.env.local`、`.env.[mode]`、`.env`，运行时无法动态修改
- 后端: Pydantic每次启动读取 `.env`，运行时可动态修改环境变量但Settings实例是缓存的

## 示例配置

### 开发环境 (.env 和 .env.local)

**后端 `.env`**:
```bash
# 基础配置
DEBUG=true
APP_NAME="TraderMate API (Dev)"
APP_VERSION=1.0.0-dev

# 数据库 (使用Docker)
MYSQL_HOST=mysql
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=dev-mysql-password-123!
TRADERMATE_DATABASE=tradermate
TUSHARE_DATABASE=tushare
TUSHARE_TOKEN=dev-token-ignore

# Redis
REDIS_HOST=redis
REDIS_PORT=6379

# JWT
SECRET_KEY=dev-secret-key-change-in-prod-1234567890
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=1440  # 24h
REFRESH_TOKEN_EXPIRE_DAYS=30

# 管理员
ADMIN_USERNAME=admin
ADMIN_EMAIL=admin@tradermate.local
ADMIN_PASSWORD=admin123!@#

# CORS
CORS_ORIGINS=["http://localhost:5173","http://localhost:3000"]

# 日志
LOG_LEVEL=DEBUG
LOG_FORMAT=text

# 性能
MAX_CONCURRENT_BACKTESTS=2
BACKTEST_TIMEOUT_SECONDS=300
```

**前端 `.env.local`**:
```bash
VITE_API_URL=http://localhost:8000
VITE_APP_ENV=development
```

### 生产环境 (.env.production)

**后端**:
```bash
DEBUG=false
APP_NAME=TraderMate API
APP_VERSION=1.0.0

# 生产数据库 (云托管或独立服务器)
MYSQL_HOST=prod-mysql.cluster.example.com
MYSQL_PORT=3306
MYSQL_USER=tradermate
MYSQL_PASSWORD=SuperSecureProductionPass!2025$%^
TRADERMATE_DATABASE=tradermate
TUSHARE_DATABASE=tushare
TUSHARE_TOKEN=prod-tushare-token

REDIS_HOST=prod-redis.cluster.example.com
REDIS_PORT=6379
REDIS_DB=0

SECRET_KEY=actual-64-char-secret-from-vault-... (openssl rand -hex 32)
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=7

# 生产必需
ADMIN_USERNAME=admin
ADMIN_EMAIL=admin@tradermate.example.com
ADMIN_PASSWORD=StrongProdPassword!2025

# CORS仅限生产域名
CORS_ORIGINS=["https://tradermate.example.com","https://app.tradermate.example.com"]

LOG_LEVEL=INFO
LOG_FORMAT=json

# 增加资源限制
MAX_CONCURRENT_BACKTESTS=8
BACKTEST_TIMEOUT_SECONDS=900

# 添加监控
SENTRY_DSN=https://xxxx@sentry.io/xxxxx
PROMETHEUS_MULTIPROC_DIR=/tmp
```

**前端**:
```bash
VITE_API_URL=https://api.tradermate.example.com
VITE_APP_ENV=production
VITE_SENTRY_DSN=https://xxxx@sentry.io/xxxxx
VITE_GA_ID=UA-XXXXXX-X
```

## 安全建议

### 敏感变量管理

1. **Never commit .env to Git!**
   ```bash
   echo ".env" >> .gitignore
   echo ".env.local" >> tradermate-portal/.gitignore
   ```

2. **使用 secrets 管理**:
   - 开发: `direnv` + 加密文件
   - Docker: `--env-file` 或 Docker Secrets
   - K8s: `k8s/secrets.yaml`
   - CI/CD: GitHub Actions Secrets / GitLab CI Variables

3. **轮换策略**:
   - `SECRET_KEY`: 每次major部署轮换，需要重新登录所有用户
   - `MYSQL_PASSWORD`: 每季度轮换
   - `ADMIN_PASSWORD`: 强制密码策略，90天过期

4. **最小权限原则**:
   - MySQL用户只授予所需权限: `SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX`
   - Redis使用独立数据库编号，避免使用默认db=0

5. **网络隔离**:
   - 生产数据库仅允许应用服务器IP访问
   - Redis禁用公网访问 (`bind 127.0.0.1`)

### 检查清单

- [ ] `.env` 已加入 `.gitignore`
- [ ] `SECRET_KEY` 长度≥32字符，高强度随机
- [ ] `MYSQL_PASSWORD` 复杂度: 大小写+数字+特殊符号，长度≥16
- [ ] `DEBUG=false` 生产环境
- [ ] `CORS_ORIGINS` 仅包含可信域名
- [ ] API密钥 (`TUSHARE_TOKEN`) 未泄露
- [ ] 使用HTTPS (`VITE_API_URL` 使用 `https://`)
- [ ] 管理员密码已修改 (非默认)

## 故障排除

### 后端忽略环境变量

**原因**: `.env` 文件不存在或不在工作目录。

**验证**:
```bash
cd tradermate
python -c "from app.infrastructure.config import get_settings; s=get_settings(); print(s.model_dump())"
```

**解决**:
```bash
# 确保 .env 在项目根目录
ls -la .env

# 清楚缓存 (Settings是lru_cache)
# 重启Python进程
```

### 前端环境变量未定义

**原因**: Vite只暴露 `VITE_*` 前缀变量。

**验证**:
```javascript
// 在浏览器控制台
console.log(import.meta.env.VITE_API_URL)
```

**解决**: 确保 `.env.local` 包含 `VITE_API_URL`，并且构建时文件存在。

### Docker环境变量未传递

**原因**: Compose文件缺少 `env_file` 或 `environment` 配置。

**检查**:
```bash
docker compose config | grep -A 10 "api:"
```

**解决**: 在 `docker-compose.yml` 中添加:
```yaml
api:
  env_file:
    - .env
```

### 数据库连接拒绝

**原因**: `MYSQL_PASSWORD` 错误或MySQL未运行。

**排查**:
```bash
# 1. 检查MySQL运行
docker compose ps mysql
# 或
systemctl status mysql

# 2. 测试连接
mysql -h 127.0.0.1 -P 3306 -u root -p

# 3. 检查docker-compose日志
docker compose logs mysql
```

---

**最后更新**: 2026-03-02  
**维护者**: @operator
