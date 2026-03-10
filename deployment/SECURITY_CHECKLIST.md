# TraderMate 安全检查清单

**版本**: 1.0  
**最后更新**: 2026-03-03  
**合规目标**: GDPR (如适用)、金融数据保护

---

## 目录

1. [概览](#概览)
2. [部署前安全配置](#部署前安全配置)
3. [网络安全](#网络安全)
4. [应用安全](#应用安全)
5. [数据安全](#数据安全)
6. [密钥与Secrets管理](#密钥与secrets管理)
7. [访问控制](#访问控制)
8. [合规性](#合规性)
9. [渗透测试清单](#渗透测试清单)
10. [持续安全监控](#持续安全监控)

---

## 概览

本清单覆盖从代码到生产的全链路安全检查，适用于 Docker Compose 和 Kubernetes 部署。

### 安全等级说明

- 🔴 **必须** (阻断性问题，必须修复)
- 🟡 **应该** (重要风险，强烈建议修复)
- 🟢 **建议** (最佳实践，持续改进)

---

## 部署前安全配置

### 1.1 基础环境

- [ ] **操作系统**: 使用受支持版本 (Ubuntu 22.04+, RHEL 9+)  
- [ ] **最小化安装**: 仅安装必要包，移除测试/开发工具
- [ ] **时间同步**: NTP 服务启用并运行 (`timedatectl status`)
- [ ] **主机安全加固**:
  - [ ] 禁用 root SSH 登录 (`PermitRootLogin no`)
  - [ ] 使用密钥认证而非密码 (`PasswordAuthentication no`)
  - [ ] 更改默认 SSH 端口 (可选，减少扫描)
  - [ ] 安装并配置 fail2ban/iptables
- [ ] **内核参数优化** (`/etc/sysctl.conf`):
  ```bash
  net.ipv4.tcp_syncookies = 1
  net.ipv4.conf.all.accept_source_route = 0
  kernel.randomize_va_space = 2
  ```

### 1.2 Docker 安全

- [ ] **Docker 版本**: 保持最新稳定版 (`docker version`)
- [ ] **避免特权模式**: 不在容器中使用 `privileged: true`
- [ ] **非 root 用户**: 容器内使用非 root 用户运行 (Dockerfile 中 `USER`)
- [ ] **只读文件系统**: 对不需要写的挂载点使用 `:ro`
- [ ] **资源限制**: 设置 `mem_limit`, `cpu_quota` 防止 DoS
- [ ] **镜像来源**: 仅使用官方镜像或可信仓库
- [ ] **镜像扫描**: 每次构建使用 Trivy/Anchore 扫描漏洞

```bash
# 示例：Trivy 扫描
trivy image tradermate-api:latest
```

---

## 网络安全

### 2.1 网络分段

- [ ] **VPC/子网隔离**: 生产环境独立 VPC，禁止公网直接访问 DB
- [ ] **安全组/防火墙规则**:
  ```
  允许入站:
    - 80/tcp (HTTP → 重定向到 HTTPS)
    - 443/tcp (HTTPS)
    - 22/tcp (SSH) 仅限管理 IP
  
  禁止入站:
    - 3306 (MySQL) 仅允许应用服务器
    - 6379 (Redis) 仅允许应用服务器
  
  出站: 按需开放
  ```
- [ ] **Docker 网络**: 应用容器间使用自定义 bridge 网络，避免使用 `host` 模式

### 2.2 TLS/HTTPS

- [ ] **证书**: 使用 Let's Encrypt 或商业 CA，禁用自签名证书 (除内部测试)
- [ ] **TLS 版本**: 仅允许 TLS 1.2+，禁用 SSLv3/TLS 1.0/1.1
- [ ] **密码套件**: 使用强加密 (ECDHE-RSA-AES256-GCM-SHA384)
- [ ] **HSTS**: 启用 Strict-Transport-Security (Nginx/Traefik)
- [ ] **证书轮换**: Let's Encrypt 自动续期或商业证书 ≤ 1 年

**Nginx 示例配置**:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
ssl_prefer_server_ciphers off;
add_header Strict-Transport-Security "max-age=63072000" always;
```

### 2.3 DDoS 防护

- [ ] **WAF**: Cloudflare/AWS WAF 或 Nginx ModSecurity
- [ ] **速率限制**:
  ```nginx
  limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
  limit_req zone=api burst=50 nodelay;
  ```
- [ ] **连接限制**: `limit_conn_zone` 限制单 IP 并发连接

---

## 应用安全

### 3.1 身份认证与授权

- [ ] **密码策略**:
  - [ ] 最小长度 12 字符
  - [ ] 必须包含大小写字母、数字、特殊字符
  - [ ] 首次登录强制修改 (管理员 default hash 检测)
  - [ ] 密码历史记录防止重复使用
- [ ] **JWT 配置**:
  - [ ] `SECRET_KEY` 足够随机 (≥ 64 字符，使用 `openssl rand -hex 64`)
  - [ ] 合理设置过期时间 (access_token: 30分钟, refresh_token: 7天)
  - [ ] 使用 RS256 算法 (可选，比 HS256 更安全)
- [ ] **会话管理**:
  - [ ] 刷新 token 使用旋转机制
  - [ ] 登出时服务端失效 token (Redis 黑名单或 DB 记录)
- [ ] **多因素认证 (MFA)**: 生产环境管理员强制启用 TOTP (可选 Phase 2)

### 3.2 输入验证与注入防护

- [ ] **SQL 注入**:
  - [ ] 100% 使用参数化查询 (SQLAlchemy 已满足)
  - [ ] 验证 `mysql/init/*.sql` 无硬编码密码
  - [ ] 数据库用户最小权限原则 (应用用户非 root)
- [ ] **XSS**:
  - [ ] Frontend React 默认转义 (✓)
  - [ ] API 返回的用户输入再次转义或剥离 HTML
- [ ] **CSRF**:
  - [ ] FastAPI 无状态，不使用 Cookie 则无 CSRF 风险
  - [ ] 如需 Cookie，配置 CSRF token 中间件
- [ ] **SSRF**:
  - [ ] 代理 `fetch` 或外部请求时验证 URL 域名白名单

### 3.3 API 安全

- [ ] **速率限制**: `/api/auth/login` 等敏感端点限流 (Redis 计数器)
- [ ] **请求大小限制**: Nginx `client_max_body_size 1M`
- [ ] **敏感端点暴露**:
  - [ ] `/docs` 和 `/redoc` 生产环境禁用或 IP 白名单
  - [ ] `/metrics` 不暴露业务细节，使用 Basic Auth 或内部网络
- [ ] **CORS**: 生产环境 `CORS_ORIGINS` 仅允许 HTTPS 域名
- [ ] **版本控制**: API 路径包含版本前缀 (`/api/v1/...`) (已满足)

### 3.4 代码安全

- [ ] **依赖漏洞扫描**:
  ```bash
  pip-audit  # Python
  npm audit  # Node.js
  ```
- [ ] **SAST 工具集成**: Bandit, Semgrep (CI 已配置)
- [ ] **秘密扫描**: 禁止硬编码密码/密钥，使用 `git-secrets` 或 `truffleHog`
  ```bash
  git secrets --scan
  ```
- [ ] **最小权限原则**:
  - [ ] Dockerfile 中使用非 root 用户 (`appuser`)
  - [ ] 容器 capabilities 最小化 (`--cap-drop ALL`)

---

## 数据安全

### 4.1 数据库安全

- [ ] **强密码**:
  - [ ] `MYSQL_PASSWORD` ≥ 32 随机字符
  - [ ] 定期轮换 (每 90 天)
- [ ] **最小权限用户**:
  ```sql
  CREATE USER 'tradermate'@'%' IDENTIFIED BY '...';
  GRANT SELECT, INSERT, UPDATE, DELETE ON tradermate.* TO 'tradermate'@'%';
  GRANT SELECT ON vnpy.* TO 'tradermate'@'%';
  GRANT SELECT ON tushare.* TO 'tradermate'@'%';
  ```
  (当前使用 root，需修改)
- [ ] **加密传输**: 启用 TLS (`--require-secure-transport`)
- [ ] **加密静态数据** (可选):
  - [ ] InnoDB 表空间加密 (`innodb_file_per_table=ON`)
  - [ ] 敏感字段应用层加密
- [ ] **审计日志**:
  - [ ] 启用 MySQL Audit Plugin 或通用查询日志
  - [ ] 保留 ≥ 90 天

### 4.2 Redis 安全

- [ ] **密码认证**: `redis.conf` 设置 `requirepass`
- [ ] **命令重命名**: 禁用危险命令 (`FLUSHDB`, `FLUSHALL`, `CONFIG`)
- [ ] **仅绑定内网**: `bind 127.0.0.1` 或容器网络内

### 4.3 备份安全

- [ ] **加密备份**: 使用 `openssl enc` 或 `gpg` 加密备份文件
- [ ] **异地存储**: 备份上传到不同区域的云存储 (S3, OSS)
- [ ] **访问控制**: 仅授权人员可访问备份
- [ ] **保留策略**: 每日备份保留 30 天，每周备份保留 12 周，月度备份保留 12 月

---

## 密钥与Secrets管理

### 5.1 Docker Compose 方案

- [ ] **.env 文件权限**: `chmod 600 .env.prod`
- [ ] **不在代码中硬编码**: 检查所有配置文件
- [ ] **版本控制排除**: `.env.prod` 在 `.gitignore` 中

### 5.2 Kubernetes 方案 (推荐)

- [ ] **使用 Secret**:
  ```bash
  kubectl create secret generic tradermate-secret \
    --from-literal=mysql-password='...' \
    --from-literal=secret-key='...' \
    --namespace tradermate
  ```
- [ ] **开启 Secret 加密**: `--encryption-provider-config`
- [ ] **RBAC 限制 Secret 读取**: 仅允许服务账户读取

### 5.3 外部密钥管理 (高级)

- [ ] **HashiCorp Vault** 或 **AWS Secrets Manager**
- [ ] **动态密钥**: 数据库密码自动轮换
- [ ] **审计日志**: 所有密钥访问记录

---

## 访问控制

### 6.1 基础设施访问

- [ ] **堡垒机/VPN**: 生产服务器仅堡垒机可 SSH
- [ ] **IAM 最小权限**:
  - [ ] AWS IAM Policies (EC2, RDS, S3)
  - [ ] 禁止使用 root 账号日常操作
- [ ] **多因素认证**: 所有管理控制台启用 MFA

### 6.2 应用用户管理

- [ ] **默认管理员**: 首次启动后立即修改 `admin` 密码
- [ ] **账号生命周期**: 离职员工账号及时禁用
- [ ] **角色分离**:
  - `Admin`: 全部权限
  - `Trader`: 策略、回测
  - `Viewer`: 只读
- [ ] **会话超时**: 30 分钟无操作自动退出

---

## 合规性

### 7.1 GDPR (欧盟通用数据保护条例)

如果用户包含欧盟公民:

- [ ] **数据主体权利**: 提供数据导出/删除接口
- [ ] **隐私政策**: 明确数据处理目的和保留期限
- [ ] **同意管理**: 注册时明确勾选同意条款
- [ ] **数据泄露通知**: 72 小时内通知监管机构 (DPA)
- [ ] **DPO (数据保护官)**: 指定联系人

### 7.2 金融数据合规

- [ ] **数据本地化**: 中国用户数据存储在国内 (如阿里云北京)
- [ ] **审计追踪**: 所有交易指令、修改记录不可篡改
- [ ] **KYC/AML**: 如涉及实盘交易，集成身份验证

---

## 渗透测试清单

定期 (每季度或重大更新后) 执行：

### 8.1 自动化扫描

```bash
# 1. 依赖漏洞
pip-audit --requirement requirements.txt
npm audit
trivy fs .

# 2. 容器镜像扫描
trivy image tradermate-api:latest
docker scan tradermate-api:latest

# 3. 动态扫描 (OWASP ZAP/Burp Suite)
zap-cli quick-scan --spider --selfcontained https://api.tradermate.com

# 4. 基础设施扫描 (Nikto/Nmap)
nmap -sV -p 443 api.tradermate.com
nikto -h https://api.tradermate.com
```

### 8.2 手动测试重点

- [ ] **认证绕过**: JWT signature 验证、token 重放
- [ ] **越权访问**: 用户 A 访问用户 B 数据
- [ ] **注入**:
  - [ ] SQL: `' OR '1'='1` 等 payload
  - [ ] NoSQL (不适用)
  - [ ] 命令注入: API 参数执行 reverse shell
- [ ] **文件上传**: 无限制上传 .py 文件并执行
- [ ] **SSRF**: `/api/data/history` 外部 URL 参数
- [ ] **敏感信息泄露**: 错误信息暴露栈跟踪、目录遍历
- [ ] **业务逻辑**: 重复回测消耗资源、修改参数绕过限制

---

## 持续安全监控

### 9.1 日常检查

- [ ] **失败登录**: 异常多地登录、频繁失败
- [ ] **异常流量**: QPS 突增 500% 以上
- [ ] **新文件**: `/app` 目录意外修改
- [ ] **Cron 任务**: 计划任务未被篡改 (`crontab -l`)
- [ ] **SUID 文件**: 定期扫描 `find / -perm -4000 -type f`

### 9.2 日志审计

关键日志集中收集 (Loki/ELK):

- `/var/log/auth.log` (SSH 登录)
- MySQL 审计日志 (`log_output=FILE`, `audit_log_policy=ALL`)
- Redis 慢日志
- 应用访问日志 + 错误日志
- Docker 事件 (`docker events`)

**审计查询示例 (Loki)**:

```
{job="varlogs"} |= "Failed password"
{job="tradermate"} |= "sqlalchemy.exc"
{job="mysql"} |= "Access denied"
```

### 9.3 事件响应流程

1. **检测**: 告警通知 (PagerDuty/Slack)
2. **评估**: 确定影响范围 (数据泄露? 服务中断?)
3. **遏制**: 隔离受影响系统 (防火墙/关闭实例)
4. **根因分析**: 检查日志、导出内存快照
5. **修复**: 补丁、密钥轮换、权限收紧
6. **恢复**: 验证服务正常，重新上线
7. **复盘**: 编写报告，更新检测规则

---

## 检查执行记录

| 检查日期 | 检查人 | 通过项目 | 未通过项目 | 处理措施 |
|----------|--------|----------|------------|----------|
| 2026-03-03 | Frank | 25 / 42 | 17 | 见备注 |
| ... | | | | |

**备注**: 当前主要缺失
1. WAF 未配置 (计划使用 Cloudflare)
2. DB 应用用户权限过高 (需创建专用用户)
3. `/metrics` 和 `/docs` 未限制 (考虑 Basic Auth)

---

**文档结束**
