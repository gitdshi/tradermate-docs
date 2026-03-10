# 紧急修复步骤 - T-001 数据库阻塞

**触发条件**: 获得 MySQL 访问权限后立即执行

---

## 步骤 1: 验证数据库连接

```bash
# 替换为实际连接信息
MYSQL_HOST=your-db-host
MYSQL_PORT=3306
MYSQL_ROOT_PASSWORD=your-root-password

# 测试连接
mysql -h ${MYSQL_HOST} -P ${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1;"
```

**预期输出**: `+---+\n| 1 |\n+---+`

**如失败**: 检查网络、防火墙、用户名/密码

---

## 步骤 2: 创建专用应用用户

```bash
# 使用根用户执行
mysql -h ${MYSQL_HOST} -P ${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD} < mysql/init/create_tradermate_user.sql
```

**交互**: 脚本执行时会提示输入密码？不会，密码已内嵌在 SQL 中。  
**记得**: 修改 SQL 文件中的 `'YourStrongPassword!'` 为实际强密码！

---

## 步骤 3: 更新应用配置

编辑 `.env` 文件 (项目根目录):

```bash
# 替换以下变量
MYSQL_HOST=mysql  # 或容器IP，如为Docker Compose则保持"mysql"
MYSQL_PORT=3306
MYSQL_USER=tradermate  # ← 改为专用用户
MYSQL_PASSWORD=YourStrongPassword!  # ← 上一步设置的密码
REDIS_HOST=redis
REDIS_PORT=6379
DEBUG=false  # 生产必须
```

**注意**: 如果使用 Docker Compose，`MYSQL_HOST=mysql` 即可（容器名）。  
如果使用远程数据库，设为实际主机名。

---

## 步骤 4: 重启服务

### Docker Compose 方式

```bash
# 停止并重新启动
docker-compose down
docker-compose up -d api worker

# 验证
docker-compose ps
docker-compose logs -f api
```

### 裸机 Systemd 方式

```bash
sudo systemctl restart tradermate-api
sudo systemctl restart tradermate-worker
sudo journalctl -u tradermate-api -f
```

---

## 步骤 5: 验证健康检查

```bash
# 等待30秒让服务启动
sleep 30

# 检查 /health 端点
curl -f http://localhost:8000/health | jq .

# 预期输出
{
  "status": "healthy",
  "timestamp": "2026-03-03T...",
  "service": "tradermate",
  "dependencies": {
    "mysql": {"status": "healthy"},
    "redis": {"status": "healthy"}
  }
}
```

**如返回 503**: 查看日志确定原因
```bash
docker-compose logs api | tail -50
# 或
journalctl -u tradermate-api -n 50
```

---

## 步骤 6: 通知 Tester 继续验证

一旦 `/health` 返回 `healthy`，回复 T-001 验证团队:

```
✅ O-002 已修复: 数据库连接恢复，应用健康检查通过
请 Tester 继续执行 T-001 剩余验证步骤。
```

---

## 常见问题排查

### Q1: `Access denied for user 'tradermate'@'%'`

**原因**: 密码错误或用户未创建

**解决**:
```bash
mysql -u root -p -e "SELECT user, host FROM mysql.user WHERE user='tradermate';"
# 确认用户存在且 host 为 '%' 或 'localhost'
```

### Q2: `Can't connect to MySQL server on 'host'`

**原因**: 网络不通、防火墙、MySQL bind 地址

**解决**:
```bash
# 检查 MySQL 监听
mysql -h host -P 3306 -u root -p -e "status"
# 或
telnet host 3306

# 检查 MySQL 配置: bind-address = 0.0.0.0 (而非 127.0.0.1)
```

### Q3: Redis 连接失败

**原因**: Redis 未运行或需要密码

**解决**:
```bash
redis-cli -h redis-host -p 6379 ping
# 应返回 PONG

# 如果配置了密码:
redis-cli -h redis-host -p 6379 -a YourRedisPassword ping
```

### Q4: 应用启动后 `mysql.connector.errors.ProgrammingError: (1049, "Unknown database 'tradermate'")`

**原因**: 数据库未初始化

**解决**:
```bash
# 执行所有初始化 SQL
mysql -u root -p < mysql/init/tradermate.sql
mysql -u root -p < mysql/init/vnpy.sql
mysql -u root -p < mysql/init/tushare.sql
mysql -u root -p < mysql/init/akshare.sql
```

---

## 修复完成检查清单

- [ ] 数据库连接成功 (`mysql -h ...` 可执行)
- [ ] 专用用户 `tradermate` 已创建
- [ ] 授予最小权限成功
- [ ] `.env` 已更新为新用户
- [ ] API 服务重启成功
- [ ] `/health` 返回 `{"status":"healthy"}`
- [ ] Tester 通知已发送

---

**完成时间记录**: _______________
**执行人**: _______________
**验证人**: Tester
