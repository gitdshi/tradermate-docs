# 故障排除指南

本文档收集本地开发环境的常见问题和解决方案。

## 目录
- [快速诊断](#快速诊断)
- [后端问题](#后端问题)
- [前端问题](#前端问题)
- [数据库问题](#数据库问题)
- [Docker问题](#docker问题)
- [网络问题](#网络问题)
- [性能问题](#性能问题)

## 快速诊断

### 1. 检查服务状态

```bash
# 后端API
curl -s http://localhost:8000/health | jq .

# 前端
curl -s http://localhost:5173 | head -20

# MySQL
docker compose exec mysql mysqladmin ping -uroot -p${MYSQL_PASSWORD}
# 或本地安装
mysqladmin ping -uroot -p

# Redis
redis-cli ping
```

### 2. 查看日志

```bash
# API容器日志 (最近100行)
docker compose logs --tail=100 api

# 滚动日志
docker compose logs -f api

# 本地非Docker环境
tail -f tradermate/logs/app.log
```

### 3. 验证环境变量

```bash
# 后端
cd tradermate
python -c "
from app.infrastructure.config import get_settings
s = get_settings()
print('MYSQL_HOST:', s.mysql_host)
print('DEBUG:', s.debug)
print('DATABASE_URL:', s.tradermate_db_url)
"

# 前端 (浏览器)
console.log(import.meta.env)
```

## 后端问题

### 问题: ModuleNotFoundError: No module named 'xxx'

**症状**: 启动时导入错误。

**原因**:
- 虚拟环境未激活
- 依赖未安装
- 路径问题

**解决**:
```bash
# 1. 确保虚拟环境激活
which python
# 应返回: /path/to/tradermate/.venv/bin/python

# 2. 重新安装依赖
pip install -r requirements.txt

# 3. 检查PYTHONPATH
echo $PYTHONPATH
# 不应包含干扰路径
```

### 问题: ImportError: cannot import name 'xxx' from 'vnpy'

**症状**: vn.py导入错误。

**原因**: vn.py版本与代码不兼容。

**解决**:
```bash
# 检查vnpy版本
pip show vnpy
# 应为4.3.0或更高

# 固定版本
pip install "vnpy==4.3.0" --force-reinstall
```

### 问题: pymysql.err.OperationalError: (2002, "Can't connect to MySQL server")

**症状**: 无法连接MySQL。

**排查**:
```bash
# 1. MySQL是否运行
systemctl status mysql       # 本地安装
docker compose ps mysql      # Docker

# 2. 检查端口
netstat -tulpn | grep 3306
lsof -i :3306

# 3. 测试连接
mysql -h 127.0.0.1 -P 3306 -u root -p

# 4. 验证 .env 配置
echo $MYSQL_HOST
echo $MYSQL_PASSWORD
```

**常见错误和修复**:
```bash
# 错误: Host '127.0.0.1' is not allowed to connect
# 解决: 在MySQL中授权
mysql -u root -p -e "
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'your_password' WITH GRANT OPTION;
FLUSH PRIVILEGES;
"

# 错误: Access denied for user 'root'@'%'
# 解决: 检查密码，确认用户权限
SELECT user, host FROM mysql.user;
```

### 问题: redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379. Connection refused.

**症状**: Redis连接失败。

**解决**:
```bash
# 启动Redis
sudo systemctl start redis  # 本地
docker compose up -d redis  # Docker

# 测试
redis-cli ping  # 应返回 "PONG"

# 检查配置
redis-cli CONFIG GET bind
# 应允许外部连接或确保应用在同一容器/网络
```

### 问题: alembic.util.exc.CommandError: Target database is not up to date.

**症状**: 迁移错误 (注意: 本项目未使用alembic，但如有迁移工具错误可参考)。

**临时解决**: 本项目用SQL初始化，直接执行SQL脚本。

### 问题: 热重载不工作 (代码修改后未重启)

**症状**: 修改 `.py` 文件后API未自动重载。

**排查**:
```bash
# 1. 是否使用 --reload 标志
docker compose ps api
# 命令应包含: uvicorn ... --reload

# 2. 文件修改时间更新
ls -l app/api/main.py

# 3. inotify watchers限制 (Linux)
cat /proc/sys/fs/inotify/max_user_watches
# 应 ≥ 8192，建议 524288

# 4. 卷挂载正确性
docker compose exec api ls -l /app/app/api/main.py
# 应看到修改时间更新
```

**解决**:
```bash
# 增加inotify限制
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 重启Docker服务
docker compose restart api
```

### 问题: 启动时报错 "Address already in use"

**症状**: 端口被占用。

**解决**:
```bash
# 1. 查找占用进程
lsof -i :8000  # API端口
lsof -i :5173  # 前端端口
lsof -i :3306  # MySQL
lsof -i :6379  # Redis

# 2. 杀死进程
kill -9 <PID>

# 3. 或修改端口
# .env 中修改 API 端口 (需同时修改 docker-compose.yml)
# docker-compose.yml 中:
#   ports:
#     - "8001:8000"  # 宿主机8001 -> 容器8000
```

## 前端问题

### 问题: Vite server not starting, port already in use

**症状**: `npm run dev` 失败，端口5173被占用。

**解决**:
```bash
# 1. 查找占用进程
lsof -i :5173
kill -9 <PID>

# 2. 或使用不同端口
npm run dev -- --port=5174

# 3. 杀死所有Node进程 (谨慎)
pkill -f "vite"
```

### 问题: Cannot find module 'xxx' or "export 'xxx' was not found

**症状**: 导入错误或缺失导出。

**原因**: node_modules未完整安装或版本冲突。

**解决**:
```bash
# 清理重新安装
rm -rf node_modules package-lock.json
npm cache clean --force
npm install

# 或使用npm ci (更严格)
npm ci

# 如果仍有问题，删除lock文件重试
```

### 问题: ESLint errors blocking development

**症状**: 启动或保存时ESLint报错。

**临时绕过**:
```bash
# 只在CI运行lint，开发时不检查
# package.json
"scripts": {
  "dev": "NODE_OPTIONS='--no-warnings' vite"
}

# 或禁用ESLint插件
# vite.config.ts 中注释掉 eslint 插件
```

**正确解决**:
```bash
# 自动修复
npm run lint -- --fix

# 暴力忽略某些规则
# .eslintrc.cjs 或 eslint.config.js 修改规则
```

### 问题: API请求失败 (CORS或404)

**症状**: 浏览器控制台网络请求失败。

**排查**:
```javascript
// 浏览器console
fetch('http://localhost:8000/api/health')
  .then(r => console.log(r.status, r.statusText))
```

**常见原因**:
1. **CORS**: 后端未允许前端源
   - 检查 `CORS_ORIGINS` 包含 `http://localhost:5173`
   - `docker compose restart api`

2. **API URL错误**: 
   - 检查 `.env.local` 中 `VITE_API_URL`
   - 确认为 `http://localhost:8000` (非 `http://127.0.0.1:8000` 或 `https://`)

3. **后端未启动**: 
   ```bash
   curl -I http://localhost:8000/health
   ```

4. **路径错误**: 
   - 前端调用 `/api/xxx`，后端路由注册为 `/api/xxx`
   - 检查 `tradermate-portal/src/lib/api.ts` 的 `baseURL`

**后端日志检查**:
```bash
docker compose logs api | grep -i "cors\|orign\|header"
```

### 问题: E2E tests fail in CI/local

**症状**: Playwright测试失败。

**原因**: 浏览器未安装或环境变量设置。

**解决**:
```bash
# 1. 安装浏览器依赖
npx playwright install --with-deps

# 2. 确保API和前端都在运行
# 3. 设置测试环境变量
export VITE_API_URL=http://localhost:8000
export BASE_URL=http://localhost:5173

# 4. 运行 headed 调试
npm run test:e2e:headed
```

### 问题: npm run build fails with TypeScript errors

**症状**: 构建时TS类型错误。

**解决**:
```bash
# 1. 检查tsconfig
cat tsconfig.json
cat tsconfig.app.json

# 2. 仅检查类型不构建
npx tsc --noEmit

# 3. 修复错误 (示例: 缺少类型声明)
# 添加 src/vite-env.d.ts
echo "declare module '*.png'" > src/vite-env.d.ts

# 4. 或跳过检查 (临时)
npm run build -- --noEmit
```

## 数据库问题

### 问题: "Unknown database 'tradermate'"

**症状**: 连接数据库但库不存在。

**解决**:
```bash
# 1. 登录MySQL
mysql -u root -p

# 2. 查看数据库
SHOW DATABASES;

# 3. 创建缺失数据库
CREATE DATABASE tradermate CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE tushare CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE vnpy CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE akshare CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

# 4. 执行初始化脚本
USE tradermate;
SOURCE tradermate/mysql/init/tradermate.sql;
USE tushare;
SOURCE tradermate/mysql/init/tushare.sql;
USE vnpy;
SOURCE tradermate/mysql/init/vnpy.sql;
USE akshare;
SOURCE tradermate/mysql/init/akshare.sql;
```

### 问题: MySQL "Access denied" after docker compose up

**症状**: 容器内连接MySQL失败。

**排查**:
```bash
# 1. 检查环境变量是否正确传递
docker compose exec api env | grep MYSQL
# 应看到 MYSQL_PASSWORD 和 MYSQL_HOST=mysql

# 2. MySQL日志
docker compose logs mysql | tail -50

# 3. 在MySQL容器内测试
docker compose exec mysql mysql -u root -p${MYSQL_PASSWORD} -e "SELECT 1"

# 4. 确认root密码正确
# 默认密码来自 .env，或在MySQL初始化时设置
```

**修复**:
```bash
# 如果忘记密码，重置MySQL root密码
docker compose down
# 删除数据卷 (会丢失数据!)
docker volume rm tradermate_mysql_data
# 重新启动，使用新密码
MYSQL_PASSWORD=newpass docker compose up -d mysql
# 然后初始化数据库
```

### 问题: MySQL "Too many connections"

**症状**: 连接池耗尽。

**解决**:
```sql
-- 登录MySQL查看
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';

-- 临时增加
SET GLOBAL max_connections = 200;

-- 永久修改 (my.cnf / my.ini)
[mysqld]
max_connections = 200
```

**应用侧优化**:
```bash
# .env 中限制连接池
# 但本项目使用SQLAlchemy默认池，可调整:
# 在代码中添加:
# engine = create_engine(..., pool_size=10, max_overflow=20, pool_pre_ping=True)
```

### 问题: Slow MySQL queries in development

**症状**: 查询慢，开发环境性能差。

**优化**:
1. **启用查询缓存**:
   ```sql
   SET GLOBAL query_cache_size = 104857600;  -- 100MB
   SET GLOBAL query_cache_type = 1;
   ```

2. **使用索引**: 
   ```sql
   EXPLAIN SELECT * FROM some_table WHERE symbol='AAPL';
   -- 检查是否用到索引，添加: CREATE INDEX idx_symbol ON some_table(symbol);
   ```

3. **减少数据量**: 使用 `LIMIT` 或时间范围查询

4. **禁用外键检查** (测试时):
   ```sql
   SET FOREIGN_KEY_CHECKS=0;
   -- 执行大量插入
   SET FOREIGN_KEY_CHECKS=1;
   ```

### 问题: Redis keys not persisting after restart

**症状**: Redis重启后数据丢失。

**原因**: 默认Redis配置不持久化RDB/AOF。

**配置持久化**:
```bash
# 在 docker-compose.yml 中添加:
redis:
  command: >
    redis-server 
    --appendonly yes 
    --appendfsync everysec
    --save 60 1
  volumes:
    - redis_data:/data
```

## Docker问题

### 问题: "permission denied" while mounting volume

**症状**: Docker挂载卷时报权限错误。

**常见于Linux**: 容器内用户UID(5678)与宿主机文件所有者不匹配。

**解决**:

#### 方案1: 调整目录权限 (开发)

```bash
sudo chown -R 1000:1000 /home/ubuntu/.openclaw/workspace/projects/TraderMate/tradermate/app
# 注意: 1000是容器内用户ID，查看Dockerfile中的 appuser UID

# 查看Dockerfile中的UID
grep "adduser" tradermate/Dockerfile
# 示例: adduser -u 5678 ...
```

#### 方案2: 使用root运行开发容器 (简单)

```yaml
# docker-compose.dev.yml
api:
  user: root  # 强制root
```

#### 方案3: 使用匿名卷避免权限问题

```yaml
api:
  volumes:
    - /dev/null:/app/app  # 不挂载代码，改用docker cp热加载?
```

#### 方案4: 修改Dockerfile使用当前用户 (推荐长期)

```dockerfile
# 开发Dockerfile
ARG UID=1000
ARG GID=1000
RUN groupadd -g $GID appgroup && useradd -u $UID -g $GID -ms /bin/bash appuser
USER appuser
```

### 问题: Docker build takes forever / slow

**症状**: 构建镜像慢。

**优化**:
1. **使用国内镜像** (China):
```bash
# daemon.json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
```

2. **利用缓存**:
```bash
# 顺序很重要: Dockerfile中不变依赖在前
COPY requirements.txt ./
RUN pip install -r requirements.txt
COPY app/ ./app

# 避免COPY . . (会失效缓存)
```

3. **多阶段构建**:
```dockerfile
FROM python:3.11 as builder
# 安装依赖到 /install
FROM python:3.11-slim
COPY --from=builder /install /usr/local
```

4. **BuildKit并行**:
```bash
DOCKER_BUILDKIT=1 docker compose build
```

### 问题: Docker container exits immediately

**症状**: 启动后立即退出。

**排查**:
```bash
# 查看容器状态
docker compose ps

# 查看退出代码
docker compose ps api
# STATUS: "Exit 1"

# 查看最后日志
docker compose logs --tail=50 api

# 进入容器调试
docker compose run --rm api bash
# 手动运行CMD
uvicorn app.api.main:app --host 0.0.0.0 --port 8000
```

**常见原因**:
- 依赖缺失: `pip install -r requirements.txt`
- 端口冲突: 修改 `ports` 映射
- 环境变量缺失: 检查 `.env` 是否加载

### 问题: Docker volume not mounted correctly

**症状**: 文件不在容器内或未更新。

**验证**:
```bash
# 查看挂载信息
docker compose inspect api | jq '.[0].Mounts'

# 进入容器检查
docker compose exec api ls -la /app/app

# 宿主机文件时间戳是否更新?
ls -la tradermate/app/api/main.py
```

**常见错误**:
```yaml
# 错: 相对路径会被解释为相对于compose文件
volumes:
  - ./app:/app/app

# 对: 绝对路径
volumes:
  - /absolute/path/to/tradermate/app:/app/app
```

## 网络问题

### 问题: Cannot reach service on localhost but container is running

**症状**: `curl: (7) Failed to connect`。

**排查**:
```bash
# 1. 容器是否运行
docker compose ps

# 2. 端口映射正确?
docker compose port api 8000
# 返回: "0.0.0.0:8000"

# 3. 宿主机防火墙
sudo ufw status
# 若active，允许端口: sudo ufw allow 8000

# 4. 容器内应用是否监听0.0.0.0
docker compose exec api netstat -tulpn | grep 8000
# 应看到: 0.0.0.0:8000 非 127.0.0.1:8000
```

### 问题: Services cannot talk to each other

**症状**: 容器间连接失败 (`api` 无法连 `mysql`)。

**排查**:
```bash
# 1. 容器是否在同一网络
docker network inspect tradermate_network

# 2. DNS解析
docker compose exec api getent hosts mysql
# 应解析到 mysql 容器IP

# 3. 从api容器测试MySQL
docker compose exec api mysql -hmysql -uroot -p${MYSQL_PASSWORD} -e "SELECT 1"

# 4. 检查依赖条件
# docker-compose.yml 中 mysql 有 healthcheck，api 有 depends_on condition
```

**常见配置错误**:
```yaml
# 错: 使用 localhost 或 127.0.0.1
MYSQL_HOST=localhost  # 在容器内localhost是容器自己!

# 对: 使用服务名
MYSQL_HOST=mysql
```

### 问题: Slow network between containers

**症状**: 容器间网络延迟高。

**原因**: Docker网络bridge模式有轻微开销，通常<1ms。如发现>10ms可能是:

- 宿主机网络拥塞
- 大量日志输出阻塞
- 容器资源不足

**解决**:
```bash
# 1. 检查资源
docker stats

# 2. 限制日志大小
# docker-compose.yml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"

# 3. 将相关服务放同一容器 (仅开发，不推荐生产)
# 或使用 host network (Linux only)
network_mode: "host"
```

## 性能问题

### 问题: High CPU usage in API container

**症状**: CPU持续>80%。

**排查**:
```bash
# 1. 查看容器资源
docker stats

# 2. 进入容器top
docker compose exec api top

# 3. Python CPU Profiling
docker compose exec api py-spy top --pid 1
# 或
docker compose exec api python -m cProfile -o /tmp/profile.out app/api/main.py
docker compose cp api:/tmp/profile.out ./
pip install snakeviz
snakeviz profile.out
```

**可能原因**:
- 无限循环或死锁 (检查代码)
- GIL冲突 (多线程Python)
- 调试模式 `DEBUG=true` 增加开销
- 未优化的查询 (N+1问题)

**临时缓解**:
```bash
# 1. 限制容器CPU
docker compose up -d --scale api=2
# 或
# docker-compose.yml 中添加:
deploy:
  resources:
    limits:
      cpus: '1.0'

# 2. 关闭DEBUG
DEBUG=false
docker compose restart api
```

### 问题: Memory leak / OOM killed

**症状**: 容器退出，日志中有 "Killed"。

**排查**:
```bash
# 1. 查看容器退出码
docker inspect tradermate_api | grep -A 5 "State"

# 2. 内存监控
docker stats --no-stream

# 3. 检查应用内存
docker compose exec api python -c "
import psutil, os
process = psutil.Process(os.getpid())
print(f'RSS: {process.memory_info().rss / 1024 / 1024:.2f} MB')
"
```

**内存泄漏来源**:
- 未关闭数据库连接 (使用with或finally)
- 大对象未释放
- 无限增长缓存 (Redis未设置TTL)
- 循环引用阻止GC

**解决**:
```python
# 1. 确保连接关闭
with engine.connect() as conn:
    result = conn.execute(...)
# with块结束后自动close

# 2. 设置内存限制
# docker-compose.yml
deploy:
  resources:
    limits:
      memory: 2G
    reservations:
      memory: 512M

# 3. OOM时自动重启
restart: unless-stopped  # 已有
```

### 问题: Slow Docker builds

**原因**: 未有效利用缓存。

**加速**:
```bash
# 1. 使用BuildKit
DOCKER_BUILDKIT=1 docker compose build

# 2. 分离依赖复制
# Dockerfile.api 保持:
COPY requirements.txt ./
RUN pip install -r requirements.txt
COPY app/ ./app

# 3. 不挂载不必要的文件
.dockerignore:
.git
__pycache__
*.pyc
.env
logs/
tests/
*.log
```

## 其他问题

### 问题: Timezone issues (时间显示错误)

**症状**: 数据库或应用时间差8小时 (Asia/Shanghai vs UTC)。

**解决**:
```bash
# 1. 容器时区
docker compose exec api date
# 应为本地时区或UTC

# 设置时区:
# docker-compose.yml
api:
  environment:
    - TZ=Asia/Shanghai
  volumes:
    - /etc/timezone:/etc/timezone:ro
    - /etc/localtime:/etc/localtime:ro

# 2. Python应用时区
import os
os.environ['TZ'] = 'Asia/Shanghai'
# 或设置系统时区
```

### 问题: File upload fails (413 Payload Too Large)

**症状**: 上传文件时API返回413。

**原因**: FastAPI默认限制1MB。

**增加限制**:
```python
# app/api/main.py
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.base import BaseHTTPMiddleware

class LimitUploadSizeMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        content_length = request.headers.get('content-length')
        if content_length and int(content_length) > 100 * 1024 * 1024:  # 100MB
            return JSONResponse({'error': 'File too large'}, status_code=413)
        return await call_next(request)

app.add_middleware(LimitUploadSizeMiddleware)
```

### 问题: Circular import errors on startup

**症状**: `ImportError: cannot import name 'xxx' from partially initialized module`.

**解决**:
```python
# 使用延迟导入 (inside function)
def my_func():
    from app.some_module import Something
    ...

# 或重构代码消除循环依赖
# 将共享代码移到独立模块
```

---

**诊断流程总结**:
1. **服务状态**: `docker compose ps`, `curl /health`
2. **日志**: `docker compose logs -f <service>`
3. **环境**: `env | grep -E 'MYSQL|REDIS|DEBUG'`
4. **连接**: `nc -zv localhost 8000`, `redis-cli ping`
5. **容器内测试**: `docker compose exec api bash` 手动运行

如果问题未解决，请收集:
- `docker compose config` (掩码密码)
- `docker compose logs --tail=200 <service>`
- `docker version && docker compose version`
- OS和内核版本 (`uname -a`)

---

**最后更新**: 2026-03-02  
**维护者**: @operator
