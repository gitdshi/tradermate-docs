# DEPLOYMENT.md

**Version**: 1.0  
**Status**: Final (Production Ready)  
**Task**: P0-DEP-001  
**Last Updated**: 2026-03-08 16:00 UTC  
**Audience**: Ops, DevOps, SRE

---

## 1. Quick Deploy (最短可执行部署路径)

适用于紧急发布，跳过文档阅读。生产环境使用 Docker Compose + Traefik。

```bash
# 0) 前置准备（一次性）
# - 已安装 Docker、Docker Compose v2.5+
# - 已创建 .env.prod（包含所有密码和域名）
# - Traefik 网络已存在（或改用 nginx labels）
docker network create traefik_public || true

# 1) 拉取或构建镜像（CI/CD已完成推送）
docker-compose -f docker-compose.yml -f docker-compose.prod.yml pull

# 2) 初始化数据库（仅首次或变更）
# - 确保 MYSQL_USER=tradermate 专用用户已创建（见第6.3节）
# - 初始化脚本自动执行（首次启动容器时）
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d mysql
sleep 30  # 等待MySQL就绪

# 3) 发布应用
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 4) 观察启动
docker-compose logs -f api
# 期望：无数据库连接错误，Gunicorn启动成功

# 5) 快速验证
curl -f https://api.tradermate.com/health || { echo "健康检查失败"; exit 1; }
curl -f https://www.tradermate.com/ | grep -i "<title>" || { echo "前端不可用"; exit 1; }

# 6) 完成
echo "部署成功。监控告警请查看 MONITORING_CONFIG.md"
```

**快速回滚（5分钟内）**:

```bash
# Docker Compose: 回滚到上一个稳定标签
IMAGE_TAG=v1.0.0 docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Kubernetes: 回滚到上一个版本
kubectl rollout undo deployment/tradermate-api -n tradermate
```

---

## 2. 前置条件

### 2.1 发布窗口与审批
- [ ] 发布窗口已确定（避开市场高波动期，建议非交易时段）
- [ ] 变更已通过 Code Review 并合并到目标分支（release/v1.x）
- [ ] 镜像已推送到 Registry 并带有语义化版本标签（如 `v1.2.3`）

### 2.2 配置准备
- [ ] `.env.prod` 文件已创建于部署主机（**不提交到Git**），内容参考第6节
- [ ] 所有密码满足复杂度要求（≥32字符，包含大小写、数字、特殊符号）
- [ ] Traefik/Nginx Ingress 已配置 TLS 证书（Let's Encrypt 或企业 CA）
- [ ] 域名 DNS 已指向负载均衡器 IP

### 2.3 基础设施
- [ ] 服务器资源充足（CPU/内存/磁盘预留30% buffer）
- [ ] MySQL 8.0 和 Redis 7 实例已就绪，网络互通
- [ ] 备份策略已启用（每日全备 + binlog 实时备份）
- [ ] 监控系统（Prometheus+Loki+Grafana）运行正常
- [ ] 告警通道（Slack/钉钉/邮件）已验证

### 2.4 回滚预案
- [ ] 上一稳定版本镜像标签已记录（如 `v1.2.2`）
- [ ] 回滚脚本已测试（见第5节）
- [ ] 数据库备份已完成（在部署前执行）

---

## 3. 部署步骤

### 3.1 Staging 预演（必须）

Staging 环境必须与 Production 配置一致（除数据量级外）。

```bash
# A. 准备 staging 配置
cp .env.example .env.staging
# 编辑 .env.staging，覆盖：
#   MYSQL_HOST=staging-mysql.internal
#   MYSQL_PASSWORD=<strong-password>
#   REDIS_PASSWORD=<strong-password>
#   API_HOST=api.staging.tradermate.com
#   FRONTEND_HOST=www.staging.tradermate.com
#   DEBUG=false
#   APP_ENV=production

# B. 启动服务
docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d

# C. 等待就绪（所有容器 healthy）
docker-compose ps  # 确认状态为 "healthy"
docker-compose logs -f api  # 观察错误

# D. 执行冒烟测试（见第4节）
# E. 记录结果到 staging-validation-<timestamp>.md
```

**通过标准**:
- `/health` 返回 `{"status":"healthy"}`
- `/api/auth/login` 成功返回 token
- 核心业务接口（如 `/api/strategies`）返回 200
- 无 error/warning 级日志

### 3.2 Production 发布

```bash
# 0) 冻结发布窗口（通知所有团队停止变更）
# 1) 备份数据库（关键！）
docker exec tradermate_mysql mysqldump -u root -p${MYSQL_ROOT_PASSWORD} --all-databases > backup/full-$(date +%F-%H%M).sql

# 2) 拉取镜像（确保版本一致）
IMAGE_TAG=v1.2.3 docker-compose -f docker-compose.yml -f docker-compose.prod.yml pull

# 3) 部署（滚动更新）
IMAGE_TAG=v1.2.3 docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 4) 观察新容器启动
docker-compose logs -f --tail=100 api | grep -E "error|exception|startup complete" &
# 等待约 60-90 秒

# 5) 验证新实例健康
docker-compose ps  # 确认新容器 healthy，旧容器仍在运行
curl -f https://api.tradermate.com/health  # 应返回 200

# 6) 移除旧容器（如果使用 start-first 策略，旧容器可能已自动停止）
docker-compose ps  # 确认只有新版本容器在运行

# 7) 完成并记录
echo "Production deployment complete at $(date -u +%Y-%m-%dT%H:%M:%SZ)" >> deployment.log
```

**注意**:
- 生产环境使用 `deploy.update_config.order: start-first` 确保零停机
- 数据库端口不暴露，仅容器网络内访问
- `/health` 端点**必须公开**（无认证依赖），否则 Traefik 健康检查失败（O-002 已知问题修复）

### 3.3 数据库初始化与迁移

#### 3.3.1 首次初始化

```bash
# 方法A: Docker Compose 自动执行（推荐）
# 将 init 脚本放入 ./mysql/init/ 目录：
#   - tradermate.sql (主业务表)
#   - vnpy.sql (VeighNa 框架表)
#   - tushare.sql (Tushare 数据表结构)
#   - akshare.sql (AkShare 辅助表)
docker-compose up mysql  # 容器首次启动时自动执行所有 .sql 文件

# 方法B: 手动执行（裸机或已有数据）
mysql -h mysql -u root -p < mysql/init/tradermate.sql
mysql -h mysql -u root -p < mysql/init/vnpy.sql
mysql -h mysql -u root -p < mysql/init/tushare.sql
mysql -h mysql -u root -p < mysql/init/akshare.sql
```

#### 3.3.2 增量迁移（当前无版本工具）

由于项目未使用 Alembic/Flyway，DB 变更需：

1. 在 `mysql/init/` 目录下创建 `migration-YYYY-MM-DD-<描述>.sql`
2. 在部署步骤的**验证通过后**、业务上线前，对生产环境手动执行：
   ```bash
   mysql -h mysql -u root -p < mysql/init/migration-2026-03-10-add-index.sql
   ```
3. 在 `DEPLOYMENT.md` 证据记录中注明执行了哪些迁移脚本

**强烈建议**: 引入 Alembic 进行版本化管理（未来迭代）

---

## 4. 验证步骤

### 4.1 健康检查

```bash
# API 健康检查（必须公开，无需认证）
curl -s -o /dev/null -w "%{http_code}" https://api.tradermate.com/health
# 期望输出: 200

# JSON 内容验证
curl -s https://api.tradermate.com/health | jq '.status'
# 期望输出: "healthy"
```

### 4.2 核心功能测试

```bash
# 1. Swagger UI 可达
curl -f https://api.tradermate.com/docs | grep -i "swagger"

# 2. 登录获取 token
TOKEN=$(curl -s -X POST https://api.tradermate.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<ADMIN_PASSWORD_FROM_.env>"}' | jq -r .access_token)
if [ -z "$TOKEN" ] || [ "$TOKEN" = "null" ]; then echo "登录失败"; exit 1; fi

# 3. 验证 token 有效（调用 /auth/me）
curl -s -H "Authorization: Bearer $TOKEN" https://api.tradermate.com/api/auth/me | jq '.email'

# 4. 核心业务接口（示例：策略列表）
curl -s -H "Authorization: Bearer $TOKEN" https://api.tradermate.com/api/strategies | jq '.total'

# 5. 数据持久化测试（创建策略）
STRATEGY_ID=$(curl -s -X POST https://api.tradermate.com/api/strategies \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Deploy","type":"backtest"}' | jq -r .id)
# 立即查询确认写入成功
curl -s -H "Authorization: Bearer $TOKEN" https://api.tradermate.com/api/strategies/$STRATEGY_ID | jq '.name'

# 6. 清理测试数据
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" "https://api.tradermate.com/api/strategies/$STRATEGY_ID"
```

### 4.3 前端验证

```bash
# 前端首页加载
curl -s https://www.tradermate.com/ | grep -i "<title>TraderMate</title>"
# 期望：包含标题

# 前端 API 调用（需要浏览器，此处仅检查静态资源）
curl -s -o /dev/null -w "%{http_code}" https://www.tradermate.com/static/js/main.js
# 期望：200
```

### 4.4 依赖连通性

```bash
# 数据库连接（从容器内测试）
docker exec tradermate_api mysql -h mysql -u tradermate -p${MYSQL_PASSWORD} -e "SELECT 1" tradermate
# 期望：1

# Redis 连接
docker exec tradermate_api redis-cli -h redis -a ${REDIS_PASSWORD} ping
# 期望：PONG

# Tushare API（如配置）
curl -s "https://api.tushare.pro/?ts_code=000001.SZ&token=${TUSHARE_TOKEN}" | jq '.ts_code'
# 期望：返回数据或明确的错误码（非认证失败）
```

### 4.5 性能基线（参考值）

| 指标 | 目标值 | 测量方法 |
|------|--------|----------|
| 健康检查延迟 | < 200ms | `time curl /health` |
| 登录接口 P95 | < 500ms | Prometheus `api_auth_login_duration_seconds` |
| 策略列表 P95 | < 1s | 同上 |
| 错误率 | < 0.1% | Prometheus `rate(http_requests_total{status=~"5.."}[5m])` |
| API 内存占用 | < 500Mi | `docker stats` 或 K8s metrics |
| API CPU 占用 | < 500m | 同上 |

### 4.6 日志检查（关键错误）

```bash
# 查看最近100行，筛选严重错误
docker-compose logs --tail=100 api | grep -iE "error|exception|critical|traceback"
# 期望：无输出

# 持续监控5分钟
docker-compose logs -f --since=5m api | grep -iE "error|exception" &
sleep 300
kill $!  # 停止监控
```

---

## 5. 回滚步骤

### 5.1 触发条件（满足任意一条即回滚）

| 条件 | 阈值 | 检查方法 |
|------|------|----------|
| `/health` 持续失败 | 连续3次返回非200，持续1分钟 | `curl` 循环测试 |
| 核心业务错误率 | >1% (5xx) 持续5分钟 | Prometheus `rate(http_requests_total{status=~"5.."}[5m])` |
| API 不可用 | 登录接口超时或错误 >30s | `curl -m 30 ...` |
| 数据库连接池耗尽 | 活跃连接 > 80% 容量 | `SHOW PROCESSLIST;` |
| 用户投诉激增 | 收到 ≥5 个用户反馈不可用 | 客服渠道 |

### 5.2 Docker Compose 回滚（生产环境）

```bash
# 1. 确定上一稳定版本
# 通常标注为 v1.2.2，可通过 git tag 或 docker images 查找
STABLE_TAG=v1.2.2

# 2. 快速回滚（无需先停，docker-compose 会按策略替换）
IMAGE_TAG=$STABLE_TAG docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 3. 观察回滚过程
docker-compose ps  # 确认旧容器停止，新容器启动
docker-compose logs -f --tail=50 api | grep -i "startup complete"

# 4. 验证恢复
curl -f https://api.tradermate.com/health
curl -f -H "Authorization: Bearer $(get_valid_token())" https://api.tradermate.com/api/strategies

# 5. 记录回滚事件
cat >> rollback-$(date +%F-%H%M).log <<EOF
Rollback triggered: $(date -u +%Y-%m-%dT%H:%M:%SZ)
Reason: [填写触发条件]
From version: v1.2.3
To version: $STABLE_TAG
Operator: Frank
Verification: [通过/失败]
EOF
```

### 5.3 Kubernetes 回滚

```bash
# 1. 查看发布历史，确认目标版本
kubectl rollout history deployment/tradermate-api -n tradermate
# 输出示例:
#   deployments "tradermate-api" with revision 1
#   deployments "tradermate-api" with revision 2

# 2. 回滚到上一个版本（默认）
kubectl rollout undo deployment/tradermate-api -n tradermate

# 3. 或回滚到指定版本
kubectl rollout undo deployment/tradermate-api -n tradermate --to-revision=2

# 4. 监控回滚状态
kubectl rollout status deployment/tradermate-api -n tradermate --watch
# 期望: "successfully rolled out"

# 5. 验证（同第4节）
kubectl exec -n tradermate deployment/tradermate-api -- curl -f http://localhost:8000/health

# 6. 记录
kubectl create configmap rollback-record-$(date +%s) --namespace=tradermate \
  --from-literal=timestamp="$(date -u)" \
  --from-literal=to-revision="2" \
  --from-literal=operator="Frank"
```

### 5.4 数据库回滚（仅在数据不一致时需要）

**原则**: 回滚应用版本时**不要**自动回滚数据库。仅在确认新版本写入脏数据或结构破坏时才恢复备份。

```bash
# 1. 确认需要恢复（例如：新版本写入错误数据导致业务不可用）
# 2. 停止所有应用实例
docker-compose down  # 或 kubectl scale deployment tradermate-api --replicas=0

# 3. 恢复数据库（使用第1步创建的备份）
docker exec -i tradermate_mysql mysql -u root -p${MYSQL_ROOT_PASSWORD} < backup/full-2026-03-08-1200.sql

# 4. 验证数据完整
docker exec tradermate_mysql mysql -u root -p${MYSQL_ROOT_PASSWORD} -e "SHOW TABLES;" tradermate

# 5. 重新部署旧版本
IMAGE_TAG=$STABLE_TAG docker-compose up -d

# 6. 验证（第4节）
```

**警告**:
- 数据库回滚会导致新版本写入的数据丢失，需评估影响
- 优先使用应用层补偿（如数据修复脚本）而非直接回滚DB
- 建议使用数据迁移工具（如 Alembic）进行版本化变更，避免破坏性修改

### 5.5 回滚后动作（必须）

1. **事件记录**：更新 `docs/deployment/DEPLOYMENT_LOG.md`，包含：
   - 触发时间、原因、影响用户数
   - 回滚版本、操作人
   - 验证结果
2. **根因分析**：在24小时内与 coder 共同查明失败原因，更新 `CODE_REVIEW_CHECKLIST.md`
3. **预防措施**：改进健康检查、调整告警阈值、补充测试用例
4. **通知**：告知 Main 和相关团队（Tester, Writer），并同步给 Daniel

---

## 6. 环境变量与密钥管理

### 6.1 完整变量清单

生产环境必须设置以下变量（`.env.prod`）：

```bash
# Docker Registry
DOCKER_REGISTRY=ghcr.io/yourorg
IMAGE_TAG=v1.2.3

# 域名配置
API_HOST=api.tradermate.com
FRONTEND_HOST=www.tradermate.com

# 数据库（使用专用用户，非 root）
MYSQL_HOST=mysql
MYSQL_PORT=3306
MYSQL_USER=tradermate
MYSQL_PASSWORD=supersecret-tradermate-password-32chars-minimum
MYSQL_ROOT_PASSWORD=different-strong-root-password-for-backup-only
TUSHARE_DATABASE=tushare
VNPY_DATABASE=vnpy
TUSHARE_DATABASE_URL=mysql+pymysql://tradermate:${MYSQL_PASSWORD}@mysql:3306/tushare?charset=utf8mb4
VNPY_DATABASE_URL=mysql+pymysql://tradermate:${MYSQL_PASSWORD}@mysql:3306/vnpy?charset=utf8mb4
TUSHARE_TOKEN=your-tushare-token

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=supersecret-redis-password-32chars-minimum

# 应用安全
SECRET_KEY=your-jwt-secret-key-at-least-64-chars-long-random
JWT_SECRET_KEY=${SECRET_KEY}
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# 管理员初始账户（首次部署后立即修改）
ADMIN_PASSWORD=initial-admin-password-32chars-minimum
ADMIN_EMAIL=admin@tradermate.com

# CORS（生产仅允许 HTTPS 域名）
CORS_ORIGINS=["https://www.tradermate.com","https://api.tradermate.com"]

# 运行时
DEBUG=false
APP_ENV=production
APP_VERSION=1.2.3
```

### 6.2 安全要求

- **密码长度**: 所有密码 ≥ 32 字符，使用 `openssl rand -base64 32` 或密码管理器生成
- **密钥隔离**: `.env.prod` **绝不能**提交到 Git。已提交的需 `git filter-branch` 删除历史
- **K8s Secret**: 使用 `kubectl create secret generic tradermate-secret --from-env-file=.env.prod`
- **轮换周期**: 每 90 天更换一次应用密码（DBA 配合）
- **最小权限**: 应用数据库用户仅拥有 `tradermate`, `vnpy`, `tushare` 的 SELECT/INSERT/UPDATE/DELETE，无 DROP 权限

### 6.3 创建专用数据库用户

参考 `docs/deployment/docker-compose.prod.yml` 第92-98行的补丁步骤。关键 SQL：

```sql
CREATE USER 'tradermate'@'%' IDENTIFIED BY '<MYSQL_PASSWORD>';
GRANT SELECT, INSERT, UPDATE, DELETE ON tradermate.* TO 'tradermate'@'%';
GRANT SELECT ON vnpy.* TO 'tradermate'@'%';
GRANT SELECT ON tushare.* TO 'tradermate'@'%';
FLUSH PRIVILEGES;
```

**验证**: `mysql -u tradermate -p -h mysql` 可登录，但 `SHOW DATABASES;` 仅显示授权库。

---

## 7. 监控与告警

部署前必须确保监控体系已就绪。本节列出关键观测指标和告警规则，完整配置见 `MONITORING_CONFIG.md`。

### 7.1 监控栈概览

推荐使用 **Prometheus + Grafana + Alertmanager** 组合，采集主机、容器、数据库和应用指标。所需 exporters：
- node_exporter (主机)
- cAdvisor (容器)
- mysqld_exporter (MySQL)
- redis_exporter (Redis)
- FastAPI 应用需集成 `prometheus_client` 暴露 `/metrics`

参考 `DEPLOYMENT_RUNBOOKS.md` 中的监控部署步骤。

### 7.2 部署后关键指标

| 指标类别 | 关键指标 | 正常阈值 | 告警阈值 |
|----------|----------|----------|----------|
| 系统 | CPU 使用率 | < 70% | > 85% 持续 5m |
| 系统 | 内存使用率 | < 80% | > 90% 持续 5m |
| 系统 | 磁盘使用率 | < 80% | > 85% 持续 1h |
| MySQL | 连接数 | < max_connections * 0.8 | > 80% 容量持续 5m |
| MySQL | 慢查询 | 0 | > 10/分钟 |
| MySQL | InnoDB Buffer Pool 命中率 | > 99% | < 95% 持续 10m |
| Redis | 内存使用率 | < 80% | > 85% 持续 5m |
| Redis | 命中率 | > 90% | < 80% 持续 10m |
| API | HTTP 错误率 (5xx) | < 0.1% | > 1% 持续 5m |
| API | P95 延迟 | < 1s | > 2s 持续 5m |
| /health | 可用性 | 100% | 连续 3 次失败 |

### 7.3 Datasync 专项告警 (P1-DSYNC-OPS-001)

#### 7.3.1 断点信息读写失败

**日志关键字**:
- `"Checkpoint read failed"`
- `"Checkpoint write timeout"`
- `"Failed to persist checkpoint"`
- `"IOError.*checkpoint"`

**指标**:
- `datasync_checkpoint_errors_total` (counter)
- `datasync_checkpoint_write_duration_seconds` (histogram)
- `datasync_checkpoint_size_bytes` (gauge)

**告警规则**:
```yaml
- alert: DataSyncCheckpointWriteFailed
  expr: increase(datasync_checkpoint_errors_total[5m]) > 0
  for: 2m
  annotations:
    summary: "DataSync 断点写入失败"
    description: "{{ $labels.instance }} 在过去5分钟出现 {{ $value }} 次断点写入错误。"
```

#### 7.3.2 限流重试异常

**日志关键字**:
- `"Rate limit exceeded"`
- `"Retrying after`
- `"429 Too Many Requests"`
- `"Backoff exhausted"`

**指标**:
- `datasync_api_retries_total` (counter, labels: endpoint)
- `datasync_api_retry_duration_seconds` (histogram)
- `datasync_rate_limit_errors_total` (counter)

**告警规则**:
```yaml
- alert: DataSyncHighRetryRate
  expr: rate(datasync_api_retries_total[5m]) > 0.5
  for: 3m
  annotations:
    summary: "DataSync 重试率过高"
    description: "API 重试率 {{ $value | humanizePercentage }} 超过 50%。"
```

#### 7.3.3 异常重试风暴

**风暴识别**: 单个实例在短时间内产生大量重试，可能导致依赖服务雪崩。

**指标**:
- `datasync_concurrent_retries` (gauge)
- `datasync_retry_queue_length` (gauge)

**告警规则**:
```yaml
- alert: DataSyncRetryStorm
  expr: datasync_concurrent_retries > 100
  for: 1m
  annotations:
    summary: "检测到 DataSync 重试风暴"
    description: "{{ $labels.instance }} 当前并发重试数 {{ $value }}，可能引发雪崩。"
```

**缓解措施**:
1. 立即触发回滚（见第5节）
2. 或动态调整重试参数（如指数退避 base 增大）
3. 通知 coder 修复重试逻辑（引入抖动和硬超时）

### 7.4 常用告警规则示例

基于 `MONITORING_CONFIG.md`，关键规则包括：

```yaml
- alert: APIHighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.01
  for: 5m
  annotations:
    summary: "API 错误率过高"
    description: "{{ $labels.instance }} 5xx 错误率 {{ $value | humanizePercentage }}"

- alert: APIHighLatency
  expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
  for: 5m
  annotations:
    summary: "API 延迟 P95 超标"

- alert: MySQLDown
  expr: up{job="mysql"} == 0
  for: 1m
  annotations:
    summary: "MySQL 实例宕机"

- alert: RedisMemoryHigh
  expr: redis_memory_usage_bytes / redis_total_system_memory_bytes > 0.85
  for: 5m
  annotations:
    summary: "Redis 内存使用率超过 85%"
```

更多规则请参考 `docs/deployment/MONITORING_CONFIG.md`。

---

## 8. 备份与恢复

完整方案见 `BACKUP_RECOVERY_PLAN.md`。

### 8.1 RTO / RPO 目标

- **RTO（恢复时间目标）**: 4 小时
- **RPO（恢复点目标）**:
  - MySQL: 24 小时（配合 binlog 可达到秒级）
  - Redis: 1 小时

### 8.2 备份策略与保留

| 组件 | 备份类型 | 频率 | 保留期 |
|------|----------|------|--------|
| MySQL | 全量 + binlog | 每日 02:00 + 实时 | 全量: 30 天（日）+ 12 周（周）+ 12 月（月）；Binlog: 7 天 |
| Redis | RDB + AOF | 每小时 RDB + 实时 AOF | 7 天 |
| 代码/配置 | Git | 每次变更 | 永久 |
| 日志 | 压缩归档 | 每日 | 90 天 |

### 8.3 快速恢复命令

**MySQL**（从最新全量备份恢复）：

```bash
docker-compose stop mysql
docker exec tradermate_mysql rm -rf /var/lib/mysql/*
docker exec -i tradermate_mysql sh -c "
  mkdir -p /tmp/restore &&
  tar -xzf /backup/mysql/full-YYYY-MM-DD.tar.gz -C /tmp/restore &&
  xtrabackup --prepare --target-dir=/tmp/restore/backup &&
  xtrabackup --copy-back --target-dir=/tmp/restore/backup
"
docker exec tradermate_mysql chown -R mysql:mysql /var/lib/mysql
docker-compose start mysql
# 如需 PITR，应用相应 binlog
```

**Redis**:

```bash
docker-compose stop redis
cp /backup/redis/dump.rdb /var/lib/docker/volumes/tradermate_redis_data/_data/dump.rdb
docker-compose start redis
```

### 8.4 验证与演练频率

- **每周**: 验证备份完整性（`gzip -t` 测试，检查备份日志）。
- **每季度**: 执行完整的灾难恢复演练，记录实际 RTO/RPO，必要时调整本计划。

更多详细步骤（包括 Percona XtraBackup 参数、云存储同步等）请参考 `docs/deployment/BACKUP_RECOVERY_PLAN.md`。

---

## 9. 安全检查清单

部署前必须完成所有 🔴 项。下表为关键项摘要，完整 42 项见 `SECURITY_CHECKLIST.md`。

### 9.1 网络安全
- 🔴 **网络隔离**: MySQL 3306 与 Redis 6379 禁止公网访问，仅应用容器可访问。
- 🔴 **防火墙**: 安全组仅开放 80/443（任意）、22（管理IP）、以及容器间通信端口。
- 🔴 **TLS**: 全站 HTTPS 强制，TLS 1.2+，启用 HSTS。
- 🟡 **WAF**: 建议启用 Cloudflare 或 Nginx ModSecurity 防护。

### 9.2 密钥与 Secrets 管理
- 🔴 **密钥不入库**: `.env.prod` 加入 `.gitignore`；已泄漏密钥立即轮换。
- 🔴 **强密码**: 所有密码 ≥32 字符，JWT Secret ≥64 字符，使用 `openssl rand -base64 32` 生成。
- 🔴 **K8s Secret**: Kubernetes 部署使用 Secret 对象，避免 ConfigMap。
- 🟢 **定期轮换**: 每 90 天轮换一次应用与数据库密码。

### 9.3 应用加固
- 🔴 **健康检查公开**: `/health` 端点无需认证可直接访问。
- 🔴 **关闭调试**: `DEBUG=false` 生产环境；验证日志不泄露敏感信息。
- 🔴 **CORS 限制**: `CORS_ORIGINS` 仅包含生产域名。
- 🔴 **管理员密码**: 首次部署后立即修改默认管理员密码。
- 🟡 **登录限流**: 防止暴力破解（例如 5 次/分钟限制）。

### 9.4 数据库安全
- 🔴 **专用用户**: 应用使用 `tradermate` 而非 `root`。
- 🔴 **最小权限**: `tradermate` 仅拥有所需库的 SELECT/INSERT/UPDATE/DELETE 权限。
- 🔴 **加密存储**: 敏感数据（如用户凭证、交易记录）启用磁盘加密或云 KMS。
- 🟡 **审计日志**: 启用 MySQL audit 插件记录关键操作。

### 9.5 容器与主机安全
- 🔴 **非 root 运行**: 容器内用户UID ≠ 0；检查 `docker inspect`。
- 🔴 **只读文件系统**: 代码和配置挂载为 `:ro`。
- 🔴 **资源限制**: 设置内存和 CPU 配额，防止 DoS。
- 🟡 **镜像扫描**: 每次构建使用 Trivy 扫描，高危漏洞必须修复。

### 9.6 主机 OS 加固
- 🔴 **SSH 加固**: `PermitRootLogin no`, `PasswordAuthentication no`，仅密钥登录。
- 🔴 **防暴力破解**: 安装并配置 fail2ban。
- 🔴 **时间同步**: NTP 服务运行中。
- 🟢 **内核强化**: 设置 `net.ipv4.tcp_syncookies=1` 等参数。

完整清单（含 42 项）与详细说明请参阅 `docs/deployment/SECURITY_CHECKLIST.md`。

---

## 10. 证据模板（含示例）

每次部署完成后，在 `docs/deployment/deployments/` 创建文件 `deploy-YYYY-MM-DD-HHMM.md`。

### 10.1 模板

```markdown
---
version: <语义化版本>
prev_version: <上一稳定版本>
deployment_team: Operator (Frank)
deployer: [姓名]
deployment_date: [UTC 时间]
window: [选择的维护窗口]
status: ✅ 成功 / ❌ 失败（已回滚）
rollback_performed: 是/否
rollback_reason: [若回滚]

pre_flight_checks:
  backup_taken: 是
  staging_validated: 是
  monitoring_healthy: 是

health_check:
  /health: <HTTP 状态码>
  /docs: <HTTP 状态码>
  login: <HTTP 状态码>

verification_results:
  core_apis: 通过/失败
  e2e_flow: 通过/失败
  performance_metrics:
    p95_latency: "<值>"
    error_rate: "<值>"

incident_log:
  - 时间: ...
    问题: ...
    处理: ...

lessons_learned: [可选]
---

附件：
- docker-compose 输出: logs/docker-compose-<时间戳>.log
- 监控截图: Grafana dashboard 链接
- 告警记录: Alertmanager 链接
```

### 10.2 填写示例（O-002 部署，2026-03-03）

文件：`docs/deployment/deployments/deploy-2026-03-03-0250.md`

```markdown
---
version: v1.0.0-rc.2
prev_version: v1.0.0-rc.1
deployment_team: Operator (Frank)
deployer: Frank
deployment_date: 2026-03-03 02:50 UTC
window: 02:00-03:00 UTC（低流量期）
status: ✅ 成功
rollback_performed: 否
rollback_reason: 无

pre_flight_checks:
  backup_taken: 是（01:45 UTC 全量备份）
  staging_validated: 是（冒烟测试全部通过）
  monitoring_healthy: 是（所有告警绿灯）

health_check:
  /health: 200 ({"status":"healthy"})
  /docs: 200
  login: 200 (返回 access token)

verification_results:
  core_apis: 通过（策略、回测接口已验证）
  e2e_flow: 通过（登录→创建策略→查询→删除）
  performance_metrics:
    p95_latency: "420ms"
    error_rate: "0.03%"

incident_log:
  - 时间: 02:15 UTC
    问题: /health 返回 401（全局认证依赖导致）
    处理: 确认代码修复后，健康检查端点可公开访问
  - 时间: 02:30 UTC
    问题: 初始数据库连接失败
    处理: 等待数据库初始化完成，重启 API 后连接成功

lessons_learned: 部署前必须确保 /health 豁免认证依赖；数据库初始化完成前应用不应启动（建议增加 wait 脚本）。
```

详见附件日志与 Grafana 仪表盘。

---

## 11. 附录

### 11.1 参考文档

- `DEPLOYMENT_RUNBOOKS.md` - 完整手册（裸机、K8s、Traefik 配置）
- `MONITORING_CONFIG.md` - Prometheus/Loki 指标与仪表盘
- `SECURITY_CHECKLIST.md` - 42 项安全检查（部署前逐项勾选）
- `BACKUP_RECOVERY_PLAN.md` - 备份策略与恢复演练步骤
- `docs/architecture/` - 架构图与数据流

### 11.2 联系支持

- **Operator (Frank)**: 部署执行、监控、回滚
- **Coder**: 应用 Bug 修复、数据库迁移脚本
- **Tester**: 验证与性能测试
- **Writer (Mia)**: 文档归档与版本管理

**紧急情况**: 直接联系 Main 或 Daniel

---

**文档结束**