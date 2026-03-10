# Staging 环境部署备用方案
**触发条件**: 10.0.0.240 Docker 安装失败或遇到不可逾越的障碍  
**时间**: 2026-03-10 00:40 UTC  
**决策者**: Main PM (with Daniel approval)

---

## 方案 A: Docker 主机安装失败 → 切换到备选服务器

### 备选服务器清单
1. **10.0.0.73** (原 MySQL 生产服务器)
   - ✅ Docker 已安装 (由 O-002 部署)
   - ✅ 可直接部署 Staging
   - ⚠️ 与生产环境共享资源，需隔离
   - 端口: 使用不同映射 (e.g., 8001:8000)

2. **operator 本地环境** (如果恢复)
   - 使用 `localhost` Docker
   - 快速但需 operator 配合

### 切换步骤
```bash
# 1. 选择备选主机 (优先 10.0.0.73)
export STAGING_HOST=10.0.0.73
export STAGING_PORT=8001  # 如果 8000 被占

# 2. SSH 到备选主机
ssh ubuntu@10.0.0.73

# 3. 克隆代码 (如果不存在)
cd ~
git clone https://github.com/gitdshi/tradermate.git
cd tradermate
git checkout main
git pull origin main

# 4. 创建 .env.staging (端口调整)
# 修改 .env.staging 中的端口映射或使用环境变量覆盖

# 5. 启动
docker compose -f docker-compose.yml -f docker-compose.staging.yml up -d

# 6. 验证
curl http://localhost:8001/health
```

---

## 方案 B: 完全无 Docker → 手动直运部署

如果所有 Docker 主机都不可用，使用 **local Python 部署** (最简环境):

### 步骤
```bash
# 1. 在 10.0.0.240 (或任意服务器) 准备 Python 环境
sudo apt update
sudo apt install -y python3.12 python3.12-venv python3-pip mysql-client redis-tools

# 2. 克隆代码
cd ~
git clone https://github.com/gitdshi/tradermate.git
cd tradermate

# 3. 创建虚拟环境
python3.12 -m venv .venv
source .venv/bin/activate

# 4. 安装依赖
pip install -r requirements.txt

# 5. 配置 .env (staging 配置)
cp .env.example .env
# 手动编辑 .env 设置数据库连接和 TUSHARE_TOKEN

# 6. 启动 MySQL + Redis (假设外部有)
# 10.0.0.73 MySQL 和 Redis 已在运行

# 7. 运行 init_market_data.py (初始化数据)
python scripts/init_market_data.py --env staging

# 8. 启动 API 服务 (使用 uvicorn)
uvicorn app.api.main:app --host 0.0.0.0 --port 8000 --reload &
# 或使用 gunicorn: gunicorn app.api.main:app -w 4 -k uvicorn.workers.UvicornWorker -b :8000

# 9. 验证
curl http://localhost:8000/health
curl http://localhost:8000/metrics
```

**注意**: 此方案缺少 Docker Compose 的进程管理，需用 systemd/supervisor 做服务守护。

---

## 方案 C: 缩短验证范围 (如果时间不足)

如果 P1-DSYNC-TEST-001 无法完成完整 2-3h 验证，优先保证核心场景:

### 最小验证集 (1-1.5h)
1. ✅ **健康检查**: Staging 服务 runs
2. ✅ **Metrics 端点**: /metrics 可访问
3. ✅ **断点恢复**: single scenario (init + interrupt + resume)
4. ✅ **限流行为**: single rate limit hit + wait
5. ⚠️ **数据完整性**: spot check (可选)

**交付物**: 仍产出 `DATASYNC_STAGING_VALIDATION.md`，但标注为"部分验证，建议后续补全"

---

## 决策流程图

```
Docker 安装成功? 
  ├─ Yes → Staging up → Tester 验证 (正常流程)
  └─ No
      ├─ 备选主机可用 (10.0.0.73)? 
      │   ├─ Yes → 切换到备选 (方案A)
      │   └─ No → 直运部署 (方案B)
      └─ 时间不足? → 最小验证集 (方案C)
```

---

## 联系人责任

- **Coder**: 执行安装/切换方案，报告结果
- **Tester**: 准备验证用例，环境就绪后立即执行
- **Main**: 决策方案切换，升级 Daniel

---

**更新**: 实际使用的方案将在群内宣布。
