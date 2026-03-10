# DataSync Resume + Adaptive Rate Limit Plan

## 概述

本方案实现 `datasync` 的两项最小改造：

1. `init_market_data.py` 支持断点续跑（phase + cursor级恢复）
2. Tushare 调用改为“先调用，失败后按错误信息动态等待”，无等待信息时指数退避

目标是在不大改现有结构的前提下，提升长任务可恢复性与限流鲁棒性。

## 改造范围

### 1) init 断点续跑

- 新增持久化表：`tradermate.init_progress`
- 字段：`phase`、`cursor_ts_code`、`cursor_date`、`status`、`updated_at`、`error`
- 启动逻辑：
  - 默认开启 `--resume`
  - 可通过 `--reset-progress` 清空进度
  - 启动后优先读取 `init_progress`，按 phase 继续
- 落点策略：
  - `stock_basic`：每个 `list_status` 落一次checkpoint
  - `daily`：每个symbol落一次checkpoint
  - `adj_factor/dividend/top10_holders`：每个symbol落一次checkpoint

### 2) Tushare 自适应限流

- `call_pro` 新增 `parse_retry_after(error_msg)`
- 命中限流错误：
  - 若能解析等待时长：`sleep = parsed_wait + jitter`
  - 若无法解析：指数退避（`base * 2^(attempt-1)`）
- 保留轻量基础节流：
  - 仍执行 `_min_interval_for(api_name)` 的最小间隔控制
  - 动态等待作为错误后的主策略

## 文件变更清单

- `tradermate/scripts/init_market_data.py`
  - 新增 `init_progress` 建表、读写、重置
  - 增加 phase/cursor 级恢复逻辑
  - 新增参数：`--resume`（默认开启）、`--reset-progress`
- `tradermate/app/datasync/service/tushare_ingest.py`
  - 新增 `parse_retry_after` 与限流错误识别
  - `call_pro` 改为“动态等待优先 + 指数退避兜底”
  - 为批量函数增加可选 `start_after_ts_code` 与 `progress_cb`

## 实现说明（当前状态）

- 已完成代码级实现（首版）
- 当前未引入新外部依赖
- 兼容已有调用：新增参数均为可选参数，默认行为保持向后兼容

## 风险与回滚点

### 风险

- `init_progress` 为新表，首次运行需数据库可写
- 恢复点为“phase + 最近cursor”，极端情况下可能重复处理最后一个cursor附近数据（幂等写可接受）
- 限流错误文本可能变化，`parse_retry_after` 需后续按线上错误样本扩充

### 回滚点

- 回滚文件：
  - `tradermate/scripts/init_market_data.py`
  - `tradermate/app/datasync/service/tushare_ingest.py`
- 回滚策略：
  1. 恢复上述文件到改造前版本
  2. 保留 `init_progress` 表不影响旧逻辑（旧逻辑不读取该表）

## 本地验证建议

```bash
# 1) 语法校验
python -m py_compile tradermate/scripts/init_market_data.py
python -m py_compile tradermate/app/datasync/service/tushare_ingest.py

# 2) 仅跑初始化脚本参数帮助
PYTHONPATH=. python tradermate/scripts/init_market_data.py --help

# 3) 断点测试（建议测试环境）
# - 首次运行后中断，再次运行确认从checkpoint恢复
# - 测试 --reset-progress 行为
```

## 后续工作

- 将本方案拆分为2个PR：
  - PR1: 断点续跑
  - PR2: 自适应限流
- 在PR描述中附：变更说明、风险、回滚方式、验证记录
