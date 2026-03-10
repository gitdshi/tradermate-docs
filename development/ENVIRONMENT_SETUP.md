# 开发环境设置完整指南

本指南为TraderMate项目提供完整的开发环境配置，涵盖**本地原生安装**和**Docker开发模式**两种方式。所有开发者 (designer, coder, tester, writer) 都应阅读本指南。

> **选择哪种方式？**
> - **Docker开发模式 (推荐)**:  simplest, 环境一致, 快速启动, 适合大多数开发者
> - **本地原生安装**: 适合需要深度调试、IDE集成的开发者，或Docker环境受限时

---

## 📚 目录

- [前置条件](#前置条件)
- [方式一: Docker开发模式 (推荐)](#方式一-docker开发模式-推荐)
- [方式二: 本地原生安装](#方式二-本地原生安装)
- [数据库初始化](#数据库初始化)
- [环境验证](#环境验证)
- [开发工具链](#开发工具链)
- [故障排除](#故障排除)
- [下一步](#下一步)

---

## 前置条件

### 必需软件 (两种方式都需要)

| 软件 | 版本要求 | 验证命令 | 安装指南 |
|------|----------|----------|----------|
| **Git** | 2.40+ | `git --version` | [git-scm.com](https://git-scm.com/) |
| **Docker** | 最新稳定版 | `docker --version` | [docs.docker.com](https://docs.docker.com/) |
| **Docker Compose** | v2.20+ | `docker compose version` | 通常随Docker Desktop安装 |

### 方式一特有 (Docker开发模式)

无需额外软件，只需Docker。

### 方式二特有 (本地原生安装)

| 软件 | 版本要求 | 验证命令 | 推荐管理工具 |
|------|----------|----------|-------------|
| **Python** | 3.11+ (建议3.11.9或3.12.x) | `python --version` | `pyenv` (跨版本管理) |
| **Node.js** | 18+ (建议18.20 LTS或20.x) | `node --version` | `nvm` 或 `fnm` |
| **MySQL Client** | 8.0+ (可选，用于数据库操作) | `mysql --version` | 包管理器安装 |
| **Redis CLI** | (可选) | `redis-cli --version` | 包管理器安装 |

---

## 方式一: Docker开发模式 (推荐)

这是最快、最一致的开发环境配置方式，利用Docker Compose管理所有服务。

### 优势

- ✅ **环境一致**: 所有开发者使用相同环境，无"works on my machine"问题
- ✅ **快速启动**: 5分钟内跑通全部依赖 (MySQL, Redis, API, Frontend)
- ✅ **隔离干净**: 不污染主机Python/Node环境
- ✅ **热重载**: 代码变更自动重载
- ✅ **易清理**: `docker compose down` 一键清理

### 步骤

#### 1. 克隆仓库

```bash
cd /home/ubuntu/.openclaw/workspace/projects/TraderMate
# 如果已克隆，确保是最新
git pull origin main
```

#### 2. 创建 `.env` 配置文件

```bash
cd tradermate

# 复制模板
cp .env.example .env

# 编辑 .env，设置必需变量 (至少修改以下几点):
```

**必需的开发环境变量**:

```bash
# MySQL密码 (至少16位，包含大小写字母、数字、特殊字符)
MYSQL_PASSWORD=YourDevPassword123!@#

# JWT密钥 (使用openssl随机生成)
SECRET_KEY=$(openssl rand -hex 32)

# 开发环境标志
DEBUG=true
```

**可选变量** (根据需求设置):

```bash
# Tushare token (获取市场数据，从 https://tushare.pro/ 注册)
# 开发环境无token也可运行，但某些数据功能受限
TUSHARE_TOKEN=your-tushare-token

# CORS源 (开发环境允许本地前端访问)
CORS_ORIGINS=["http://localhost:5173","http://localhost:3000","http://127.0.0.1:5173"]
```

**生产环境才需设置的变量** (开发无需设置):

```bash
# ADMIN_PASSWORD, ADMIN_EMAIL 等 (生产部署时设置)
```

#### 3. 启动开发栈

```bash
# 从 tradermate/ 目录启动所有服务
docker compose up -d

# 查看服务状态
docker compose ps

# 预期输出:
#   Name                   State              Ports
# tradermate-api-1       Up (healthy)       0.0.0.0:8000->8000/tcp
# tradermate-mysql-1     Up (healthy)       0.0.0.0:3306->3306/tcp
# tradermate-redis-1     Up (healthy)       0.0.0.0:6379->6379/tcp
# tradermate-portal-1    Up (healthy)       0.0.0.0:5173->80/tcp
# tradermate-worker-1    Up
```

#### 4. 查看日志

```bash
# 实时查看API日志
docker compose logs -f api

# 查看所有服务日志
docker compose logs -f

# 查看最近100行
docker compose logs --tail=100 api

# 查看过去1小时日志
docker compose logs --since 1h api
```

#### 5. 数据库初始化 (首次启动)

`docker-compose.yml` 已配置自动初始化，但首次可能需要手动执行：

```bash
# 等待MySQL完全就绪 (约30秒)
docker compose logs -f mysql | grep -m 1 "ready for connections"

# 执行初始化脚本 (如果自动脚本未执行)
docker compose exec -T mysql mysql -u root -p${MYSAL_PASSWORD} < mysql/init/tradermate.sql
docker compose exec -T mysql mysql -u root -p${MYSAL_PASSWORD} < mysql/init/tushare.sql
docker compose exec -T mysql mysql -u root -p${MYSAL_PASSWORD} < mysql/init/vnpy.sql
docker compose exec -T mysql mysql -u root -p${MYSAL_PASSWORD} < mysql/init/akshare.sql
```

#### 6. 验证服务

```bash
# API健康检查
curl http://localhost:8000/health

# 预期输出:
# {"status":"healthy","timestamp":"2026-03-03T...","service":"tradermate","dependencies":{"mysql":{"status":"healthy"},"redis":{"status":"healthy"}}}

# API文档 (Swagger UI)
curl http://localhost:8000/docs | head -20

# 前端首页
curl http://localhost:5173 | head -20
```

#### 7. 登录系统

首次启动会自动创建admin用户，密码在日志中:

```bash
# 查看初始admin密码
docker compose logs api | grep -i "admin password"

# 或查看数据库
docker compose exec mysql mysql -u root -p${MYSQL_PASSWORD} -e "SELECT username, email FROM tradermate.users LIMIT 1;"
```

使用admin账户登录前端 (http://localhost:5173)，首次登录需修改密码。

#### 8. 停止和清理

```bash
# 停止所有服务 (保留数据)
docker compose down

# 停止并删除所有数据容器 (⚠️ 数据会丢失!)
docker compose down -v

# 重启单个服务
docker compose restart api
docker compose restart mysql

# 完全重置环境
docker compose down -v
docker volume prune -f  # 删除所有未使用的卷
docker compose up -d
```

---

## 方式二: 本地原生安装

适用于需要IDE深度调试、或Docker环境受限的开发者。

### 步骤

#### 1. 后端设置

```bash
cd /home/ubuntu/.openclaw/workspace/projects/TraderMate/tradermate

# 1. 创建Python虚拟环境
python -m venv .venv

# 2. 激活虚拟环境
source .venv/bin/activate  # Linux/macOS
# Windows: .venv\Scripts\Activate.ps1

# 3. 升级pip
pip install --upgrade pip

# 4. 安装依赖
pip install -r requirements.txt

# 5. 安装开发工具 (如果requirements-dev.txt存在)
if [ -f requirements-dev.txt ]; then
    pip install -r requirements-dev.txt
else
    pip install black flake8 isort mypy pytest pytest-cov pytest-asyncio httpx
fi

# 6. 验证依赖
python -c "import fastapi, sqlalchemy, redis, rq, pymysql; print('Dependencies OK')"
```

#### 2. 环境变量配置

```bash
# 复制模板
cp .env.example .env

# 编辑 .env，至少设置以下变量:
# - MYSQL_PASSWORD (开发密码)
# - SECRET_KEY (JWT密钥)
# - DEBUG=true
```

详细变量说明见 [环境变量参考](../development/ENV_VARIABLES_REFERENCE.md)。

#### 3. 数据库安装和初始化

##### 选项A: 使用本地MySQL + Redis (推荐用于本地原生安装)

**Ubuntu/Debian**:

```bash
# 安装MySQL
sudo apt update
sudo apt install mysql-server mysql-client
sudo systemctl start mysql
sudo systemctl enable mysql

# 安装Redis
sudo apt install redis-server
sudo systemctl start redis
sudo systemctl enable redis

# 创建数据库
mysql -u root -p -e "
CREATE DATABASE tradermate CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE tushare CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE vnpy CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE akshare CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
"
```

**macOS**:

```bash
# MySQL
brew install mysql@8.0
brew services start mysql@8.0
mysql -u root -e "CREATE DATABASE tradermate; CREATE DATABASE tushare; ..."

# Redis
brew install redis
brew services start redis
```

**初始化Schema**:

```bash
# 确保 .env 中的 MYSQL_PASSWORD 正确

# 执行初始化脚本
mysql -u root -p < mysql/init/tradermate.sql
mysql -u root -p < mysql/init/tushare.sql
mysql -u root -p < mysql/init/vnpy.sql
mysql -u root -p < mysql/init/akshare.sql
```

##### 选项B: 继续使用Docker数据库 (混合模式)

如果你喜欢Docker数据库但想要本地Python环境:

```bash
# 只启动数据库服务
docker compose up -d mysql redis

# 设置 .env:
MYSQL_HOST=localhost  # 如果使用localhost连接Docker MySQL，需要端口转发
# 或保持 MYSQL_HOST=mysql (使用Docker网络，仅当Python也在容器中时才有效)
```

对于纯本地Python环境，建议安装本地MySQL。

#### 4. 前端设置

```bash
cd /home/ubuntu/.openclaw/workspace/projects/TraderMate/tradermate-portal

# 1. 安装依赖
npm ci  # 或 npm install

# 2. 配置环境变量
# 创建 .env.local (Vite要求VITE_前缀)
cat > .env.local << 'EOF'
VITE_API_URL=http://localhost:8000
EOF

# 3. 启动开发服务器
npm run dev

# 默认访问: http://localhost:5173
```

#### 5. 启动后端

```bash
cd /home/ubuntu/.openclaw/workspace/projects/TraderMate/tradermate

# 确保虚拟环境已激活
source .venv/bin/activate

# 启动开发服务器 (热重载)
uvicorn app.api.main:app --host 0.0.0.0 --port 8000 --reload

# 或使用脚本
./scripts/api_service.sh start
```

---

## 数据库初始化 (通用)

无论使用哪种方式，都需要初始化数据库Schema。

### 完整初始化流程

```bash
# 1. 确认MySQL运行
# Docker方式: docker compose ps
# 本地方式: systemctl status mysql

# 2. 执行所有init SQL文件 (顺序重要)
# tradermate.sql - 主应用表
# tushare.sql - Tushare数据源表
# vnpy.sql - vnPy兼容表
# akshare.sql - AkShare数据源表

# Docker方式:
docker compose exec -T mysql mysql -u root -p${MYSQL_PASSWORD} < mysql/init/tradermate.sql
docker compose exec -T mysql mysql -u root -p${MYSQL_PASSWORD} < mysql/init/tushare.sql
docker compose exec -T mysql mysql -u root -p${MYSQL_PASSWORD} < mysql/init/vnpy.sql
docker compose exec -T mysql mysql -u root -p${MYSQL_PASSWORD} < mysql/init/akshare.sql

# 本地方式:
mysql -u root -p < mysql/init/tradermate.sql
mysql -u root -p < mysql/init/tushare.sql
mysql -u root -p < mysql/init/vnpy.sql
mysql -u root -p < mysql/init/akshare.sql
```

### 验证数据库

```bash
# 登录MySQL
docker compose exec mysql mysql -u root -p${MYSQL_PASSWORD}  # Docker
# 或
mysql -u root -p  # 本地

# 查看数据库列表
SHOW DATABASES;

# 应看到: tradermate, tushare, vnpy, akshare, information_schema, mysql, performance_schema, sys
```

---

## 环境验证

### 后端验证

```bash
# 1. 数据库连接测试
python -c "
from app.infrastructure.db.connections import get_tradermate_engine
engine = get_tradermate_engine()
with engine.connect() as conn:
    print('✅ Database connection OK')
"

# 2. Redis连接测试
python -c "
import redis
r = redis.Redis(host='localhost', port=6379, db=0)
r.ping()
print('✅ Redis connection OK')
"

# 3. 健康检查端点 (API需已启动)
curl http://localhost:8000/health | jq .

# 4. 查看API文档
open http://localhost:8000/docs  # macOS
xdg-open http://localhost:8000/docs  # Linux
```

### 前端验证

```bash
cd tradermate-portal

# 1. 类型检查
npm run build  # 仅编译检查，不生成文件

# 2. 代码检查
npm run lint

# 3. 单元测试
npm run test:run

# 4. 访问 http://localhost:5173, 应看到TraderMate Portal首页
```

### API连接测试

```bash
# 登录API (首次需获取admin初始密码)
curl -X POST "http://localhost:8000/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"查看日志中的随机密码"}'

# 预期返回 access_token，后续请求使用:
# Authorization: Bearer <token>
```

### 完整测试套件

```bash
# 后端测试
cd tradermate
pytest tests/ -v --cov=app --cov-report=xml --cov-report=term

# 前端单元测试
cd ../tradermate-portal
npm run test:run -- --coverage

# 前端E2E测试 (需API+前端都运行)
npm run test:e2e
```

---

## 开发工具链

### 后端工具 (Python)

```bash
# 代码格式化
black app tests
isort app tests

# 代码检查
flake8 app tests
mypy app --ignore-missing-imports

# 安全扫描
bandit -r app
safety check

# 运行测试
pytest tests/ -v
pytest tests/ -k test_strategies  # 特定测试
```

### 前端工具 (React/Vite)

```bash
cd tradermate-portal

# 代码检查和修复
npm run lint -- --fix

# 类型检查
npx tsc --noEmit

# 单元测试
npm run test:run
npm run test:ui  # 交互式

# E2E测试
npm run test:e2e

# 构建生产版本
npm run build

# 预览生产构建
npm run preview
```

### Docker命令速查

```bash
# 查看所有容器
docker compose ps

# 查看资源使用
docker stats

# 进入容器
docker compose exec api bash
docker compose exec mysql bash
docker compose exec redis sh

# 查看日志 (实时)
docker compose logs -f api

# 重启服务
docker compose restart api

# 重建单个服务镜像 (修改Dockerfile后)
docker compose build api

# 清理无用资源
docker system prune -a --volumes  # ⚠️ 会删除所有未使用镜像/卷
```

---

## 故障排除

### 端口占用

```bash
# 检查端口
lsof -i :8000  # 或: netstat -tulpn | grep :8000

# 杀死占用进程
kill -9 <PID>

# 或修改docker-compose.yml中的端口映射
# 例如: "8001:8000" 将API映射到主机8001端口
```

### MySQL连接失败

```bash
# 检查MySQL是否运行
docker compose ps mysql  # Docker方式
# 或
systemctl status mysql   # 本地方式

# 验证密码
# 检查 .env 中的 MYSQL_PASSWORD 是否与数据库root密码一致

# 测试连接
docker compose exec mysql mysql -u root -p${MYSQL_PASSWORD} -e "SELECT 1;"  # Docker
mysql -u root -p -e "SELECT 1;"  # 本地

# 检查用户权限 (MySQL 8+ 默认使用auth_socket，可能不允许密码登录)
# 如果使用Docker，mysql服务已配置为允许root密码登录
```

### Redis连接失败

```bash
# Docker方式
docker compose logs redis
docker compose exec redis redis-cli ping

# 本地方式
redis-cli ping

# 预期输出: PONG
```

### 依赖安装失败

```bash
# 使用国内镜像源 (中国开发者)
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple --trusted-host pypi.tuna.tsinghua.edu.cn

# TA-Lib安装失败 (常见于Linux)
# 先安装系统依赖:
sudo apt-get install -y ta-lib
# 或使用预编译wheel
pip install TA-Lib‑0.4.28‑cp311‑cp311‑manylinux1_x86_64.whl
```

详细故障排除见 [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)。

---

## 下一步

- **[开发者必读](development/README.md)** - 了解开发工作流和最佳实践
- **[API概览](development/API_README.md)** - 查看所有API端点和使用示例
- **[前端指南](development/frontend/FRONTEND_README.md)** - 前端架构和组件说明
- **[部署手册](deployment/DEPLOYMENT_RUNBOOKS.md)** - 准备将应用部署到生产环境
- **[文档标准](standards/DOCUMENTATION_STANDARDS.md)** - 了解如何为项目贡献文档

---

## 支持和反馈

遇到问题?

1. 查阅 [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) 常见问题
2. 搜索项目Issues
3. 联系团队: @coder, @operator, @tester

---

**最后更新**: 2026-03-03  
**维护者**: @operator, @coder  
**相关文档**:
- [Docker Compose参考](https://docs.docker.com/compose/)
- [FastAPI文档](https://fastapi.tiangolo.com/)
- [Vite文档](https://vitejs.dev/)
