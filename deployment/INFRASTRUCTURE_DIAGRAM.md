# TraderMate 生产环境架构图

**版本**: 1.0  
**最后更新**: 2026-03-03

---

## 目录

1. [整体架构图](#整体架构图)
2. [Docker Compose 部署架构](#docker-compose-部署架构)
3. [Kubernetes 部署架构](#kubernetes-部署架构)
4. [网络流量图](#网络流量图)
5. [数据流图](#数据流图)
6. [组件依赖关系](#组件依赖关系)

---

## 整体架构图

### 架构概览 (Level 1)

```mermaid
graph TB
    subgraph "外部用户"
        U1[Web 用户]
        U2[移动用户]
        U3[管理员]
    end

    subgraph "CDN / 负载均衡层"
        LB[Cloudflare 或 AWS CloudFront]
        ELB[Nginx/Traefik]
    end

    subgraph "应用层 (Docker Compose / K8s)"
        API[API Service<br/>FastAPI 8000]
        Worker[Worker Service<br/>RQ Queue]
        Portal[Portal Frontend<br/> Static Assets]
    end

    subgraph "数据层"
        MySQL[MySQL 8.0<br/>主从复制]
        Redis[Redis 7<br/>哨兵/集群]
    end

    subgraph "监控层"
        Prometheus[Prometheus]
        Grafana[Grafana]
        Loki[Loki Logs]
    end

    U1 --> LB
    U2 --> LB
    U3 --> LB
    LB --> ELB
    ELB --> API
    ELB --> Portal
    API --> MySQL
    API --> Redis
    Worker --> MySQL
    Worker --> Redis
    Portal --> API
    Prometheus --> API
    Prometheus --> MySQL
    Prometheus --> Redis
    Prometheus --> Node
    Loki --> API
    Loki --> MySQL
    Loki --> Redis
```

---

## Docker Compose 部署架构

### 服务拓扑

```mermaid
graph LR
    subgraph "Docker Network: tradermate_network"
        API
        Worker
        Tradermate[TraderMate<br/>Desktop]
        MySQL
        Redis
    end

    subgraph "Host Network / Traefik"
        Portal
    end

    API -.->|depends_on| MySQL
    API -.->|depends_on| Redis
    Worker -.->|depends_on| MySQL
    Worker -.->|depends_on| Redis
    Tradermate -.->|depends_on| MySQL
    Portal -.->|links| API

    API -->|HTTP/JSON| MySQL
    API -->|Redis Queue| Redis
    Worker -->|RQ| Redis
    Tradermate -->|SQLAlchemy| MySQL
```

### 端口映射

| 服务 | 容器端口 | 主机端口 | 说明 | 生产建议 |
|------|----------|----------|------|----------|
| api | 8000 | 80 (Traefik) | REST API | 不直接暴露，通过 Ingress |
| portal | 80 | 443 (Traefik) | Nginx 前端 | 仅 HTTPS |
| mysql | 3306 | (none) | 数据库 | 禁止公网 |
| redis | 6379 | (none) | 队列 | 禁止公网 |

---

## Kubernetes 部署架构

### 命名空间与资源

```mermaid
graph TB
    NS[Namespace: tradermate]

    subgraph NS
        DepAPI[Deployment: api]
        SvcAPI[Service: ClusterIP]
        HPA[HPA: autoscale]
        Ingress[Ingress]

        DepWorker[Deployment: worker]
        SvcWorker[Service]

        DepMySQL[StatefulSet: mysql]
        PVCMySQL[PVC: data]
        SvcMySQL[Service]

        DepRedis[StatefulSet: redis]
        PVCRedis[PVC: data]
        SvcRedis[Service]

        DepPrometheus[Prometheus]
        DepGrafana[Grafana]
    end

    Ingress --> SvcAPI
    SvcAPI --> PodAPI[Pod: api]
    PodAPI --> PVCMySQL
    PodAPI --> PVCRedis
```

### Pod 调度策略

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tradermate-api
spec:
  nodeSelector:
    role: app
  tolerations:
  - key: "production"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: api
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
```

---

## 网络流量图

### 用户请求路径

```mermaid
sequenceDiagram
    participant User as Web Client
    participant CDN as Cloudflare CDN
    participant Ingress as Nginx/Traefik
    participant API as FastAPI
    participant DB as MySQL
    participant Cache as Redis

    User->>CDN: GET https://api.tradermate.com/health
    CDN->>Ingress: Forward (HTTPS termination)
    Ingress->>API: HTTP/1.1 (X-Forwarded-For)
    API->>DB: SELECT 1 (health check)
    DB-->>API: OK
    API-->>Ingress: {"status":"healthy"}
    Ingress-->>CDN: 200 OK
    CDN-->>User: {"status":"healthy"}
```

### 内部通信

```mermaid
graph LR
    Ingress --> API
    API -->|读写| DB
    API -->|队列| Cache
    Worker -->|消费| Cache
    Worker -->|读写| DB
    API -->|同步| Tradermate[TraderMate Desktop]
```

---

## 数据流图

### 用户登录与策略管理

```mermaid
flowchart TD
    Start[用户访问前端] --> Auth[登录页]
    Auth --> POST[/POST /api/auth/login/]
    POST --> Validate[验证用户名密码]
    Validate --> DB1[查询 users 表]
    DB1 -->|返回hash| Compare[bcrypt.compare]
    Compare -->|成功| GenJWT[生成 JWT Token]
    GenJWT --> Store[存储 access_token 到 localStorage]
    Store --> Redirect[跳转到 Dashboard]

    Dashboard --> LoadStrats[加载策略列表]
    LoadStrats --> GET[/GET /api/strategies/]
    GET --> DB2[查询 strategies 表]
    DB2 --> Return[返回策略JSON]
    Return --> Render[React 渲染]
```

### 回测任务流程

```mermaid
flowchart LR
    UI[Frontend UI] -->|submit| API1[POST /api/queue/backtest]
    API1 -->|push job| Redis[(Redis Queue)]
    API1 -->|save to| DB[(MySQL backtest_history)]

    Worker[RQ Worker] -->|pop job| Redis
    Worker -->|execute| VNPY[VeighNa Backtest Engine]
    VNPY -->|read data| TushareDB[(Tushare DB)]
    VNPY -->|write result| DB
    Worker -->|update status| Redis
    Worker -->|notify| WebSocket[WebSocket (可选)]

    UI -->|poll| API2[GET /api/backtest/{job_id}]
    API2 --> DB
    API2 --> UI

    UI -->|display| Charts[回测图表]
```

---

## 组件依赖关系

### 启动顺序依赖

```mermaid
graph TD
    order1[1. MySQL]
    order2[2. Redis]
    order3[3. API Service]
    order4[4. Worker Service]
    order5[5. Portal Frontend]

    order1 -->|healthy| order2
    order2 -->|healthy| order3
    order3 -->|healthy| order4
    order4 -.-> order5
    order3 -.-> order5
```

**说明**:
- API 和 Worker 依赖 MySQL + Redis
- Portal 仅依赖 API (可以独立启动)
- 健康检查 `depends_on.condition: service_healthy` 确保顺序

### 运行时数据依赖

```mermaid
graph LR
    API -->|连接池| MySQL
    API -->|pub/sub| Redis
    Worker -->|消息队列| Redis
    Worker -->|数据访问| MySQL
    Portal -->|REST API| API
    Backtest[Backtest Job] -->|市场数据| Tushare[(Tushare 数据源)]
    Backtest -->|回测引擎| VNPY[(VN.py 数据库)]
```

---

## 容灾架构 (多可用区)

### 高可用部署 (可选)

```mermaid
graph TB
    subgraph "AZ 1"
        LB1[Load Balancer]
        API1[API Pod #1]
        DB1[MySQL Primary]
    end

    subgraph "AZ 2"
        LB2[Load Balancer]
        API2[API Pod #2]
        DB2[MySQL Replica]
    end

    subgraph "AZ 3"
        RedisMaster[Redis Master]
        RedisReplica[Redis Replica]
    end

    LB1 --> API1
    LB2 --> API2
    API1 --> DB1
    API2 -->|read only| DB2
    DB1 -.->|replication| DB2
    RedisMaster -->|replication| RedisReplica
    API --> RedisMaster
```

**故障转移**:
- API 实例无状态，多副本自动负载均衡
- MySQL 使用 Orchestrator 或 MHA 自动 failover
- Redis 使用 Sentinel 自动故障转移

---

## 监控覆盖范围

```mermaid
graph TD
    Prometheus -->|Scrape| NodeExporter
    Prometheus -->|Scrape| CAdvisor
    Prometheus -->|Scrape| MySQLExporter
    Prometheus -->|Scrape| RedisExporter
    Prometheus -->|Scrape| APIMetrics

    AlertManager -->|Route| Prometheus
    AlertManager -->|Notify| Slack
    AlertManager -->|Notify| Email
    AlertManager -->|Notify| PagerDuty

    Grafana -->|Query| Prometheus
    Grafana -->|Query| Loki
    Grafana -->|Dashboard| AlertManager
```

---

**文档结束**

> **提示**: 以上图表使用 Mermaid 语法，可在支持 Mermaid 的 Markdown 编辑器（如 GitHub、GitLab、Obsidian）中可视化。如需 PNG/SVG 导出，使用 Mermaid CLI 或在线工具。
