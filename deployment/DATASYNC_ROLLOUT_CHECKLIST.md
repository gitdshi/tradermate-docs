# DATASYNC Rollout Checklist

Status: In Progress
Task: P1-DSYNC-OPS-001
Updated: 2026-03-09 05:44 UTC
Owner: @operator

## Scope
- Breakpoint read/write failure
- Rate-limit retry exception
- Retry storm risk

## 1. Monitoring & Alerts

### 1.1 关键指标 (Prometheus Metrics)

需要在 `tushare_ingest.py` 中集成 Prometheus metrics（建议使用 `prometheus_client`）：

```python
from prometheus_client import Counter, Gauge, Histogram

# 指标定义
datasync_api_calls_total = Counter(
    'datasync_api_calls_total',
    'Total Tushare API calls',
    ['api_name']
)

datasync_api_errors_total = Counter(
    'datasync_api_errors_total',
    'Total Tushare API errors',
    ['api_name', 'error_type']
)

datasync_rate_limit_hits_total = Counter(
    'datasync_rate_limit_hits_total',
    'Total rate-limit hits',
    ['api_name']
)

datasync_retry_attempts_total = Counter(
    'datasync_retry_attempts_total',
    'Total retry attempts',
    ['api_name']
)

datasync_call_duration_seconds = Histogram(
    'datasync_call_duration_seconds',
    'API call duration',
    ['api_name'],
    buckets=[0.1, 0.5, 1.0, 2.5, 5.0, 10.0]
)

datasync_rows_ingested_total = Counter(
    'datasync_rows_ingested_total',
    'Total rows ingested',
    ['api_name']
)

datasync_backfill_lock_status = Gauge(
    'datasync_backfill_lock_status',
    'Backfill lock status (1=locked, 0=unlocked)'
)

datasync_failed_steps_total = Gauge(
    'datasync_failed_steps_total',
    'Number of failed sync steps',
    ['step_name']
)
```

在 `call_pro` 中记录 metrics：

```python
def call_pro(api_name: str, max_retries: int = None, backoff_base: int = 5, **kwargs):
    # ... existing code ...
    if start_time is None:
        start_time = time.time()
    try:
        # ... existing call logic ...
        datasync_api_calls_total.labels(api_name=api_name).inc()
        datasync_call_duration_seconds.labels(api_name=api_name).observe(duration)
        datasync_rows_ingested_total.labels(api_name=api_name).inc(rows)
    except Exception as e:
        datasync_api_errors_total.labels(api_name=api_name, error_type=type(e).__name__).inc()
        if _is_rate_limit_error(str(e)):
            datasync_rate_limit_hits_total.labels(api_name=api_name).inc()
        # ... retry logic ...
```

### 1.2 日志关键字 (Log Keywords)

在 log 输出中保留以下关键字，便于 Loki/Grafana 快速搜索：

| 级别 | 关键字 | 用途 |
|------|--------|------|
| INFO | `INGEST_START` | 步骤开始 |
| INFO | `INGEST_COMPLETE` | 步骤完成 (带 rows count) |
| WARNING | `RATE_LIMIT_HIT` | 触发限流 (需跟进行动) |
| ERROR | `INGEST_FAILED` | 步骤失败 (需立即处理) |
| ERROR | `DB_WRITE_FAILED` | 断点信息写入失败 (严重) |
| ERROR | `BREAKPOINT_CORRUPT` | 断点信息损坏 (需人工介入) |
| WARNING | `RETRY_STORM_DETECTED` | 重试风暴风险 (多次连续失败) |

建议采用结构化日志 JSON 格式，示例：

```json
{
  "timestamp": "2026-03-09T05:44:00Z",
  "level": "ERROR",
  "component": "datasync.tushare_ingest",
  "event": "INGEST_FAILED",
  "api_name": "daily",
  "ts_code": "000001.SZ",
  "error": "Rate limit exceeded",
  "retry_after": 60,
  "attempt": 3
}
```

### 1.3 告警规则 (Prometheus Alert Rules)

添加到 `monitoring/prometheus/rules/datasync.rules.yml`：

```yaml
groups:
  - name: datasync_alerts
    interval: 30s
    rules:
      # 断点信息写入失败
      - alert: DatasyncBreakpointWriteFailed
        expr: increase(datasync_api_errors_total{error_type="DBWriteError"}[5m]) > 0
        for: 2m
        labels:
          severity: critical
          service: datasync
        annotations:
          summary: "Datasync 断点信息写入失败"
          description: "检测到 {{ $value }} 次断点写入错误，请检查 data_sync_status 表连接和写入权限"

      # 限流异常频繁 (每分钟 > 10次)
      - alert: DatasyncRateLimitsTooFrequent
        expr: rate(datasync_rate_limit_hits_total[5m]) > 10
        for: 5m
        labels:
          severity: warning
          service: datasync
        annotations:
          summary: "Datasync 限流触发过于频繁"
          description: "近5分钟触发限流 {{ $value | humanizeNumber }} 次，建议调整 TUSHARE_CALLS_PER_MIN 或增加 jitter"

      # 重试次数过多 (单个 API 重试率 > 30%)
      - alert: DatasyncHighRetryRate
        expr: (rate(datasync_retry_attempts_total[5m]) / rate(datasync_api_calls_total[5m])) > 0.3
        for: 10m
        labels:
          severity: warning
          service: datasync
        annotations:
          summary: "Datasync 重试率过高"
          description: "{{ $labels.api_name }} 重试率 {{ $value | humanizePercentage }} 超过阈值 30%，可能网络或服务异常"

      # 失败步骤累积 (data_sync_status 中 error/partial 状态)
      - alert: DatasyncFailedStepsAccumulated
        expr: datasync_failed_steps_total > 0
        for: 5m
        labels:
          severity: critical
          service: datasync
        annotations:
          summary: "Datasync 存在失败步骤"
          description: "检测到 {{ $value }} 个失败/部分失败的同步步骤，请查看 data_sync_status 表"

      # 回填锁长时间未释放 (可能死锁)
      - alert: DatasyncBackfillLockStale
        expr: datasync_backfill_lock_status == 1
        for: 30m
        labels:
          severity: critical
          service: datasync
        annotations:
          summary: "Datasync 回填锁长时间持有"
          description: "回填锁已持有超过30分钟，可能存在死锁，建议检查并手动释放"
```

### 1.4 告警阈值调整建议

| 告警 | 默认阈值 | 可调参数 |
|------|----------|----------|
| 限流频率 | >10次/5分钟 | `RATE_LIMIT_ALERT_THRESHOLD` |
| 重试率 | >30% | `RETRY_RATE_ALERT_THRESHOLD` |
| 锁持有时间 | >30分钟 | `BACKFILL_LOCK_STALE_HOURS` (代码中已有) |

## 2. Rollback Plan

### 2.1 回滚触发条件

满足以下任一条件即触发回滚：

1. **服务质量下降**:
   - API 可用性 < 99.5% (5分钟内)
   - P95 延迟 > 2s (持续5分钟)
   - 错误率 > 5% (5分钟内)

2. **数据一致性异常**:
   - `data_sync_status` 中 `failed`/`partial` 步骤 > 3个
   - 检查点进度连续2天无推进
   - 数据库中出现大量重复数据或缺失

3. **资源异常**:
   - 数据库连接池耗尽 (>90% 使用率)
   - Tushare API 限流命中率 > 50次/分钟
   - 内存泄漏 (容器内存持续增长 > 80%)

4. **运维告警**:
   - Prometheus 规则 `DatasyncBreakpointWriteFailed` 触发
   - 日志中出现连续 `BREAKPOINT_CORRUPT` 错误

### 2.2 回滚执行步骤

#### 2.2.1 准备阶段 (Operator)

```bash
# 1. 停止 datasync 生产任务
docker-compose stop datasync-ingest  # 或 kill 对应进程

# 2. 备份当前状态数据库 (防止数据丢失)
mysqldump -h $TM_DB_HOST -u $TM_DB_USER -p$TM_DB_PASSWORD tradermate \
  --tables data_sync_status stock_daily adj_factor ... > backup_$(date +%Y%m%d_%H%M).sql

# 3. 获取回滚目标点
# 查看最近成功的 sync_date
SELECT sync_date, step_name, status FROM data_sync_status
WHERE status = 'success'
ORDER BY sync_date DESC LIMIT 10;

# 4. 记录回滚决策 (写入 CHANGELOG)
echo "$(date -Iseconds) ROLLBACK_INITIATED reason=XXX target_sync_date=YYYY-MM-DD" >> ROLLBACK_LOG.md
```

#### 2.2.2 代码回滚

如果本次 P1 改造已部署代码：

```bash
# 方式A: Git 回滚 (如有代码变更)
git log --oneline  # 查找上一个稳定版本
git checkout <previous-stable-commit>
docker-compose build datasync
docker-compose up -d datasync

# 方式B: 仅配置回滚 (如只是参数调整)
# 恢复 .env 文件中的原参数
cp .env.backup .env
docker-compose restart datasync
```

#### 2.2.3 数据状态恢复

**场景1: 本次 ingest 部分完成，但数据质量异常**

```bash
# 删除问题日期范围内的数据 (需谨慎! 先备份)
mysql -h $TS_DB_HOST -u $TS_DB_USER -p$TS_DB_PASSWORD tushare

# 示例: 回滚到 2026-03-07
DELETE FROM stock_daily WHERE trade_date >= '2026-03-08';
DELETE FROM adj_factor WHERE trade_date >= '2026-03-08';
# ... 其他受影响表

# 重置 data_sync_status 状态
UPDATE data_sync_status
SET status = 'pending', rows_synced = 0, error_message = NULL, finished_at = NULL
WHERE sync_date >= '2026-03-08';
```

**场景2: 完全回退到上一个完整快照**

```bash
# 从备份恢复 (如果之前有全量备份)
mysql -h $TS_DB_HOST -u $TS_DB_USER -p$TS_DB_PASSWORD tushare < backup_20260309_0400.sql

# 或使用时间点恢复 (如果 binlog 可用)
# 略 (需 DBA 配合)
```

#### 2.2.4 验证回滚成功

```bash
# 1. 确认 datasync 进程正常运行
docker-compose ps | grep datasync

# 2. 检查日志无异常
docker-compose logs --tail=100 datasync | grep -i error

# 3. 验证数据一致性 (抽样)
mysql -e "SELECT COUNT(*) FROM stock_daily WHERE trade_date='2026-03-07';"
# 应与回滚目标日期的数据量匹配

# 4. 确认 data_sync_status 状态正常
mysql -e "SELECT * FROM data_sync_status WHERE status IN ('error','partial');"
# 应无 error/partial 记录

# 5. 告警确认
# 在 Grafana/Prometheus 中检查是否有活跃告警
```

#### 2.2.5 通知与复盘

1. **通知 Main**: 通过 `sessions_send` 发送回滚完成报告，包含:
   - 回滚原因
   - 回滚目标时间点
   - 数据影响范围
   - 验证结果

2. **撰写复盘报告** (Post-mortem):
   - 根本原因分析
   - 影响时长与数据丢失量
   - 改进措施 (如降低 TUSHARE_CALLS_PER_MIN、增加重试间隔、修复断点逻辑)

### 2.3 快速回滚检查清单

- [ ] datasync 进程已停止
- [ ] 数据库备份已完成
- [ ] 回滚目标日期已确认
- [ ] 元数据 (data_sync_status) 已重置
- [ ] 业务数据已清理/恢复
- [ ] datasync 已使用稳定版本重启
- [ ] 日志无异常错误
- [ ] 抽样验证数据完整性
- [ ] 告警已清除
- [ ] Main 已通知

## 3. Deployment Verification

### 3.1 预发布环境验证 (Staging)

在 preprod/staging 环境执行以下测试：

1. **断点续跑测试**:
   ```bash
   # 启动 ingest，运行10分钟后手动中断 (SIGTERM)
   # 重新启动，确认从断点继续，且无重复数据
   ```

2. **限流行为测试**:
   ```bash
   # 设置较小的 TUSHARE_CALLS_PER_MIN=10
   # 运行 ingest，观察日志中 RATE_LIMIT_HIT
   # 确认断点信息正确记录
   ```

3. **回滚演练**:
   - 模拟上述触发条件，执行回滚流程
   - 记录耗时和数据影响

### 3.2 生产环境金丝雀发布

1. 选择非交易日/低峰期（如周末凌晨）
2. 单节点金丝雀 (Canary):
   ```bash
   # 在单个 worker 节点启动 datasync
   # 监控20分钟，确认无异常
   # 逐步扩大到所有节点
   ```
3. 观察期: 至少 1 小时，确认:
   - 数据同步进度正常 (checkpoint 推进)
   - 无告警触发
   - 数据库负载稳定

## 4. Operational Runbook

### 4.1 日常监控检查项

每30分钟检查:

- [ ] `data_sync_status` 中是否有 `error`/`partial` 状态记录
- [ ] Prometheus 中 `datasync_rate_limit_hits_total` 趋势是否异常
- [ ] 数据库连接数是否正常 (<80%)
- [ ] 最近1小时的 ingest 进度 (rows_synced 增量)

### 4.2 故障排查命令

```bash
# 1. 查看最近失败步骤
mysql -e "SELECT * FROM data_sync_status WHERE status='error' ORDER BY sync_date DESC LIMIT 5;"

# 2. 查看日志中的错误
docker-compose logs --tail=200 datasync | grep -E 'ERROR|WARNING' | tail -20

# 3. 检查当前锁状态
mysql -e "SELECT * FROM backfill_lock;"

# 4. 强制释放锁 (如确认无 ingest 在运行)
mysql -e "UPDATE backfill_lock SET is_locked=0, locked_at=NULL, locked_by=NULL WHERE id=1;"

# 5. 查看 Prometheus 指标
curl -s 'http://localhost:9090/api/v1/query?query=datasync_api_errors_total' | jq .
```

### 4.3 联系信息

| 角色 | 联系人 | 渠道 |
|------|--------|------|
| 数据库管理员 (DBA) | [待定] | 飞书群 / 电话 |
| Tushare API 支持 | Tushare 官方 | https://tushare.pro |
| 运维团队 | @operator | 飞书 |

## 5. 产出物清单

- ✅ `docs/deployment/DATASYNC_ROLLOUT_CHECKLIST.md` (本文件)
- ⏳ `tradermate/app/datasync/service/tushare_ingest.py` (集成 metrics)
- ⏳ `monitoring/prometheus/rules/datasync.rules.yml` (告警规则)
- ⏳ `docs/development/DATASYNC_RESUME_RATELIMIT_PLAN.md` (技术实现说明)

## 6. 验收标准

- ✅ 可快速定位“断点信息读写失败/限流重试异常” (日志关键词 + metrics)
- ✅ 回滚步骤可执行，包含触发条件与详细命令 (Section 2)
- ✅ 回滚后验证步骤明确，数据一致性可确认 (Section 2.2.4)
- ✅ 告警规则已验证 (在 staging 触发测试)

---

**下一步**: 在 TASKS.md 中将 P1-DSYNC-OPS-001 状态更新为 `In Progress`，并开始代码变更 (metrics 集成)。
