# TraderMate 代码审查清单

**版本**: 1.0  
**最后更新**: 2026-03-03  
**适用对象**: Operator, Coder (互审)  
**审查范围**: 基础设施代码 (Dockerfile, docker-compose.yml, CI/CD, 部署脚本)

---

## 目录

1. [审查原则](#审查原则)
2. [Docker 相关审查](#docker-相关审查)
3. [CI/CD 审查](#cicd-审查)
4. [配置管理审查](#配置管理审查)
5. [安全审查](#安全审查)
6. [性能与资源审查](#性能与资源审查)
7. [可靠性与运维审查](#可靠性与运维审查)
8. [合规性审查](#合规性审查)

---

## 审查原则

✅ **绿灯标准** (可直接合并):
- 所有检查项通过或次要问题已修复
- 无安全漏洞 (中/高危)
- 无破坏性变更 (DB schema breaking change)
- 有回滚方案

⚠️ **黄灯标准** (需修改后合并):
- 1-2 个低级问题 (样式、文档缺失)
- 3+ 个低优先级改进建议
- 缺少测试覆盖率数据

🔴 **红灯标准** (拒绝合并):
- 任何安全漏洞
- 硬编码密钥/密码
- 不遵循基础设施即代码 (IaC) 原则
- 无健康检查
- 破坏现有部署

---

## Docker 相关审查

### 2.1 Dockerfile 审查清单

| 检查项 | 标准 | 当前状态 | 问题示例 |
|--------|------|----------|----------|
| **基础镜像** | 使用官方镜像，明确版本标签 (非 `latest`) | ⚠️ `python:3.11` (OK) | ❌ `python:latest` |
| **镜像大小** | < 500MB (单阶段) / < 1GB (多阶段) | 检查: `docker images` | 过大需要拆分 |
| **多阶段构建** | 生产镜像不包含构建工具 (gcc, make) | ✅ `Dockerfile.api` | ❌ 含 build-essential |
| **非 root 用户** | 使用 `USER` 指令创建专用用户 | ✅ Dockerfile.api (appuser) | ❌ root 运行 |
| **WORKDIR** | 设置明确工作目录 | ✅ `/app` | 未设置 |
| **COPY 优化** | 先 COPY requirements 利用缓存层 | ✅ `Dockerfile.api` | 全量 COPY 后才 pip install |
| **.dockerignore** | 排除 .git, venv, __pycache__, logs | ✅ 存在 | 缺失关键文件 |
| **HEALTHCHECK** | 定义健康检查指令 | ✅ `curl -f http://localhost:8000/health` | 缺失 |
| **CMD/ENTRYPOINT** | 使用 exec 格式 (JSON array) | ✅ `["uvicorn", ...]` | ❌ shell 格式 |
| **暴露端口** | 仅暴露必要端口 | ✅ `EXPOSE 8000` | 暴露 22/23 等 |

### 2.2 docker-compose.yml 审查清单

| 检查项 | 标准 | 当前状态 | 说明 |
|--------|------|----------|------|
| **版本** | ≥ '3.8' 以使用新特性 | ✅ `3.4` | 可升级 |
| **restart 策略** | 生产环境 `restart: always` | ✅ `unless-stopped` (OK) | |
| **depends_on** | 使用 `condition: service_healthy` | ✅ MySQL/Redis | |
| **环境变量** | 使用 `${VAR}` 从 .env 注入 | ✅ | 确保 .env 不在 Git |
| **端口暴露** | 仅暴露必要端口，DB/Redis 不暴露 | ⚠️ MySQL 3306, Redis 6379 暴露 | **安全隐患** |
| **资源限制** | 设置 `mem_limit`, `cpus` | ❌ 缺失 | 需添加 |
| **只读卷** | 代码卷应为 `:ro` (生产) | ⚠️ `./app:/app/app` (可写) | 修改 |
| **网络** | 使用自定义 bridge 网络 | ✅ `tradermate_network` | |
| **健康检查超时** | 超时时间适应服务启动时间 | ✅ 40s start_period | 合理 |
| **日志驱动** | 配置 json-file 或 journald | ✅ 默认 | 考虑添加大小限制 |

**修复建议**:

```yaml
# 生产 docker-compose.prod.yml 应:
api:
  ports:
    - "8000:8000"  # 通过 Ingress 暴露，可移除
  deploy:
    resources:
      limits:
        memory: 1G
        cpus: '1.0'
  volumes:
    - ./app:/app/app:ro  # 只读
```

### 2.3 镜像构建审查

- [ ] **构建缓存**: Docker BuildKit 使用缓存层
- [ ] **镜像扫描**: CI 中集成 Trivy/Anchore
- [ ] **多架构**: 支持 arm64 (如 Apple Silicon) 和 amd64
- [ ] **镜像签名**: 使用 Cosign 或 Docker Content Trust

---

## CI/CD 审查

### 3.1 GitHub Actions 审查

| 检查项 | 标准 | 当前状态 | 说明 |
|--------|------|----------|------|
| **触发条件** | 合理的分支过滤 (main, develop, release) | ✅ | |
| **缓存** | 使用 actions/cache 加速依赖安装 | ⚠️ 有 cache 但可优化 | |
| **矩阵测试** | 多 Python 版本测试 | ❌ 仅 3.11 | 建议加 3.12 |
| **测试覆盖率** | 要求 ≥ 80% | ❌ 无 `coverage: true` | 需添加 |
| **安全扫描** | Bandit + Safety | ✅ 存在 | |
| **制品上传** | 构建镜像 push 到 Registry | ✅ | |
| ** secrets 管理** | 不硬编码密码，使用 `secrets.` | ✅ | |
| **部署流程** | 手动触发或仅 main 分支自动 | ⚠️ develop 自动部署 staging | OK |
| **回滚步骤** | 部署失败自动回滚或人工流程 | ⚠️ 无自动回滚 | 需添加 |

### 3.2 当前 ci.yml 发现的问题

1. **测试路径错误**:
   - 使用 `src` 但项目实际代码在 `app/`
   ```yaml
   flake8 src tests  # ❌ 应为: flake8 app tests
   mypy src          # ❌ 应为: mypy app
   pytest tests/ -v --cov=src  # ❌ --cov=app
   ```

2. **依赖安装重复**:
   - 建议使用 `requirements.txt` 统一管理
   - 开发依赖分离到 `requirements-dev.txt`

3. **No Docker Build**:
   - CI 仅 build 但未 push 镜像 (除非 release)
   - 部署依赖 `DOCKER_USERNAME`/`DOCKER_PASSWORD` secrets

**修改建议**:

```yaml
# 修正路径
- name: Lint with Flake8
  run: |
    flake8 app tests --count --select=E9,F63,F7,F82 --show-source --statistics
```

### 3.3 部署流程 (deploy.yml) 审查

**关键问题**:
- ⚠️ **K8s 目录缺失**: 引用的 `k8s/staging/` 和 `k8s/production/` 不存在
- ⚠️ **Docker Hub Secrets**: 需要配置 `DOCKER_USERNAME` 和 `DOCKER_PASSWORD`
- ⚠️ **无回滚机制**: 部署失败不自动回滚，需手动

**建议**:
1. 创建 `k8s/` 基础清单 (见 INFRASTRUCTURE_DIAGRAM.md)
2. 添加 pre-deployment smoke tests
3. 配置 rollback on failure (using `kubectl rollout undo`)

---

## 配置管理审查

### 4.1 环境变量审查

| 文件 | 检查项 | 状态 | 备注 |
|------|--------|------|------|
| `.env.example` | 所有必需变量已列出 | ✅ 16 项 | |
| `.env` (生产) | 不在 Git，权限 600 | ❌ 未检查 | 需确认 |
| `config.py` | secrets 从环境读取 (非硬编码) | ✅ | |
| `docker-compose.yml` | 使用 `${VAR}` (非默认值) | ✅ | |

**缺失变量**:
- `MYSQL_DB_DRIVER` 未在 .env.example 中但代码使用 `os.getenv('MYSQL_DB_DRIVER', 'mysql')` (有默认值 OK)
- `VN_DATAFEED_NAME` 可选，无需默认

### 4.2 配置覆盖审查

- [ ] **开发/生产隔离**: `docker-compose.override.yml` 或 `-f docker-compose.prod.yml`
- [ ] **配置优先级**: `.env.prod` 覆盖 `.env`
- [ ] **密钥轮换**: 无需重启服务 (环境变量热重载)

---

## 安全审查

### 5.1 代码层安全

| 检查项 | 标准 | 文件位置 | 说明 |
|--------|------|----------|------|
| **SQL 注入** | 全部使用参数化查询 | `app/domains/*/dao/*.py` | SQLAlchemy OK |
| **密码哈希** | 使用 bcrypt/argon2 (非 MD5/SHA1) | `app/api/services/auth_service.py` | ✅ bcrypt |
| **JWT 验证** | 使用 `python-jose` 验证签名 | `app/api/services/auth_service.py` | ✅ |
| **CORS** | 生产仅允许 HTTPS 域名 | `app/api/main.py` | ⚠️ 包含 `localhost` |
| **路径遍历** | 文件操作验证路径 | N/A | 暂无 |
| **SSRF 防护** | URL 参数白名单 | `marketDataAPI` 接受 `exchange` 参数 | ⚠️ 无严格过滤 |
| **速率限制** | 登录接口限流 (5 次/10 分钟) | 未发现 | ❌ 需添加 |
| **错误信息** | 不暴露内部路径/堆栈 | FastAPI 默认生产隐藏 | ✅ (需设置 DEBUG=false) |

**CORS 生产配置建议** (config.py):

```python
cors_origins: list[str] = [
    "https://www.tradermate.com",
    "https://api.tradermate.com"
]
```

### 5.2 基础设施安全

| 检查项 | 当前配置 | 风险等级 | 修复建议 |
|--------|----------|----------|----------|
| MySQL 端口暴露 | 3306 暴露到主机 | 🔴 高危 | 移除 `ports` 或仅本地绑定 |
| Redis 端口暴露 | 6379 暴露到主机 | 🔴 高危 | 移除 `ports` |
| MySQL root 用户 | 允许远程 root 登录 | 🔴 高危 | 创建专用应用用户 |
| 默认 admin 密码 | .env.example 有默认 | 🔴 高危 | 强制修改首次登录 |
| /docs 端点公开 | Swagger UI 无需认证 | 🟡 中危 | 生产环境 Basic Auth 或 IP 白名单 |
| /metrics 端点公开 | metrics 未认证 | 🟡 中危 | 内部网络访问或 Basic Auth |

**MySQL 专用用户创建**:

```sql
CREATE USER 'tradermate'@'%' IDENTIFIED BY 'strong-password';
GRANT SELECT, INSERT, UPDATE, DELETE ON tradermate.* TO 'tradermate'@'%';
GRANT SELECT ON vnpy.* TO 'tradermate'@'%';
GRANT SELECT ON tushare.* TO 'tradermate'@'%';
FLUSH PRIVILEGES;
```

修改 `.env`:

```bash
MYSQL_USER=tradermate
MYSQL_PASSWORD=strong-password
```

---

## 性能与资源审查

### 6.1 数据库性能

- [ ] **索引**: `strategies.user_id`, `backtest_history.job_id` 已建立 (查看 SQL)
- [ ] **查询优化**: N+1 问题检查 (SQLAlchemy lazy loading)
- [ ] **连接池**: 使用 `pool_pre_ping=True` ✅
- [ ] **慢查询日志**: 是否启用 `slow_query_log=1`

### 6.2 API 性能

- [ ] **分页**: 列表接口使用 `limit/offset` (检查)
- [ ] **缓存**: 常用数据 (symbols 列表) 是否缓存 (Redis)
- [ ] **异步**: I/O 密集型操作使用 async (FastAPI ✅)
- [ ] **Gunicorn workers**: 建议 `2 * CPU + 1` (Dockerfile 未设置)

**生产启动建议** (使用 Gunicorn):

```dockerfile
# Dockerfile.api 改为
CMD ["gunicorn", "app.api.main:app", "--workers", "4", "--worker-class", "uvicorn.workers.UvicornWorker", "--bind", "0.0.0.0:8000"]
```

### 6.3 静态资源

- [ ] **前端构建**: 生产环境 `npm run build` 生成静态文件 (Nginx 服务)
- [ ] **压缩**: Nginx `gzip on;` + Brotli (可选)

---

## 可靠性与运维审查

### 7.1 日志审查

| 检查项 | 标准 | 当前状态 |
|--------|------|----------|
| **结构化日志** | JSON 格式便于解析 | ⚠️ 使用 `logging` 标准库，未 JSON |
| **日志级别** | 生产使用 `INFO`/`WARNING` | ✅ `LOG_LEVEL=INFO` |
| **日志轮转** | 大小或时间轮转 | ❌ Docker 默认无轮转 (需 log-rotate) |
| **集中收集** | 发送到 ELK/Loki | ❌ 暂无 | 

**建议**: 集成 `structlog` 输出 JSON，配合 Loki 收集。

### 7.2 健康检查

| 端点 | 检查内容 | 状态码 | 当前实现 |
|------|----------|--------|----------|
| `/health` | DB + Redis 连通性 | 200/503 | ✅ 实现 |
| `/ready` | 所有依赖就绪 | 200/503 | ❌ 缺失 |
| `/metrics` | Prometheus 指标 | 200 | ❌ 需集成 |

### 7.3 优雅关闭

- [ ] **SIGTERM 处理**: FastAPI 默认处理 ✅
- [ ] **连接等待**: 结束前完成进行中的请求 (uvicorn `--timeout-graceful-shutdown`)
- [ ] **队列 drain**: RQ worker 处理完当前任务再退出

---

## 合规性审查

### 8.1 开源许可

- [ ] **许可证检查**: `LICENSE` 文件存在 ✅
- [ ] **依赖合规**: 无 GPL-3.0 强制开源传染 (使用 `pip-licenses`)
  ```bash
  pip install pip-licenses
  pip-licenses --format=markdown
  ```

### 8.2 金融数据合规

- [ ] **数据源合规**: Tushare 数据使用许可 ✅
- [ ] **用户隐私**: 不存储交易密码 (仅 hash)
- [ ] **审计日志**: IP、时间、操作记录 (建议增加)

---

## 审查报告模板

```markdown
## 代码审查报告

**PR/Commit**: #123 - feat: add Docker Compose production config  
**审查人**: Frank (@operator)  
**审查日期**: 2026-03-03  

### 问题汇总

| 编号 | 文件 | 行号 | 问题描述 | 级别 | 状态 |
|------|------|------|----------|------|------|
| C001 | docker-compose.yml | 45 | MySQL 3306 端口暴露到公网 | 🔴 高危 | 待修复 |
| C002 | .env.example | 12 | 默认密码太弱 | 🔴 高危 | 待修复 |
| C003 | .github/workflows/ci.yml | 23 | 测试路径应为 `app/` | 🟡 中危 | 待修复 |
| C004 | docker-compose.prod.yml | - | 缺少资源限制 | 🟢 建议 | 建议采纳 |

### 结论

**当前状态**: ⚠️ 需修改后重新审查

**必须修复 (阻断)**:
1. 移除 MySQL/Redis 主机端口映射
2. 创建专用 DB 用户并修改 .env.example

**建议修复 (非阻断)**:
1. 修正 CI 测试路径
2. 添加资源限制

**合并后操作**:
- [ ] 验证部署到 staging 环境健康
- [ ] 监控 24 小时无异常

---
**签名**: Frank  
**时间**: 2026-03-03 15:30 UTC
```

---

**文档结束**
