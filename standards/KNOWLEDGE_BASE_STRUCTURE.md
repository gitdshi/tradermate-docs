# KNOWLEDGE_BASE 知识库组织方案

**提案日期**: 2026-03-02
**提案人**: Writer
**目标**: 建立可扩展、易于导航的系统化知识库

---

## 设计理念

TraderMate知识库采用 **"分层、按受众组织、可搜索"** 的结构。不是简单堆砌文件，而是构建一个**学习路径和参考系统**。

### 核心原则
1. **新用户快速上手** - 5分钟内知道在哪里找
2. **开发者深度查询** - 详尽的技术参考
3. **运维可靠操作** - Runbook驱动的故障处理
4. **长期可维护性** - 清晰的结构便于添加新内容

---

## 目录结构详解

### 根入口：`KNOWLEDGE_BASE/README.md`

这是主要的文档导航页，类似于 `docs/README.md` 的扩展版，但更专注于**用户导航**而非技术概览。

模板示例：
```markdown
# TraderMate Knowledge Base

欢迎来到TraderMate知识库。本文档为您提供从入门到精通的全方位指南。

## 🎯 快速导航

| 如果你是... | 从这里开始 |
|------------|-----------|
| **新用户** | [用户指南](user-guides/getting-started.md) |
| **策略开发者** | [API参考](api-reference/overview.md) |
| **贡献开发者** | [开发者指南](development/contributing.md) |
| **运维人员** | [部署手册](operations/deployment.md) |
| **遇到问题** | [故障排查](troubleshooting/common-errors.md) |

## 📚 文档结构

```tree
KNOWLEDGE_BASE/
├── 用户指南 (user-guides/)     # 如何使用平台
├── API参考 (api-reference/)   # 开发者API文档
├── 运维操作 (operations/)     # 部署和监控
├── 开发贡献 (development/)    # 为项目做贡献
├── 故障排查 (troubleshooting/)# 问题诊断
├── 学习路径 (learning/)       # 教程和最佳实践
└── 参考 (reference/)          # 术语、缩写、汇总表
```

## 🔍 搜索

使用 `Ctrl+F` 搜索当前页面，或访问 [全文搜索站点](link-to-search)（如部署）。

## 📖 更新日志

最新变更见 [CHANGELOG.md](../../CHANGELOG.md)。
```

---

## 子目录详解

### 1️⃣ `user-guides/` - 用户指南

**目标读者**: 个人交易者、最终用户
**难度**: 入门到中级
**格式**: 任务驱动（"如何做X"）

```
user-guides/
├── README.md                  # 用户文档入口
├── getting-started.md         # 快速入门（30分钟上手）
├── account-setup.md           # 账户注册和设置
├── strategies/
│   ├── overview.md           # 策略概念介绍
│   ├── create.md             # 创建策略
│   ├── edit.md               # 编辑策略
│   ├── import.md             # 导入策略
│   └── examples/             # 内置策略示例
├── backtesting/
│   ├── single-backtest.md    # 单次回测
│   ├── batch-backtest.md     # 批量回测
│   ├── interpreting-results.md # 结果解读
│   └── optimization.md       # 参数优化
├── market-data/
│   ├── viewing-charts.md     # 查看K线图
│   ├── technical-indicators.md # 技术指标
│   └── exporting-data.md     # 导出数据
├── risk-management.md        # 风险管理功能介绍
├── portfolio-tracker.md      # 组合跟踪（如适用）
├── faq.md                    # 用户常见问题
└── glossary.md               # 用户术语表（简化版）
```

**关键文档示例** (`getting-started.md`):
```markdown
# 快速入门：您的第一个回测

> 预计时间：30分钟

本指南将带您完成完整的回测流程，从注册到查看结果。

## 前置条件
- 已注册账户并登录
- 电脑联网

## 步骤1：创建策略

1. 导航至 **策略** 页面
2. 点击 **"新建策略"** 按钮
3. 填写策略信息...

## 步骤2：运行回测

...

## 步骤3：分析结果

...

## 常见问题

Q: 回测失败了怎么办？
A: 检查策略代码是否有语法错误...

## 下一步
- [学习更多策略技巧](strategies/create.md)
- [查看所有API](../api-reference/overview.md)
```

---

### 2️⃣ `api-reference/` - API参考

**目标读者**: 策略开发者、集成开发者
**难度**: 中级到高级
**格式**: 技术参考（端点、参数、响应）

```
api-reference/
├── README.md                  # API文档入口和导航
├── overview.md                # API概览、认证、速率限制
├── authentication.md          # 认证详细流程
├── models/                    # 数据模型（Pydantic schemas）
│   ├── user.md
│   ├── strategy.md
│   ├── backtest.md
│   └── common.md
├── endpoints/
│   ├── strategies.md          # 策略CRUD
│   ├── backtest.md            # 回测API
│   ├── data.md                # 市场数据
│   ├── queue.md               # 队列管理
│   └── optimization.md        # 优化API
├── examples/
│   ├── python-client.md       # Python客户端示例（requests）
│   ├── register-login.md      # 注册登录完整流程
│   ├── run-backtest.py        # 回测完整代码示例
│   ├── batch-optimization.py  # 批量优化示例
│   └── webhook-integration.md # Webhook集成（如适用）
├── errors.md                  # 错误代码和释义
└── changelog.md               # API变更历史（与CHANGELOG.md同步）
```

**设计决策**:
- 每个端点独立文件，便于维护和链接
- `models/` 目录集中管理数据模型
- `examples/` 提供可运行的完整代码

---

### 3️⃣ `operations/` - 运维操作

**目标读者**: DevOps、运维工程师
**难度**: 高级
**格式**: 操作手册和Runbook

```
operations/
├── README.md                  # 运维文档入口
├── deployment.md              # 完整部署指南（P0）
├── infrastructure.md          # 基础设施说明（服务器、网络）
├── configuration/             # 配置详解
│   ├── environment-variables.md
│   ├── docker-compose.md
│   └── security.md
├── monitoring/                # 监控
│   ├── metrics.md             # 关键指标定义
│   ├── dashboards.md          # Grafana仪表板
│   ├── alerts.md              # 告警规则和阈值
│   └── log-aggregation.md     # 日志聚合方案
├── runbooks/                  # Runbook（P0）
│   ├── startup.md             # 服务启动检查
│   ├── shutdown.md            # 优雅关闭
│   ├── backup-restore.md      # 备份恢复
│   ├── scaling.md             # 水平扩展
│   ├── incident-response.md   # 事故响应
│   └── common-failures.md     # 常见故障处理
├── maintenance/               # 日常维护
│   ├── daily-checks.md
│   ├── weekly-tasks.md
│   ├── monthly-tasks.md
│   └── upgrade-procedures.md
└── disaster-recovery.md       # 灾难恢复计划
```

**关键文档示例** (`runbooks/startup.md`):
```markdown
# Runbook: 服务启动和健康检查

## 适用场景
- 系统重启后
- 部署新版本后
- 收到"服务不可用"告警

## 检查清单

### 1. 数据库状态
```bash
# 检查MySQL是否运行
docker ps | grep mysql
# 或
systemctl status mysql

# 验证连接
mysql -u root -p -e "SELECT 1"
```
**结果**: 应返回 `1`

### 2. Redis状态
```bash
redis-cli ping
```
**期望**: 输出 `PONG`

### 3. API服务
```bash
curl http://localhost:8000/health
```
**期望**: `{"status":"healthy",...}`

### 4. 前端服务
```bash
curl -I http://localhost:5173
```
**期望**: `HTTP/1.1 200 OK`

## 故障处理

| 症状 | 可能原因 | 解决步骤 |
|------|----------|----------|
| API健康检查失败 | 数据库连接失败 | 1. 检查`docker-compose logs api`<br>2. 验证`DATABASE_URL` |
| ... | ... | ... |

## 联系人
- Backend: @coder
- DevOps: @operator
```

---

### 4️⃣ `development/` - 开发贡献

**目标读者**: 开源贡献者、新团队成员
**难度**: 中级
**格式**: 指南和最佳实践

```
development/
├── README.md                  # 贡献入口（链接到CONTRIBUTING）
├── contributing.md            # 贡献者指南（P1）
├── environment-setup.md       # 开发环境搭建（详细）
├── code-style/
│   ├── python.md              # PEP8 + 项目特定规则
│   ├── typescript.md          # TS/ESLint配置
│   ├── sql.md                 # SQL风格（命名、索引）
│   └── git-commits.md         # Conventional Commits详解
├── testing/
│   ├── unit-testing.md        # 单元测试指南
│   ├── integration-testing.md # 集成测试
│   ├── e2e-testing.md         # Playwright E2E
│   ├── coverage.md            # 覆盖率要求和解读
│   └── mocking.md             # Mock策略
├── architecture/
│   ├── overview.md            # 系统架构总览（C4模型）
│   ├── component-diagrams.md  # 组件图
│   ├── data-flow.md           # 数据流图
│   └── technology-choices.md  # 技术选型（ADR集合）
├── database/
│   ├── migrations.md          # 数据库迁移流程
│   ├── schema.md              # 当前数据库模式（自动生成？）
│   └── seeding.md             # 初始数据
├── git-workflow.md            # git-flow分支模型详解
└── code-review-checklist.md  # 代码审查清单（P0交付标准）
```

**关键文档示例** (`contributing.md`):
```markdown
# 为TraderMate做贡献

我们欢迎社区贡献！本指南帮助您快速上手。

## 行为准则
所有贡献者必须遵守 [行为准则](CODE_OF_CONDUCT.md)。

## 快速开始

### 1. Fork和Clone
```bash
git clone https://github.com/yourusername/tradermate.git
cd tradermate
```

### 2. 安装依赖
```bash
# 后端
cd tradermate
pip install -r requirements.txt

# 前端
cd ../tradermate-portal
npm install
```

### 3. 创建功能分支
```bash
git checkout -b feature/amazing-feature
```

### 4. 提交更改
```bash
git add .
git commit -m "feat: add amazing feature"
```

### 5. 推送和PR
```bash
git push origin feature/amazing-feature
# 然后在GitHub打开PR
```

## 开发流程
...

## 资源
- [代码风格](code-style/python.md)
- [测试指南](testing/unit-testing.md)
- [架构概览](architecture/overview.md)
```

---

### 5️⃣ `troubleshooting/` - 故障排查

**目标读者**: 所有角色（用户、开发、运维）
**难度**: 根据主题变化
**格式**: 症状-原因-解决方案（决策树）

```
troubleshooting/
├── README.md                  # 排查入口（决策树）
├── common-errors.md           # 高频错误列表
├── by-component/
│   ├── api.md                 # API相关故障
│   ├── frontend.md            # 前端问题
│   ├── database.md            # 数据库问题
│   ├── redis.md               # Redis问题
│   └── tushare.md             # Tushare API问题
├── by-symptom/
│   ├── connection-timeouts.md # 连接超时
│   ├── slow-performance.md    # 性能慢
│   ├── memory-errors.md       # 内存错误
│   └── import-errors.md       # 导入错误
├── logs-guide.md              # 如何查看和分析日志
├── debug-mode.md              # 开启调试模式
├── reproducing-issues.md      # 如何复现bug
└── reporting-bugs.md          # 如何提交有效的bug报告
```

**关键文档示例** (`common-errors.md`):
```markdown
# 常见错误和解决方案

> 本页面按症状分类，使用Ctrl+F搜索你的错误信息。

## 数据库连接错误

### 症状
```
sqlalchemy.exc.OperationalError: (MySQL Operational Error) 2003
Can't connect to MySQL server on 'localhost' (10061)
```

### 原因
- MySQL服务未运行
- `DATABASE_URL` 配置错误
- 网络/防火墙阻断

### 解决方案

1. **确认MySQL运行**
   ```bash
   docker ps | grep mysql
   ```
   - 如果未运行: `docker-compose up -d mysql`

2. **检查配置**
   ```bash
   echo $DATABASE_URL
   # 应是: mysql+pymysql://user:pass@host/db
   ```

3. **测试连接**
   ```bash
   mysql -u root -p -h localhost
   ```

4. **查看MySQL日志**
   ```bash
   docker logs tradermate_mysql
   ```

---

## API返回401未授权

### 症状
```json
{
  "detail": "Not authenticated"
}
```

### 原因
- 未提供 `Authorization` 头
- Token过期或无效

### 解决方案

1. **确认登录状态**
   ```bash
   curl http://localhost:8000/api/auth/me \
     -H "Authorization: Bearer YOUR_TOKEN"
   ```

2. **重新登录获取新Token**
   ```bash
   curl -X POST http://localhost:8000/api/auth/login \
     -d '{"username":"...","password":"..."}'
   ```

---

📝 **未找到你的问题？** [提交issue](https://github.com/.../issues) 或查看 [完整故障排查决策树](by-symptom/).
```

---

### 6️⃣ `learning/` - 学习路径和教程

**目标读者**: 从新手到专家各级别
**难度**: 渐进式（Beginner → Intermediate → Advanced）
**格式**: 分步教程、最佳实践

```
learning/
├── README.md                      # 学习路径总览
├── beginner-tutorials/            # 入门教程
│   ├── 01-first-backtest.md       # 第一个回测（最简）
│   ├── 02-interpret-results.md    # 如何解读回测报告
│   ├── 03-optimize-parameters.md # 参数优化基础
│   ├── 04-create-strategy.md      # 从零编写策略
│   └── 05-export-data.md          # 导出和分析数据
├── intermediate/
│   ├── custom-indicators.md       # 自定义技术指标
│   ├── advanced-backtesting.md    # 高级回测技巧（滑点、佣金）
│   ├── portfolio-management.md    # 组合管理和多策略
│   ├── risk-management.md         # 高级风险管理
│   └── performance-metrics.md     # 深度绩效分析
├── advanced/
│   ├── high-frequency.md          # 高频交易（Tick数据）
│   ├── machine-learning.md        # ML集成（如果支持）
│   ├── multi-asset.md             # 多资产类别
│   └── production-deployment.md   # 实盘部署注意事项
└── reference/
    ├── best-practices.md          # 最佳实践汇总
    ├── anti-patterns.md           # 常见错误和反模式
    ├── performance-tips.md        # 性能优化技巧
    └── resources.md               # 学习资源推荐（书籍、博客、课程）
```

**关键理念**: 每个教程是**独立的、可验证的**，用户完成后应该有明确成果。

---

### 7️⃣ `reference/` - 参考材料

**目标读者**: 需要快速查询信息的开发者
**难度**: 中级
**格式**: 压缩的参考信息（表格、列表）

```
reference/
├── README.md                      # 参考入口
├── glossary.md                    # 完整术语表（用户+技术）
├── acronyms.md                    # 缩写词表
├── api-endpoints-summary.md       # 所有端点的汇总表
├── database-schemas.md            # 数据库ER图和表说明
├── configuration-reference.md     # 所有配置选项详解
├── supported-markets.md           # 支持的市场和交易所
├── data-fields.md                 # Tushare数据字段说明
├── error-codes.md                 # 所有错误代码
├── status-codes.md                # HTTP状态码和业务状态码
├── limits-and-quotas.md           # API限制、配额、速率限制
└── version-compatibility.md       # 版本兼容性矩阵
```

---

## 导航设计

### 交叉链接策略
- 每个文档底部提供 **"下一步"** 或 **"相关链接"**
- 在关键概念首次出现时添加 `[术语链接到glossary.md]`
- 使用相对链接保持仓库可移植性

### 首页导航建议
在 `docs/README.md` 中添加：
```markdown
## 文档导航

| 类型 | 文档 |
|------|------|
| **快速开始** | [用户快速入门](../KNOWLEDGE_BASE/user-guides/getting-started.md) |
| **API开发** | [API参考](../KNOWLEDGE_BASE/api-reference/overview.md) |
| **运维部署** | [部署手册](../KNOWLEDGE_BASE/operations/deployment.md) |
| **故障排查** | [常见问题](../KNOWLEDGE_BASE/troubleshooting/common-errors.md) |
| **学习路径** | [教程总览](../KNOWLEDGE_BASE/learning/) |
```

---

## 实施计划

### Phase 1: 创建目录结构 (Week 1)
- 创建 `KNOWLEDGE_BASE/` 及其子目录（空的README导航）
- 复制/移动现有相关文档到新位置
- 更新 `docs/README.md` 添加导航

### Phase 2: 填充内容 (Week 2-4)
- 优先级P0文档直接创建在根`docs/`（需要快速交付）
- 同时将分散文档整理迁移到`KNOWLEDGE_BASE/`对应位置
- 创建交叉链接

### Phase 3: 统一和审查 (Week 5)
- 应用 `DOCUMENTATION_STANDARDS.md` 格式检查
- 确保所有链接工作
- 编写 `KNOWLEDGE_BASE/README.md` 导航页

### Phase 4: 搜索优化 (可选)
- 如果部署静态站点，添加搜索功能（Algolia DocSearch或本地搜索）
- 或者生成单一HTML（Pandoc）用于本地搜索

---

## 工具链支持

### 自动化脚本建议
```bash
# scripts/
├── build_docs.sh              # 构建静态站点（如使用MkDocs）
├── check_links.sh             # 批量检查链接
├── validate_structure.py      # 验证目录结构完整性
└── generate_api_reference.py  # 从FastAPI生成OpenAPI JSON
```

### CI验证
在GitHub Actions中添加：
- Markdown lint
- 链接检查
- 目录结构验证
- 拼写检查

---

## 维护策略

### 谁维护？
- Writer agent 是主要维护者
- 各agent对自己产生的文档负责（coder维护API、tester维护测试文档）

### 更新频率
- 每次功能发布后，**随代码更新文档**
- 每月审查死链和过时内容
- 每季度更新CHANGELOG和版本兼容性表

### 文档债务追踪
在 `TASKS.md` 中创建标记为 `@writer` 的任务：
```
- [W-001] 创建DEPLOYMENT.md
- [W-002] 迁移API文档到KNOWLEDGE_BASE/
- [W-003] 更新数据库架构图
```

---

## 成功度量

- ✅ 用户可以在10分钟内找到关键信息
- ✅ 新贡献者在1小时内了解项目并提交PR
- ✅ 运维人员可以独立完成标准部署
- ✅ 所有文档链接有效（CI自动检查）
- ✅ 文档覆盖率（功能都有对应文档）≥95%

---

## 附录：当前文档迁移映射

| 现有文档 | 新位置 | 需要修改 |
|---------|--------|----------|
| `docs/API_README.md` | `KNOWLEDGE_BASE/api-reference/overview.md` | 重命名为overview.md |
| `docs/api/*.md` | `KNOWLEDGE_BASE/api-reference/endpoints/` | 移动 |
| `docs/TESTING.md` | `development/testing/` | 移动 |
| `docs/frontend/*` | `development/frontend/` 或保留在子目录 | 评估 |
| `docs/project/ENV_VARIABLES.md` | `reference/configuration-reference.md` | 整合 |
| `docs/DATABASE_ARCHITECTURE.md` | `reference/database-schemas.md` | 重命名和补充 |
| `docs/DATA_SYNC_PLAN.md` | `operations/infrastructure.md`或保留 | 评估 |
| `tushare_stock_endpoints.md` | `reference/data-fields.md` | 整合 |

---

**下一步**: 经Main批准后，Writer开始实施Phase 1目录创建，并制定详细迁移计划。
