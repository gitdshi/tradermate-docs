# TraderMate 生产环境部署手册

**版本**: 1.0  
**最后更新**: 2026-03-03  
**适用范围**: Staging 和 Production 环境

---

## 目录

1. [架构概览](#架构概览)
2. [部署方式选择](#部署方式选择)
3. [Docker Compose 生产部署](#docker-compose-生产部署)
4. [Kubernetes 部署指南](#kubernetes-部署指南)
5. [裸机部署方案](#裸机部署方案)
6. [环境变量配置](#环境变量配置)
7. [初始化与迁移](#初始化与迁移)
8. [验证与测试](#验证与测试)
9. [回滚策略](#回滚策略)
10. [维护操作](#维护操作)

---

## 架构概览

TraderMate 采用微服务架构，包含以下核心组件：

```
┌─────────────────────────────────────────────────────────────┐
│                        Load Balancer (Nginx)               │
│                  (SSL Termination + Routing)              │
├─────────────┬─────────────┬─────────────┬─────────────────┤
│   API       │   Worker    │  Frontend   │   MySQL         │
│  (8000)     │  (Redis)    │   (5173)    │   (3306)        │
│             │   Queue     │             │                 │
└─────────────┴─────────────┴─────────────┴─────────────────┘
```

### 服务清单

| 服务 | 端口 | 说明 | 生产配置建议 |
|------|------|------|--------------|
| `api` | 8000 | FastAPI 后端 | 2+ 实例，启用 Gunicorn |
| `worker` | - | RQ 后台 worker | 2+ 实例，按队列扩展 |
| `tradermate` | - | VeighNa 桌面应用 | 1 实例 (可选，用于策略开发) |
| `mysql` | 3306 | 主数据库 | 启用主从复制，定期备份 |
| `redis` | 6379 | 任务队列 + 缓存 | 启用持久化，考虑集群 |
| `portal` | 80/443 | 前端 (Nginx) | 静态资源 + 反向代理 API |

---

## 部署方式选择

### 方案对比

| 特性 | Docker Compose | Kubernetes | 裸机 (systemd) |
|------|---------------|------------|----------------|
| **复杂度** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **可扩展性** | 中等 | 高 | 低 |
| **高可用** | 需手动 | 原生支持 | 需手动 |
| **学习曲线** | 低 | 高 | 中等 |
| **推荐场景** | 初期生产、staging | 大规模生产 | 无容器环境 |

### 推荐策略

- **Staging 环境**: Docker Compose (快速迭代)
- **Production 小型**: Docker Compose + Traefik/Nginx (≤ 千万级交易)
- **Production 大型**: Kubernetes (需要弹性伸缩)

---

## Docker Compose 生产部署

### 3.1 准备生产配置文件

创建 `docker-compose.prod.yml` 覆盖开发配置：

```yaml
version: '3.4'

services:
  api:
    image: ${DOCKER_REGISTRY}/tradermate-api:${IMAGE_TAG:-latest}
    restart: always
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      rollback_config:
        parallelism: 0
    environment:
      - DEBUG=false
      - APP_ENV=production
      # 其他环境变量通过 .env.prod 注入
    volumes:
      - ./logs:/app/logs:ro  # 只读挂载代码，日志写入 stdout
    networks:
      - traefik_public
      - tradermate_network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.tradermate-api.rule=Host(`${API_HOST}`)"
      - "traefik.http.routers.tradermate-api.entrypoints=websecure"
      - "traefik.http.routers.tradermate-api.tls.certresolver=myresolver"
      - "traefik.http.services.tradermate-api.loadbalancer.server.port=8000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  worker:
    image: ${DOCKER_REGISTRY}/tradermate-api:${IMAGE_TAG:-latest}
    restart: always
    deploy:
      replicas: 2
    environment:
      - DEBUG=false
      - APP_ENV=production
    volumes:
      - ./logs:/app/logs:ro
    networks:
      - tradermate_network
    depends_on:
      - mysql
      - redis

  mysql:
    image: mysql:8.0
    restart: always
    command: --default-authentication-plugin=mysql_native_password --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/backup:/backup
      - ./mysql/init:/docker-entrypoint-initdb.d:ro
    networks:
      - tradermate_network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 20s
      timeout: 5s
      retries: 5
      start_period: 60s
    # 生产环境建议使用命名卷或外部存储
    # deploy:
    #   placement:
    #     constraints:
    #       - node.role == database

  redis:
    image: redis:7-alpine
    restart: always
    command: redis-server --appendonly yes --save 60 1 --loglevel warning
    volumes:
      - redis_data:/data
    networks:
      - tradermate_network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 20s
      timeout: 5s
      retries: 5

  portal:
    image: ${DOCKER_REGISTRY}/tradermate-portal:${IMAGE_TAG:-latest}
    restart: always
    build:
      context: ../tradermate-portal
      dockerfile: Dockerfile
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
    networks:
      - traefik_public
    # 如果使用 Traefik，移除 ports 并使用 labels
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.tradermate.rule=Host(`${FRONTEND_HOST}`)"
      - "traefik.http.routers.tradermate.entrypoints=websecure"
      - "traefik.http.routers.tradermate.tls.certresolver=myresolver"

volumes:
  mysql_data:
    driver: local
  redis_data:
    driver: local

networks:
  tradermate_network:
    driver: bridge
  traefik_public:
    external: true
```

### 3.2 环境变量管理

创建 `.env.prod`（**切勿提交到Git**）：

```bash
# Docker Registry
DOCKER_REGISTRY=ghcr.io/yourorg  # 或 yourusername
IMAGE_TAG=1.0.0

# 域名配置
API_HOST=api.tradermate.com
FRONTEND_HOST=www.tradermate.com

# 数据库
MYSQL_ROOT_PASSWORD=supersecret-root-password-32chars-min
MYSQL_PASSWORD=supersecret-app-password-32chars-min

# 应用配置
SECRET_KEY=your-jwt-secret-64-chars-minimum
TUSHARE_TOKEN=your-tushare-token
ADMIN_PASSWORD=initial-admin-password
ADMIN_EMAIL=admin@tradermate.com

# 可选：VNPY 数据源配置
VN_DATAFEED_NAME=tushare
```

**⚠️ 安全警告**：
- 生产密码必须 ≥ 32 字符，使用密码管理器生成
- 使用 HTTPS 并配置 TLS 证书 (Let's Encrypt 推荐)
- 限制 MySQL/Redis 端口不暴露到公网

### 3.3 启动生产集群

```bash
# 1. 登录 Docker Registry
docker login ${DOCKER_REGISTRY}

# 2. 拉取或构建镜像 (CI/CD 已推送)
docker-compose -f docker-compose.yml -f docker-compose.prod.yml pull

# 3. 初始化网络 (如果使用 Traefik)
docker network create traefik_public

# 4. 启动服务
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 5. 检查状态
docker-compose ps
docker-compose logs -f api

# 6. 验证健康检查
curl -f https://${API_HOST}/health || echo "API not ready"
```

### 3.4 扩展与收缩

```bash
# 扩展 API 实例数 (docker-compose v2.5+)
docker-compose up -d --scale api=3

# 或编辑 docker-compose.prod.yml 中的 replicas，然后重新部署
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --no-recreate
```

---

## Kubernetes 部署指南

> 当前仓库缺少 `k8s/` 目录，需要创建清单文件。

### 4.1 清单结构

```
k8s/
├── base/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── mysql/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── pvc.yaml
│   ├── redis/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── pvc.yaml
│   ├── api/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   └── ingress.yaml
│   └── worker/
│       ├── deployment.yaml
│       └── service.yaml
├── staging/
│   └── kustomization.yaml
└── production/
    └── kustomization.yaml
```

### 4.2 核心资源示例

#### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tradermate
  labels:
    environment: production
```

#### API Deployment (使用 ConfigMap + Secret)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tradermate-api
  namespace: tradermate
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tradermate-api
  template:
    metadata:
      labels:
        app: tradermate-api
    spec:
      containers:
      - name: api
        image: ${DOCKER_REGISTRY}/tradermate-api:${IMAGE_TAG}
        ports:
        - containerPort: 8000
        envFrom:
        - configMapRef:
            name: tradermate-config
        - secretRef:
            name: tradermate-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 90
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 5
```

#### Ingress (TLS)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tradermate-ingress
  namespace: tradermate
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - api.tradermate.com
    - www.tradermate.com
    secretName: tradermate-tls
  rules:
  - host: api.tradermate.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tradermate-api-service
            port:
              number: 8000
  - host: www.tradermate.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tradermate-portal-service
            port:
              number: 80
```

### 4.3 部署流程

```bash
# 1. 配置 kubectl 上下文
kubectl config use-context production-cluster

# 2. 创建命名空间和配置
kustomize build k8s/base | kubectl apply -f -

# 3. 创建 Secret (从 .env.prod 转换)
kubectl create secret generic tradermate-secret \
  --namespace tradermate \
  --from-env-file=.env.prod

# 4. 部署 (staging)
kubectl apply -k k8s/staging

# 5. 部署 (production)
kubectl apply -k k8s/production

# 6. 验证
kubectl get pods -n tradermate
kubectl logs -f deployment/tradermate-api -n tradermate

# 7. 等待所有 Pod Ready
kubectl wait --for=condition=Ready pod --all -n tradermate --timeout=300s
```

### 4.4 CI/CD 集成

在 `.github/workflows/deploy.yml` 中使用 Azure/k8s-deploy action（当前配置已存在，只需补充 `k8s/` 清单）。

---

## 裸机部署方案 (Systemd)

适用于无 Docker/K8s 环境，直接运行 Python 服务。

### 5.1 目录结构

```
/opt/tradermate/
├── tradermate/          # 代码目录 (git clone)
├── venv/               # Python 虚拟环境
├── logs/              # 应用日志
├── run/
│   ├── api.service
│   ├── worker.service
│   └── tradermate.service
├── .env                # 生产环境变量
└── mysql/
    ├── data/          # MySQL 数据目录
    └── init/          # 初始化脚本
```

### 5.2 Systemd Unit 文件

**api.service**:

```ini
[Unit]
Description=TraderMate API
After=network.target mysql.service redis.service
Wants=mysql.service redis.service

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/opt/tradermate/tradermate
EnvironmentFile=/opt/tradermate/.env
ExecStart=/opt/tradermate/venv/bin/uvicorn app.api.main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=10

# 资源限制
LimitNOFILE=65536
TimeoutStopSec=30

# 日志
StandardOutput=journal
StandardError=journal
SyslogIdentifier=tradermate-api

[Install]
WantedBy=multi-user.target
```

**worker.service**:

```ini
[Unit]
Description=TraderMate Worker
After=network.target mysql.service redis.service
Wants=mysql.service redis.service

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/opt/tradermate/tradermate
EnvironmentFile=/opt/tradermate/.env
ExecStart=/opt/tradermate/venv/bin/rq worker --url redis://localhost:6379 backtest optimization
Restart=always
RestartSec=10
SyslogIdentifier=tradermate-worker

[Install]
WantedBy=multi-user.target
```

**mysql.service** (如果使用系统 MySQL):

```ini
# Skip if using Docker MySQL
```

### 5.3 安装与启动

```bash
# 1. 准备环境
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3-pip mysql-server redis-server

# 2. 克隆代码
cd /opt/tradermate
git clone <repo> tradermate

# 3. 创建虚拟环境
python3.11 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r tradermate/requirements.txt

# 4. 配置环境
cp tradermate/.env.example .env
# 编辑 .env，设置数据库密码等

# 5. 启动 MySQL/Redis
sudo systemctl start mysql
sudo systemctl enable mysql
sudo systemctl start redis
sudo systemctl enable redis

# 6. 初始化数据库
mysql -u root -p < tradermate/mysql/init/tradermate.sql
mysql -u root -p < tradermate/mysql/init/vnpy.sql
mysql -u root -p < tradermate/mysql/init/tushare.sql
mysql -u root -p < tradermate/mysql/init/akshare.sql

# 7. 加载 systemd unit
sudo cp run/*.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable tradermate-api.timer tradermate-worker.timer
sudo systemctl start tradermate-api tradermate-worker

# 8. 启动前端 (需要 Node.js)
cd /opt/tradermate/tradermate-portal
npm ci --only=production
npm run build
# 配置 Nginx 指向 dist/ 目录
```

---

## 环境变量配置

### 6.1 完整变量列表

参考 `.env.example` 文件，生产环境必须设置：

| 变量名 | 必需 | 默认 | 说明 |
|--------|------|------|------|
| `MYSQL_HOST` | 是 | `mysql` | MySQL 主机 |
| `MYSQL_PORT` | 否 | `3306` | MySQL 端口 |
| `MYSQL_USER` | 是 | `root` | MySQL 用户名 |
| `MYSQL_PASSWORD` | **是** | - | MySQL 密码（≥32字符） |
| `TUSHARE_TOKEN` | 否 | - | Tushare API token |
| `SECRET_KEY` | **是** | - | JWT 密钥（≥64字符） |
| `ADMIN_PASSWORD` | 是（生产） | - | 初始管理员密码 |
| `ADMIN_EMAIL` | 是 | - | 管理员邮箱 |
| `REDIS_HOST` | 是 | `redis` | Redis 主机 |
| `REDIS_PORT` | 否 | `6379` | Redis 端口 |
| `CORS_ORIGINS` | 否 | 开发配置 | 生产应设为前端域名 |

### 6.2 环境隔离

- **Staging**: 使用独立数据库，复制生产数据脱敏
- **Production**: 严格隔离，不同 VPC/子网

---

## 初始化与迁移

### 7.1 数据库初始化

首次部署执行：

```bash
# 方法1: Docker Compose (自动执行 mysql/init/*.sql)
docker-compose up mysql

# 方法2: 手动执行
mysql -h ${MYSQL_HOST} -u root -p${MYSQL_PASSWORD} < mysql/init/tradermate.sql
mysql -h ${MYSQL_HOST} -u root -p${MYSQL_PASSWORD} < mysql/init/vnpy.sql
mysql -h ${MYSQL_HOST} -u root -p${MYSQL_PASSWORD} < mysql/init/tushare.sql
mysql -h ${MYSQL_HOST} -u root -p${MYSQL_PASSWORD} < mysql/init/akshare.sql
```

### 7.2 迁移策略

当前项目**无 Alembic/Flyway**，DB 变更需：
1. 修改 `mysql/init/*.sql`（新增或变更表）
2. 对现有环境手动执行增量 SQL
3. 在 DEPLOYMENT.md 记录每次变更

**未来建议**: 引入 Alembic 进行版本化迁移。

---

## 验证与测试

### 8.1 健康检查端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/health` | GET | 综合健康（DB + Redis） |
| `/ready` | GET | 就绪检查（所有依赖就绪） |
| `/metrics` | GET | Prometheus 指标（需配置） |

### 8.2 部署后检查清单

- [ ] 所有容器/Pod 状态为 `healthy`
- [ ] Nginx/Traefik 路由正确
- [ ] HTTPS 证书有效
- [ ] `/health` 返回 `{"status":"healthy"}`
- [ ] 前端可访问并成功调用 API
- [ ] 日志无错误（`docker-compose logs` 或 `kubectl logs`）
- [ ] 数据库备份任务已配置
- [ ] 监控告警规则已导入

### 8.3 端到端测试

```bash
# API 功能测试
curl -f https://api.tradermate.com/health
curl -f https://api.tradermate.com/docs  # Swagger UI

# 前端测试
curl -f https://www.tradermate.com/ | grep -i "<title>"

# 登录流程测试
TOKEN=$(curl -X POST https://api.tradermate.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"your-password"}' | jq -r .access_token)
curl -H "Authorization: Bearer $TOKEN" https://api.tradermate.com/api/auth/me
```

---

## 回滚策略

### 9.1 Docker Compose 回滚

```bash
# 1. 拉取上一个稳定版本镜像
docker-compose -f docker-compose.yml -f docker-compose.prod.yml pull tradermate-api@previous-tag

# 2. 重新部署（指定旧版本IMAGE_TAG）
IMAGE_TAG=v1.0.0 docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 3. 验证健康后，更新 .env 中的 IMAGE_TAG
```

### 9.2 Kubernetes 回滚

```bash
# 查看发布历史
kubectl rollout history deployment/tradermate-api -n tradermate

# 回滚到上一个版本
kubectl rollout undo deployment/tradermate-api -n tradermate

# 回滚到指定版本
kubectl rollout undo deployment/tradermate-api -n tradermate --to-revision=2

# 监控状态
kubectl rollout status deployment/tradermate-api -n tradermate
```

### 9.3 数据库回滚

- **依赖应用层备份**: 每次 DB 变更前手动备份
- **恢复命令**: `mysql < backup.sql`
- **建议**: 使用 pt-online-schema-change 进行在线变更

---

## 维护操作

### 10.1 日志查看

```bash
# Docker Compose
docker-compose logs -f --tail=100 api
docker-compose logs api > api-$(date +%F).log

# Kubernetes
kubectl logs -f deployment/tradermate-api -n tradermate --tail=100

# 裸机 (journalctl)
journalctl -u tradermate-api -f
```

### 10.2 备份恢复

见 `BACKUP_RECOVERY_PLAN.md`。

### 10.3 证书更新 (Let's Encrypt)

```bash
# 如果使用 Traefik，自动续期
# 如果使用 Nginx + certbot
sudo certbot renew --dry-run
sudo systemctl reload nginx
```

### 10.4 容量扩展

- **API 实例**: 增加 `replicas` (K8s HPA 可自动)
- **数据库**: 主从复制 + 读写分离（需应用层支持）
- **Redis**: 集群模式（≥6节点）

---

## 联系与支持

- 部署问题: 联系 @operator
- 应用问题: 联系 @coder
- 监控告警: 联系 @tester

---

**文档结束**
