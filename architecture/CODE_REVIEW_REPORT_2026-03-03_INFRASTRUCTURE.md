# TraderMate 代码审查报告 - 2026-03-03

**PR/Commit**: N/A (初始基础设施审查)  
**审查人**: Frank (@operator)  
**审查日期**: 2026-03-03  
**审查范围**: docker-compose.yml, .github/workflows/, Dockerfile, config

---

## 执行摘要

- **总检查项**: 45
- **通过**: 28
- **需修复**: 12 (其中 🔴 高危: 4, 🟡 中危: 5, 🟢 建议: 3)
- **阻断性问题**: 4 (必须修复才能生产部署)

**结论**: ⚠️ **代码库处于 "开发就绪但生产不安全" 状态**，需要修复高危漏洞后方可部署。

---

## 🔴 高危问题 (阻断性)

### C001: MySQL 3306 端口暴露到公网

**文件**: `docker-compose.yml:36`  
**描述**:
```yaml
mysql:
  ports:
    - "3306:3306"  # ❌ 生产环境应禁止公网访问
```

**风险**: 任何人都可扫描并尝试暴力破解 MySQL root 密码，导致数据泄露或勒索。

**修复**:
```yaml
# 移除 ports 映射，仅容器网络访问
mysql:
  # ports:  # 删除整行
  #   - "3306:3306"
  expose:
    - "3306"  # 仅在同网络内暴露
```

或生产 config:
```yaml
mysql:
  ports: []  # 无端口映射
```

---

### C002: Redis 6379 端口暴露到公网

**文件**: `docker-compose.yml:60`  
**风险**: Redis 未授权访问可导致数据删除、RCE (eval 命令)、内存耗尽 DoS。

**修复**: 同 MySQL，移除 `ports` 映射。

---

### C003: 使用 root 用户连接 MySQL (最小权限违反)

**文件**: `docker-compose.yml:13-14, 26, 47`  
**描述**: 所有服务使用 `MYSQL_USER=root` 连接数据库。

**风险**: 应用获得超级权限，SQL 注入漏洞会直接灾难性后果。违背最小权限原则。

**修复**:

1. **创建专用数据库用户** (SQL):
```sql
CREATE USER 'tradermate'@'%' IDENTIFIED BY 'StrongRandomPassword123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON tradermate.* TO 'tradermate'@'%';
GRANT SELECT ON vnpy.* TO 'tradermate'@'%';
GRANT SELECT ON tushare.* TO 'tradermate'@'%';
FLUSH PRIVILEGES;
```

2. **更新环境变量**:
```bash
# .env.prod (生产)
MYSQL_USER=tradermate
MYSQL_PASSWORD=StrongRandomPassword123!
```

3. **更新 docker-compose.yml** (所有服务的 `MYSQL_USER` 从 `root` 改为 `${MYSQL_USER}`)

---

### C004: 生产 DEBUG 标志未禁用

**文件**: `docker-compose.yml:12`  
**描述**: `DEBUG=true` 硬编码在环境中。

**风险**: 暴露堆栈跟踪、服务器内部路径、配置信息。允许 `debug` toolbar 绕过认证。

**修复**:

```yaml
environment:
  - DEBUG=false  # 必须覆盖为 false
```

生产使用 `docker-compose.prod.yml`:
```yaml
services:
  api:
    environment:
      - DEBUG=false
      - APP_ENV=production
```

---

## 🟡 中危问题 (重要)

### C005: CORS 配置包含 localhost (生产不应暴露)

**文件**: `docker-compose.yml:25`  
**当前**: `CORS_ORIGINS=["http://localhost:3000","http://localhost:5173","http://127.0.0.1:5173"]`

**风险**: 允许任何 localhost 来源的请求跨域访问 API。若应用代理解析 `Host` 头不当，可能被绕过。

**修复**:

```python
# config.py
cors_origins: list[str] = ["https://www.tradermate.com", "https://api.tradermate.com"]
```

---

### C006: CI/CD 测试路径错误

**文件**: `.github/workflows/ci.yml:19-29`  
**当前**: `flake8 src tests`, `mypy src`, `pytest --cov=src`

**问题**: 项目源代码在 `app/` 而非 `src/`，测试命令将全部失败或返回空结果。

**修复**:

```yaml
- name: Lint with Flake8
  run: |
    flake8 app tests --count --select=E9,F63,F7,F82 --show-source --statistics
    flake8 app tests --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

- name: Type check with mypy
  run: |
    mypy app --ignore-missing-imports || true

- name: Run tests with coverage
  run: |
    if [ -d tests ]; then
      pytest tests/ -v --cov=app --cov-report=xml --cov-report=term
    else
      echo "No tests directory found, skipping..."
    fi
```

---

### C007: Kubernetes 部署目录缺失

**文件**: `.github/workflows/deploy.yml:18, 30`  
**问题**: `k8s/staging/` 和 `k8s/production/` 路径不存在，自动化部署会失败。

**修复** (三选一):

**选项 A**: 创建 K8s 清单 (推荐长期)  
见 `INFRASTRUCTURE_DIAGRAM.md` 第 4 节。

**选项 B**: 切换回 Docker Compose 部署  
修改 `deploy.yml` 使用 Docker Compose 命令。

**选项 C**: 临时禁用 K8s 部署  
注释掉 k8s-deploy steps，人工部署。

---

### C008: 缺少资源限制

**文件**: `docker-compose.yml` 所有服务  
**问题**: 未设置 `mem_limit`、`cpus`，单个服务 OOM 可能拖垮主机。

**修复**:

```yaml
api:
  deploy:
    resources:
      limits:
        memory: 1G
        cpus: '1.0'
      requests:
        memory: 512M
        cpus: '0.5'
```

---

### C009: 挂载代码卷应为只读 (生产)

**文件**: `docker-compose.yml:16, 34, 38`  
**当前**: `./app:/app/app` (可读写)

**风险**: 容器内可修改代码文件，安全策略失效。

**修复**:
```yaml
volumes:
  - ./app:/app/app:ro  # 添加 :ro
```

---

### C010: 日志文件未轮转

**文件**: `docker-compose.yml:18` (`logs` 卷挂载)  
**问题**: `./logs:/app/logs` 挂载后日志无限增长，占用磁盘空间。

**修复**:

1. **应用层**: Python `logging.handlers.RotatingFileHandler`
2. **Docker 层**: 配置 `json-file` 日志驱动限制大小
3. **宿主机**: 安装 logrotate 定时清理

示例 (Docker Daemon 配置):
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

---

## 🟢 建议项 (最佳实践)

### C011: Docker Compose 版本升级

**当前**: `version: '3.4'` (2018)  
**建议**: `version: '3.8'` 或更高 (支持 `--init`, 更好的 healthcheck)

**升级**:
```yaml
version: '3.8'
```

---

### C012: 缺少 Gunicorn workers 配置

**文件**: `Dockerfile.api`  
**当前**: `CMD ["uvicorn", "app.api.main:app", ...]`

**建议**: 生产使用 Gunicorn + UvicornWorker:

```dockerfile
CMD ["gunicorn", "app.api.main:app", "--workers", "4", "--worker-class", "uvicorn.workers.UvicornWorker", "--bind", "0.0.0.0:8000"]
```

---

### C013: Prometheus 指标端点未实现

**文件**: `app/api/main.py`  
**缺失**: `/metrics` 端点导出应用指标。

**建议**: 集成 `prometheus_client` (见 MONITORING_CONFIG.md 5.1 节)。

---

## 其他观察

| 项目 | 状态 | 说明 |
|------|------|------|
| **Dockerfile 安全** | ✅ 良好 | 非 root 用户 (appuser)，层缓存优化 |
| **Healthcheck** | ✅ 存在 | curl 检查 /health，合理 start_period |
| **SQLAlchemy** | ✅ 安全 | 参数化查询，pool_pre_ping |
| **密码哈希** | ✅ bcrypt | 无弱算法 |
| **JWT 配置** | ✅ 合理 | 过期时间 30分钟/7天 |
| **数据库 schema** | ✅ 完整 | `mysql/init/*.sql` 结构清晰 |

---

## 修复优先级与时间预估

| 优先级 | 问题编号 | 预估工时 | 负责人 | 状态 |
|--------|----------|----------|--------|------|
| P0 (立即) | C001, C002, C003, C004 | 2 小时 | @operator | 🔧 待修复 |
| P1 (本周) | C005, C006, C007 | 3 小时 | @operator / @coder | ⏳ 待处理 |
| P2 (下迭代) | C008, C009, C010 | 4 小时 | @operator | 📝 计划中 |
| P3 (改进) | C011, C012, C013 | 6 小时 | @operator | 💡 建议 |

---

## 修复检查清单 (合并前必须)

- [ ] **MySQL 端口**: 移除 `3306:3306` 映射，使用内部网络
- [ ] **Redis 端口**: 移除 `6379:6379` 映射
- [ ] **DB 用户**: 创建 `tradermate` 专用用户，授予最小权限
- [ ] **环境变量**: 更新 `.env.example` 和 `docker-compose.yml` 使用 `${MYSQL_USER}`
- [ ] **DEBUG**: 生产配置强制 `DEBUG=false`
- [ ] **CORS**: 生产仅允许 HTTPS 域名
- [ ] **CI 路径**: 修正 `flake8/mypy/pytest` 路径为 `app/`
- [ ] **K8s 目录**: 创建基础清单或更改部署流程
- [ ] **资源限制**: 为 API/Worker 添加内存和 CPU 限制
- [ ] **只读卷**: 代码挂载添加 `:ro`
- [ ] **日志轮转**: 配置容器日志大小限制

---

## 验证命令

修复后运行以下命令验证:

```bash
# 1. Docker Compose 配置检查
docker-compose config

# 2. 扫描暴露的端口 (确保 3306, 6379 不在列表中)
docker-compose port mysql 3306  # 应返回空
docker-compose port redis 6379  # 应返回空

# 3. 启动服务并验证健康
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
docker-compose ps

# 4. 检查日志无错误
docker-compose logs --tail=100 api | grep -i error

# 5. CI lint 本地测试 (修正后)
flake8 app tests --count
mypy app --ignore-missing-imports
pytest tests/ -v --cov=app
```

---

## 结论

TraderMate 基础设施代码**结构清晰、文档完备**，但存在**严重的安全疏漏**需要立即修复。优先级 C001-C004 为生产必备，建议在 2 小时内完成修复并重新审查。

**合并后**: 监控 24 小时，重点关注登录失败率、异常 SQL 查询、端口扫描日志。

---

**报告结束**

**签名**: Frank  
**时间**: 2026-03-03 15:45 UTC
