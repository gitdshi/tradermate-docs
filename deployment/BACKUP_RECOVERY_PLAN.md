# TraderMate 备份与灾难恢复方案

**版本**: 1.0  
**最后更新**: 2026-03-03  
**RTO (恢复时间目标)**: 4 小时  
**RPO (恢复点目标)**: 24 小时 (数据库) / 1 小时 (Redis)

---

## 目录

1. [概览](#概览)
2. [备份策略](#备份策略)
3. [备份执行流程](#备份执行流程)
4. [恢复流程](#恢复流程)
5. [故障转移与高可用](#故障转移与高可用)
6. [验证与演练](#验证与演练)
7. [工具与脚本](#工具与脚本)

---

## 概览

### 备份对象

| 组件 | 数据类型 | 备份频率 | 保留期 | 恢复优先级 |
|------|----------|----------|--------|-----------|
| MySQL | 交易数据、用户策略、回测结果 | 每日 02:00 + Binlog 实时 | 30 天 (每日) + 12 月 (月度) | 1 (最高) |
| Redis | 任务队列、缓存会话 | 每小时 RDB + AOF 实时 | 7 天 | 2 |
| 代码/配置 | 应用代码、docker-compose、Nginx 配置 | 每次变更 (Git) | 永久 | 3 |
| 日志 | 应用日志、访问日志 | 每日压缩归档 | 90 天 | 4 (可选) |

### 存储选择

- **本地/同城**: NFS/SMB 共享存储或本地磁盘 (快速恢复)
- **异地/云端**: AWS S3 / 阿里云 OSS / 腾讯云 COS (防灾)
- **加密**: 使用 GPG 或 openssl 加密敏感数据

---

## 备份策略

### 2.1 MySQL 全量 + Binlog

**类型**: Percona XtraBackup + 二进制日志

**优势**:
- 热备份，无需停机
- 支持增量备份 (节省空间和时间)
- 精确到秒级恢复 (基于 binlog)

**保留策略**:
- 每日全量: 保留 30 天
- 每周全量: 保留 12 周
- 每月全量: 保留 12 月

**命名规范**:
```
/backup/mysql/full-2026-03-01.sql.gz
/backup/mysql/binlog-2026-03-01.000001.sql.gz
```

### 2.2 Redis RDB + AOF

**类型**: RDB 快照 + 写入时追加文件

**配置** (`redis.conf`):

```conf
save 3600 1
save 300 100
save 60 10000
dbfilename dump.rdb
dir /backup/redis/
appendonly yes
appendfsync everysec
```

**保留策略**:
- 每小时 RDB 文件, 保留 24 小时
- AOF 文件持续写入, 每日轮转归档

**注意**: Redis AOF 恢复较慢，RDB 快，根据 RTO 选择。

### 2.3 代码/配置备份

- **Git**: 已包含 (GitHub/GitLab)
- **Docker 镜像**: CI/CD 自动推送镜像仓库，标签包含 Git SHA
- **手动配置**: Nginx、systemd unit 文件同步到 Git

### 2.4 日志归档

使用 **logrotate** 或 Docker 日志驱动:

```bash
# /etc/logrotate.d/tradermate
/opt/tradermate/logs/*.log {
    daily
    rotate 90
    compress
    delaycompress
    missingok
    notifempty
    create 644 ubuntu ubuntu
    postrotate
        docker-compose -f /opt/tradermate/docker-compose.yml logs --no-color > /backup/logs/tradermate-$(date +%F).log
    endscript
}
```

---

## 备份执行流程

### 3.1 Docker Compose 环境

创建 `scripts/backup.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/backup/mysql"
DATE=$(date +%F)
RETENTION_DAYS=30
MYSQL_CONTAINER="tradermate_mysql"

echo "[$(date)] Starting MySQL backup..."

# 1. 全量备份 (使用 Percona XtraBackup)
docker exec $MYSQL_CONTAINER sh -c "
  xtrabackup --backup \
    --user=root \
    --password='${MYSQL_PASSWORD}' \
    --target-dir=/tmp/backup \
    --databases='tradermate vnpy tushare akshare' \
  && tar -czf /tmp/backup-${DATE}.tar.gz -C /tmp backup \
  && mv /tmp/backup-${DATE}.tar.gz /backup/mysql/
"

# 2. 备份 Binlog (用于增量恢复)
docker exec $MYSQL_CONTAINER mysql -u root -p${MYSQL_PASSWORD} -e "FLUSH LOGS; SHOW BINARY LOGS;" \
  | tail -n +2 | awk '{print $1}' > /tmp/binlog_list.txt

while read -r binlog; do
  docker exec $MYSQL_CONTAINER mysqlbinlog --read-from-remote-server \
    --host=localhost --user=root --password=${MYSQL_PASSWORD} \
    --raw ${binlog} > /backup/mysql/binlog-${DATE}-${binlog}.sql
done < /tmp/binlog_list.txt

# 3. 同步到云存储 (可选)
aws s3 sync /backup/mysql/ s3://tradermate-backups/mysql/ --delete

# 4. 清理旧备份
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "binlog-*.sql" -mtime +7 -delete

echo "[$(date)] Backup completed successfully."
```

**Cron 定时任务**:

```bash
# crontab -e (root)
0 2 * * * /opt/tradermate/scripts/backup.sh >> /var/log/backup.log 2>&1
```

### 3.3 Redis 备份

Redis 自动 RDB 已配置到 `/backup/redis/`，每小时一次。

**同步到云存储**:

```bash
0 * * * * aws s3 sync /backup/redis/ s3://tradermate-backups/redis/ --delete
```

### 3.4 裸机环境

备份脚本逻辑相同，但直接 `mysqldump` 或 `xtrabackup` 到本地磁盘。

---

## 恢复流程

### 4.1 MySQL 恢复流程

#### 场景 A: 全量恢复 (最新备份)

```bash
# 1. 停止数据库 (Docker)
docker-compose stop mysql

# 2. 准备新数据目录
docker exec tradermate_mysql rm -rf /var/lib/mysql/*
docker exec tradermate_mysql mkdir -p /var/lib/mysql

# 3. 恢复全量备份
docker exec -i tradermate_mysql sh -c "
  mkdir -p /tmp/restore && \
  tar -xzf /backup/mysql/full-2026-03-01.tar.gz -C /tmp/restore && \
  xtrabackup --prepare --target-dir=/tmp/restore/backup
  xtrabackup --copy-back --target-dir=/tmp/restore/backup
"

# 4. 修正权限
docker exec tradermate_mysql chown -R mysql:mysql /var/lib/mysql

# 5. 启动 MySQL
docker-compose start mysql

# 6. 应用 Binlog (恢复到指定时间点)
# docker exec -i tradermate_mysql mysql -u root -p${MYSQL_PASSWORD} < /backup/mysql/binlog-2026-03-01-000001.sql

# 7. 验证
docker exec tradermate_mysql mysql -u root -e "SHOW DATABASES;"
```

#### 场景 B: 单表恢复到指定时间点 (PITR)

```bash
# 1. 恢复到临时实例
docker exec tradermate_mysql sh -c "
  mkdir -p /tmp/pitr && \
  cd /tmp/pitr && \
  mysqlbinlog --stop-datetime='2026-03-01 14:30:00' \
    /backup/mysql/binlog-2026-03-01.000001.sql > apply.sql
"

# 2. 在临时库恢复全量并应用 apply.sql，导出单表
# mysqldump -u root -p tradermate strategies > strategies_2026-03-01.sql
# 导入生产库
```

### 4.2 Redis 恢复

```bash
# 1. 停止 Redis
docker-compose stop redis

# 2. 替换 RDB 文件
cp /backup/redis/dump.rdb /var/lib/docker/volumes/tradermate_redis_data/_data/dump.rdb
# 或挂载卷直接复制:
docker cp /backup/redis/dump.rdb tradermate_redis:/data/dump.rdb

# 3. 启动 Redis
docker-compose start redis

# 4. 验证
docker exec tradermate_redis redis-cli PING
docker exec tradermate_redis redis-cli DBSIZE
```

### 4.3 前端/配置恢复

```bash
cd /opt/tradermate
git checkout main
git pull origin main
# 回滚到特定 commit
git reset --hard <commit-sha>

docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

---

## 故障转移与高可用

### 5.1 MySQL 主从复制

**架构**:
```
Primary (Master) → Replica (Read Replica)
```

**配置 Primary** (`my.cnf`):

```ini
[mysqld]
server-id=1
log_bin=mysql-bin
binlog_format=ROW
expire_logs_days=7
```

```sql
CREATE USER 'replica'@'%' IDENTIFIED BY 'replica-password';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%';
FLUSH PRIVILEGES;
SHOW MASTER STATUS;  # 记录 File 和 Position
```

**配置 Replica**:

```ini
[mysqld]
server-id=2
read_only=1
```

```sql
CHANGE MASTER TO
  MASTER_HOST='primary-ip',
  MASTER_USER='replica',
  MASTER_PASSWORD='replica-password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=1234;
START SLAVE;
```

**故障转移**:
1. 检测主库宕机 (`SELECT 1 FROM mysql.slave_master_info`)
2. 提升从库为主库 (`STOP SLAVE; RESET MASTER;`)
3. 应用层切换连接字符串
4. 重建新从库 (原主库恢复后作为新从库)

### 5.2 Redis Sentinel

对于生产环境，建议使用 **Redis Cluster** (≥6 节点) 而非单实例。

Sentinel 配置 (`sentinel.conf`):

```conf
port 26379
sentinel monitor mymaster redis-master 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

**客户端配置**: 使用支持 Sentinel 的库 (redis-py Sentinel)

### 5.3 API 多实例负载均衡

Docker Compose 使用 Traefik 自动负载均衡 (见 `DEPLOYMENT_RUNBOOKS.md`)。  
K8s 使用 Service `ClusterIP` + Ingress。

---

## 验证与演练

### 6.1 备份验证 (每周)

```bash
# 1. 检查备份文件存在且完整
ls -lh /backup/mysql/full-*.tar.gz
gzip -t /backup/mysql/full-2026-03-01.tar.gz  # 测试完整性

# 2. 启动临时 MySQL 容器恢复最新备份
docker run -d --name mysql-test -e MYSQL_ROOT_PASSWORD=test \
  -v /backup/mysql/latest:/backup \
  mysql:8.0 --skip-networking

# 3. 执行恢复脚本 (确保无错误)
docker exec mysql-test sh -c "
  xtrabackup --prepare --target-dir=/backup &&
  xtrabackup --copy-back --target-dir=/backup &&
  chown -R mysql:mysql /var/lib/mysql
"

# 4. 连接并查询数据行数
docker exec mysql-test mysql -u root -ptest -e "SELECT COUNT(*) FROM tradermate.users;"

# 5. 清理
docker rm -f mysql-test
```

**输出示例**:
```
[INFO] Backup size: 2.3 GB
[INFO] Restore time: 8m 32s
[INFO] Row count verified: 42 users
```

### 6.2 灾难恢复演练 (季度)

**场景**: Primary 数据库节点完全损坏

**步骤**:
1. 通知团队，进入演练模式 (模拟真实事件)
2. 停止所有应用 (`docker-compose down`)
3. 选择最近的全量备份 (例如 24 小时前)
4. 启动临时 MySQL 从备份恢复 (见 4.1)
5. 应用 Binlog 到故障时刻
6. 启动 Redis 从 RDB 恢复
7. 启动全部服务
8. 执行健康检查:
   ```bash
   curl -f http://localhost:8000/health
   docker-compose logs --tail=100 api | grep -i error
   ```
9. 执行冒烟测试:
   - 登录 API
   - 查询策略列表
   - 提交简短回测
10. 记录 RTO (实际耗时) 和 RPO (数据丢失量)
11. 清理环境，恢复生产正常配置
12. 编写演练报告，更新预案

**评估标准**:
- ✅ RTO < 4 小时
- ✅ RPO < 1 小时 (最多丢失 1 小时内未备份的数据)
- ✅ 应用功能正常

---

## 工具与脚本

### 7.1 备份管理脚本

`scripts/backup-manager.sh`:

```bash
#!/usr/bin/env bash
# Usage: backup-manager.sh [list|verify|restore|prune]

case "$1" in
  list)
    echo "=== MySQL Backups ==="
    ls -lh /backup/mysql/full-*.tar.gz
    echo "=== Redis Backups ==="
    ls -lh /backup/redis/dump.rdb*
    ;;
  verify)
    LATEST=$(ls -t /backup/mysql/full-*.tar.gz | head -1)
    echo "Verifying $LATEST..."
    gzip -t $LATEST && echo "✅ OK" || echo "❌ Corrupted"
    ;;
  restore)
    # restore <backup-file> [target-dir]
    echo "Restoring $2 to $3..."
    ;;
  prune)
    # 清理超过保留期的备份
    find /backup -type f -mtime +30 -delete
    echo "Old backups pruned."
    ;;
  *)
    echo "Commands: list|verify|restore|prune"
    ;;
esac
```

### 7.2 监控备份任务

Prometheus 规则示例:

```yaml
- alert: BackupFailed
  expr: backup_last_success_timestamp < (time() - 86400)
  for: 1h
  annotations:
    summary: "Backup job has not succeeded in 24 hours"
```

使用 [automatic-backup-exporter](https://github.com/dschoen/backup_exporter) 输出备份指标。

---

## 附录

### A. 加密备份示例

```bash
# 加密
gpg --batch --passphrase "${BACKUP_PASSPHRASE}" \
  --symmetric --cipher-algo AES256 \
  /backup/mysql/full-2026-03-01.tar.gz

# 解密
gpg --batch --passphrase "${BACKUP_PASSPHRASE}" \
  --decrypt full-2026-03-01.tar.gz.gpg > full-2026-03-01.tar.gz
```

### B. 云存储同步 (AWS S3)

```bash
aws s3 sync /backup/mysql/ s3://tradermate-backups/mysql/ \
  --storage-class STANDARD_IA \
  --delete \
  --sse AES256
```

### C. 恢复检查清单

- [ ] 备份文件完整无损 (`gzip -t`, `sha256sum`)
- [ ] 目标存储空间充足 (`df -h`)
- [ ] 环境变量 (.env) 已准备
- [ ] 服务已停止 (`docker-compose stop`)
- [ ] 恢复日志记录完整
- [ ] 恢复后验证: 数据一致性、应用可访问
- [ ] 通知相关人员恢复完成

---

**文档结束**
