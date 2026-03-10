# Deployment 部署文档导航

欢迎查看TraderMate部署相关文档。本文档是部署、运维和监控的**总览索引**，帮助你快速找到所需信息。

---

## 📋 部署前准备

在开始部署前，请确保：

1. ✅ 已阅读 [开发环境设置](../development/ENVIRONMENT_SETUP.md) 了解技术栈
2. ✅ 服务器满足 [系统要求](#系统要求)（见下文）
3. ✅ 已获取所有必需的外部服务凭证（Tushare token、SSL证书等）
4. ✅ 已熟悉 [环境变量参考](../development/ENV_VARIABLES_REFERENCE.md)

---

## 🗂️ 文档结构

本目录包含以下文档，按阅读顺序组织：

### 1. 核心部署指南

**[DEPLOYMENT_RUNBOOKS.md](./DEPLOYMENT_RUNBOOKS.md)** - **主要阅读** (19KB)
> 完整的部署流程手册，涵盖：
> - 架构概览和部署方式选择（Docker / Kubernetes / 裸机）
> - 分步部署指令
> - 环境变量配置
> - 数据库初始化和迁移
> - 验证测试清单
> - 回滚策略
> - 日常维护操作

**何时阅读**: 第一次部署或重新部署时从头到尾阅读

---

### 2. 专项配置指南

**[MONITORING_CONFIG.md](./MONITORING_CONFIG.md)** (20KB)
> 监控系统配置指南：
> - Prometheus指标暴露和采集
> - Grafana仪表板导入和配置
> - 关键告警规则及其阈值
> - 日志聚合方案（ELK/Loki）
> - 性能基准和容量规划

**何时阅读**: 需要配置监控或理解监控指标时

**[SECURITY_CHECKLIST.md](./SECURITY_CHECKLIST.md)** (12KB)
> 安全配置检查清单：
> - HTTPS/SSL/TLS配置（Nginx/Traefik）
> - 防火墙规则（UFW/iptables）
> - 密钥管理和轮换策略
> - JWT安全设置
> - CORS和访问控制
> - 漏洞扫描建议

**何时阅读**: 安全审计前或生产环境配置时

**[BACKUP_RECOVERY_PLAN.md](./BACKUP_RECOVERY_PLAN.md)** (12KB)
> 备份与灾难恢复计划：
> - 数据库备份策略（全量/增量、自动化）
> - 配置文件备份
> - 恢复流程（分步骤）
> - 备份验证和测试
> - RTO/RPO目标

**何时阅读**: 设置备份策略或执行恢复演练时

---

### 3. 基础设施参考

**[INFRASTRUCTURE_DIAGRAM.md](./INFRASTRUCTURE_DIAGRAM.md)** (7.8KB)
> 基础设施架构图（ASCII + 描述）
> - 服务拓扑
> - 网络流量路径
> - 数据流和依赖

**何时阅读**: 理解系统架构时

**注意**: `INFRASTRUCTURE_STATUS.md` 已归档为**历史评估报告**，不再维护。当前状态请参考对应Agent的最新报告。

**[docker-compose.prod.yml](./docker-compose.prod.yml)** (8KB)
> 生产环境Docker Compose配置文件
> - 服务定义（api, worker, frontend, mysql, redis）
> - 网络和卷配置
> - 资源限制和健康检查
> - 环境变量注入

**何时阅读**: 理解部署配置细节或自定义配置时

---

## 🎯 按场景查找

### "我想部署TraderMate到新服务器"
1. 阅读 [DEPLOYMENT_RUNBOOKS.md](./DEPLOYMENT_RUNBOOKS.md) 完整流程
2. 配置 `docker-compose.prod.yml` 中的环境变量
3. 参考 [SECURITY_CHECKLIST.md](./SECURITY_CHECKLIST.md) 完成安全加固
4. 运行部署验证（DEPLOYMENT_RUNBOOKS.md中有清单）

### "我需要配置监控和告警"
1. 阅读 [MONITORING_CONFIG.md](./MONITORING_CONFIG.md)
2. 导入Grafana仪表板（如有）
3. 配置Prometheus scrape配置
4. 调整告警阈值

### "发生故障，需要排查"
1. 参考 [DEPLOYMENT_RUNBOOKS.md](./DEPLOYMENT_RUNBOOKS.md) 的"常见故障处理"章节
2. 查看相关日志（文档中有日志位置说明）
3. 参考 [TROUBLESHOOTING.md](../development/TROUBLESHOOTING.md) 通用故障排查

### "需要备份数据库并恢复"
1. 阅读 [BACKUP_RECOVERY_PLAN.md](./BACKUP_RECOVERY_PLAN.md)
2. 遵循"自动备份"或"手动备份"流程
3. 演练恢复流程（使用测试环境）

### "如何进行安全审计"
1. 通读 [SECURITY_CHECKLIST.md](./SECURITY_CHECKLIST.md)
2. 逐项检查并标记 ✅/❌
3. 修复 ❌ 项后重新检查

---

## 📊 文档维护

| 文档 | 维护者 | 更新频率 | 上次更新 |
|------|--------|----------|----------|
| DEPLOYMENT_RUNBOOKS.md | @operator | 每次部署流程变更 | 2026-03-03 |
| MONITORING_CONFIG.md | @operator | 监控需求变更 | 2026-03-03 |
| SECURITY_CHECKLIST.md | @operator | 安全威胁或配置变更 | 2026-03-03 |
| BACKUP_RECOVERY_PLAN.md | @operator | 备份策略变更 | 2026-03-03 |
| INFRASTRUCTURE_DIAGRAM.md | @operator | 架构变更 | 2026-03-03 |
| docker-compose.prod.yml | @operator | 服务配置变更 | 2026-03-03 |
| DEPLOYMENT_VALIDATION_*.md | @operator | 每次验证后 | 2026-03-03 |

---

## 🔗 相关文档

- [开发环境设置](../development/ENVIRONMENT_SETUP.md) - 本地开发配置
- [架构文档](../architecture/README.md) - 系统架构详解
- [故障排查](../development/TROUBLESHOOTING.md) - 通用问题解决
- [项目管理](../project-management/TASKS.md) - 任务跟踪

---

## 💡 建议阅读顺序（新运维人员）

1. **第一天**: ENVIRONMENT_SETUP.md（了解技术栈） → 本README → DEPLOYMENT_RUNBOOKS.md（完整阅读）
2. **第一周**: 在测试环境实践部署流程
3. **后续**: 根据需要阅读专项文档（监控、安全、备份）

---

** Maintainer**: @operator
** last Updated**: 2026-03-03
** 相关标准**: [DOCUMENTATION_STANDARDS.md](../../standards/DOCUMENTATION_STANDARDS.md)
