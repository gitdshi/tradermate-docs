# TraderMate 监控告警配置指南

**版本**: 1.0  
**最后更新**: 2026-03-03

---

## 目录

1. [监控架构概览](#监控架构概览)
2. [Prometheus 配置](#prometheus-配置)
3. [Grafana 仪表板](#grafana-仪表板)
4. [应用到 SMTP/钉钉/企业微信](#告警通知配置)
5. [关键指标与告警规则](#关键指标与告警规则)
6. [日志聚合方案](#日志聚合方案)
7. [部署步骤](#部署步骤)

---

## 监控架构概览

推荐使用 **Prometheus + Grafana + Alertmanager** 组合：

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│   Traders   │────│  Prometheus  │────│  Alertmanager│
│   Mate App  │    │  (指标采集)  │    │  (告警路由)  │
└─────────────┘    └──────────────┘    └─────────────┘
                            │                     │
                            ▼                     ▼
                       ┌─────────┐         ┌─────────┐
                       │ Grafana │         │  邮件/钉│
                       │ (可视化)│         │ 钉/企微 │
                       └─────────┘         └─────────┘
```

### 组件版本建议

| 组件 | 版本 | 用途 |
|------|------|------|
| Prometheus | 2.50+ | 时序数据采集与存储 |
| Grafana | 10+ | 仪表板展示 |
| Alertmanager | 0.25+ | 告警分组、抑制、通知 |
| node_exporter | 1.7+ | 主机指标采集 |
| cAdvisor | 0.49+ | 容器指标采集 |
| mysqld_exporter | 0.15+ | MySQL 指标 |
| redis_exporter | 1.15+ | Redis 指标 |
| blackbox_exporter | 0.25+ | HTTP/TCP 探活 |

---

## Prometheus 配置

### 2.1 Docker Compose 集成

在 `docker-compose.prod.yml` 中添加：

```yaml
  prometheus:
    image: prom/prometheus:latest
    container_name: tradermate_prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus:/etc/prometheus:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'  # 允许热重载
    networks:
      - tradermate_network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.prometheus.rule=Host(`${MONITOR_HOST}`)"
      - "traefik.http.routers.prometheus.entrypoints=websecure"
      - "traefik.http.routers.prometheus.tls.certresolver=myresolver"
      - "traefik.http.middlewares.prometheus-auth.basicauth.usersfile=/etc/prometheus/.htpasswd"

  node-exporter:
    image: prom/node-exporter:latest
    container_name: tradermate_node_exporter
    restart: always
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - tradermate_network
    deploy:
      mode: global  # 每个节点一个实例

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: tradermate_cadvisor
    restart: always
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    devices:
      - /dev/kmsg
    networks:
      - tradermate_network

  mysqld-exporter:
    image: prom/mysqld-exporter:latest
    container_name: tradermate_mysqld_exporter
    restart: always
    environment:
      - DATA_SOURCE_NAME=root:${MYSQL_PASSWORD}@(mysql:3306)/
    networks:
      - tradermate_network

  redis-exporter:
    image: oliver006/redis_exporter:latest
    container_name: tradermate_redis_exporter
    restart: always
    environment:
      - REDIS_ADDR=redis://redis:6379
    networks:
      - tradermate_network
```

### 2.2 Prometheus 配置文件

创建 `monitoring/prometheus/prometheus.yml`：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 30s
  external_labels:
    cluster: 'tradermate-production'
    environment: 'production'

rule_files:
  - "/etc/prometheus/rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 'alertmanager:9093'

scrape_configs:
  # 自身指标
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # 主机指标
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    metrics_path: /metrics
    params:
      'collect[]': ['cpu', 'mem', 'disk', 'diskio', 'net', 'system']

  # 容器指标
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  # MySQL 指标
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysqld-exporter:9104']

  # Redis 指标
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  # API 应用指标 (需集成 prometheus_client)
  - job_name: 'tradermate-api'
    static_configs:
      - targets: ['api:8000']
    metrics_path: /metrics  # FastAPI 需暴露 /metrics

  # Traefik 指标
  - job_name: 'traefik'
    static_configs:
      - targets: ['traefik:8080']

  # 黑盒探活 (HTTP/HTTPS 端点)
  - job_name: 'blackbox-http'
    static_configs:
      - targets: ['blackbox-exporter:9115']
    metrics_path: /probe
    params:
      module: [http_2xx]
      target: ['https://api.tradermate.com/health']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 'blackbox-exporter:9115'

  # 前端静态资源探活
  - job_name: 'frontend'
    static_configs:
      - targets: ['blackbox-exporter:9115']
    metrics_path: /probe
    params:
      module: [http_2xx]
      target: ['https://www.tradermate.com/']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 'blackbox-exporter:9115'
```

### 2.3 告警规则文件

创建 `monitoring/prometheus/rules/application.rules.yml`：

```yaml
groups:
  - name: tradermate_api_alerts
    interval: 30s
    rules:
      # API 高延迟 (P95 > 500ms)
      - alert: APIHighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "API 响应延迟过高"
          description: "{{ $labels.endpoint }} P95 延迟为 {{ $value }}s (阈值 0.5s)"

      # API 错误率 > 5%
      - alert: APIHighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "API 错误率过高"
          description: "{{ $labels.endpoint }} 错误率为 {{ $value | humanizePercentage }} (阈值 5%)"

      # 健康检查失败
      - alert: APIHealthCheckFailed
        expr: up{job="tradermate-api"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "API 健康检查失败"
          description: "API 服务不可达已超过 1 分钟"

  - name: database_alerts
    interval: 30s
    rules:
      # MySQL 连接数过高
      - alert: MySQLTooManyConnections
        expr: mysql_global_status_threads_connected > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL 连接数过多"
          description: "当前连接数 {{ $value }} 超过阈值 100"

      # MySQL 慢查询
      - alert: MySQLSlowQueries
        expr: rate(mysql_global_status_slow_queries[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL 慢查询过多"
          description: "慢查询速率 {{ $value }}/s 超过阈值 10/s"

      # Redis 内存使用率 > 80%
      - alert: RedisHighMemoryUsage
        expr: (redis_memory_used_bytes / redis_memory_max_bytes) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis 内存使用率过高"
          description: "当前使用率 {{ $value | humanizePercentage }} 超过 80%"

  - name: system_alerts
    interval: 30s
    rules:
      # 主机 CPU 使用率 > 80%
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance)(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "主机 CPU 使用率过高"
          description: "实例 {{ $labels.instance }} CPU 使用率为 {{ $value }}%"

      # 主机内存使用率 > 85%
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "主机内存使用率过高"
          description: "实例 {{ $labels.instance }} 内存使用率为 {{ $value }}%"

      # 磁盘空间不足 < 10%
      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "磁盘空间不足"
          description: "实例 {{ $labels.instance }} 分区 {{ $labels.mountpoint }} 可用空间仅 {{ $value }}%"
```

---

## Grafana 仪表板

### 3.1 安装

```bash
docker run -d \
  --name tradermate-grafana \
  -p 3000:3000 \
  -v ./monitoring/grafana:/var/lib/grafana \
  -e "GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}" \
  --network tradermate_network \
  grafana/grafana:latest
```

### 3.2 预置仪表板 JSON

在 `monitoring/grafana/provisioning/dashboards/` 放置 JSON 仪表板文件：

#### API 业务仪表板 (API-Business.json)

```json
{
  "dashboard": {
    "title": "TraderMate API - 业务监控",
    "panels": [
      {
        "title": "QPS",
        "type": "graph",
        "targets": [{
          "expr": "sum(rate(http_requests_total[1m]))",
          "legendFormat": "Total QPS"
        }]
      },
      {
        "title": "响应延迟 P50/P95/P99",
        "type": "graph",
        "targets": [
          {"expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))", "legendFormat": "P50"},
          {"expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))", "legendFormat": "P95"},
          {"expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))", "legendFormat": "P99"}
        ]
      },
      {
        "title": "错误率 (5xx)",
        "type": "graph",
        "targets": [{
          "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
          "legendFormat": "Error Rate %"
        }]
      }
    ]
  }
}
```

#### 基础设施仪表板 (Infrastructure.json)

- 主机 CPU/Memory/Network/Disk
- 容器资源使用 (cAdvisor)
- MySQL 连接数、QPS、InnoDB 缓冲池
- Redis 内存、命中率、命令速率

### 3.3 仪表板导入

登录 Grafana (http://localhost:3000)，使用 Admin 账户，从 JSON 文件导入。

---

## 告警通知配置

### 4.1 Alertmanager 配置

创建 `monitoring/alertmanager/alertmanager.yml`：

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@tradermate.com'
  smtp_auth_username: 'alerts@tradermate.com'
  smtp_auth_password: 'app-password'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'email-notifications'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-oncall'
    - match:
        severity: warning
      receiver: 'slack-notifications'

receivers:
  - name: 'email-notifications'
    email_configs:
      - to: 'devops@tradermate.com'
        headers:
          subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        html: |
          <h3>Alert Summary</h3>
          {{ range .Alerts }}
          <p><strong>Labels:</strong> {{ .Labels }}</p>
          <p><strong>Annotations:</strong> {{ .Annotations }}</p>
          {{ end }}

  - name: 'slack-notifications'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/...'
        channel: '#tradermate-alerts'
        send_resolved: true
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ .CommonAnnotations.description }}'

  - name: 'pagerduty-oncall'
    pagerduty_configs:
      - routing_key: 'your-pagerduty-integration-key'

  - name: 'dingtalk'
    webhook_configs:
      - url: 'https://oapi.dingtalk.com/robot/send?access_token=...'
        send_resolved: true
        http_config:
          bearer_token: ''
        relabel_configs:
          - source_labels: [__address__]
            action: replace
            regex: (.+):9090
            target_label: __param_webhook_url
```

### 4.2 免费用方案

- **邮件**: SMTP (Gmail/企业邮箱)
- **钉钉/企业微信**: 使用 Webhook URL
- **Telegram**: Bot API (推荐备用)

### 4.3 Alertmanager Docker 集成

```yaml
  alertmanager:
    image: prom/alertmanager:latest
    container_name: tradermate_alertmanager
    restart: always
    ports:
      - "9093:9093"
    volumes:
      - ./monitoring/alertmanager:/etc/alertmanager:ro
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    networks:
      - tradermate_network
```

---

## 关键指标与告警规则

### 5.1 应用层 (FastAPI)

需要在代码中集成 Prometheus client：

```python
# app/api/metrics.py
from prometheus_client import Counter, Histogram, Gauge, generate_latest, REGISTRY
from fastapi import Request, Response

http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

http_request_duration_seconds = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['endpoint']
)

# 中间件
@app.middleware("http")
async def monitor_requests(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time

    http_requests_total.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()

    http_request_duration_seconds.labels(
        endpoint=request.url.path
    ).observe(duration)

    return response

@app.get("/metrics")
async def metrics():
    return Response(generate_latest(REGISTRY), media_type="text/plain")
```

**关键指标**:

| 指标 | 类型 | 说明 |
|------|------|------|
| `http_requests_total` | Counter | 请求总数 (按 method,endpoint,status) |
| `http_request_duration_seconds` | Histogram | 请求延迟分布 |
| `rq_jobs_queued_total` | Gauge | Redis 队列等待任务数 |
| `rq_jobs_completed_total` | Counter | 完成任务数 |
| `backtest_duration_seconds` | Histogram | 回测耗时 |

### 5.2 数据库层

- **MySQL**: `mysqld_exporter` 自动采集
  - `mysql_global_status_threads_connected`: 连接数
  - `mysql_slave_status_seconds_behind_master`: 复制延迟
  - `mysql_global_status_slow_queries`: 慢查询数

- **Redis**: `redis_exporter` 自动采集
  - `redis_memory_used_bytes`: 内存使用
  - `redis_connected_clients`: 连接数
  - `redis_keyspace_hits_total` / `redis_keyspace_misses_total`: 命中率

### 5.3 基础设施层

- **主机**: `node_exporter`
  - `node_cpu_seconds_total`
  - `node_memory_MemAvailable_bytes`
  - `node_filesystem_avail_bytes`
  - `node_network_receive_bytes_total`

- **容器**: `cAdvisor`
  - `container_cpu_usage_seconds_total`
  - `container_memory_usage_bytes`
  - `container_network_receive_bytes_total`

---

## 日志聚合方案

### 6.1 方案选择

| 方案 | 复杂度 | 成本 | 推荐度 |
|------|--------|------|--------|
| ELK Stack (Elasticsearch + Logstash + Kibana) | 高 | 高 | ⭐⭐ |
| Grafana Loki | 低 | 低 | ⭐⭐⭐⭐⭐ |
| 云服务 (Datadog, New Relic) | 低 | 很高 | ⭐⭐⭐ |
| 本地 rsyslog + 文件 | 极低 | 极低 | ⭐ |

**推荐**: **Grafana Loki** (轻量、与 Prometheus/Grafana 集成好)

### 6.2 Loki Docker Compose

```yaml
  loki:
    image: grafana/loki:latest
    container_name: tradermate_loki
    restart: always
    ports:
      - "3100:3100"
    volumes:
      - ./monitoring/loki:/loki
    command: -config.file=/loki/loki-local-config.yaml
    networks:
      - tradermate_network

  promtail:
    image: grafana/promtail:latest
    container_name: tradermate_promtail
    restart: always
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./monitoring/promtail:/etc/promtail:ro
    command: -config.file=/etc/promtail/promtail-config.yaml
    networks:
      - tradermate_network
```

### 6.3 Promtail 配置

`monitoring/promtail/promtail-config.yaml`:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        regex: 'stdout'
        action: keep

  - job_name: system
    static_configs:
      - targets:
        - localhost
        labels:
          job: varlogs
          __path__: /var/log/**/*.log

  - job_name: app
    static_configs:
      - targets:
        - localhost
        labels:
          job: tradermate
          __path__: /opt/tradermate/tradermate/logs/**/*.log
```

### 6.4 Grafana 数据源

在 Grafana 中添加 Loki 数据源 (`http://loki:3100`)，然后创建日志浏览仪表板。

---

## 部署步骤

### 7.1 一键部署 (Docker Compose)

```bash
cd /opt/tradermate

# 1. 创建目录结构
mkdir -p monitoring/{prometheus,rules,alertmanager,grafana,loki,promtail}

# 2. 复制配置文件 (从 docs/monitoring-templates/)
# cp -r monitoring-templates/* monitoring/

# 3. 编辑环境变量
cp .env.prod .
# 设置 GRAFANA_PASSWORD, ALERTMANAGER_SMTP_PASSWORD 等

# 4. 启动监控栈
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d prometheus grafana alertmanager loki promtail

# 5. 验证
curl http://localhost:9090/metrics  # Prometheus
curl http://localhost:3000/api/health  # Grafana
curl http://localhost:3100/ready  # Loki
```

### 7.2 验证指标采集

```bash
# 访问 Prometheus UI
http://localhost:9090/graph

# 查询示例指标
http_requests_total
node_cpu_seconds_total
mysql_global_status_threads_connected
```

### 7.3 测试告警

手动触发测试：

```bash
# 在 Prometheus 控制台执行
ALERT_RULE_COUNT
```

或在 Alertmanager 触发测试：

```bash
curl -X POST http://localhost:9093/-/reload  # 重载配置
# 临时修改规则阈值为极低值触发
```

---

## 附录：指标体系

### A. 业务指标 (SLI/SLO)

| 指标 | 目标 | 计算公式 |
|------|------|----------|
| 可用性 | 99.9% | `成功请求数 / 总请求数` |
| 延迟 | P95 < 200ms | `histogram_quantile(0.95, http_request_duration_seconds_bucket)` |
| 错误率 | < 0.1% | `5xx请求数 / 总请求数` |

### B. 导出到外部服务

支持 Grafana Cloud、Datadog、New Relic 等，只需配置 Prometheus remote write。

---

**文档结束**
