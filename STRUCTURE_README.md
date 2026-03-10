# 📚 TraderMate 文档结构规范

**版本**: 1.0
**生效日期**: 2026-03-03
**维护者**: Writer (Mia)
**适用范围**: 所有项目文档（`docs/` 目录）

---

## 🎯 概述

本文档定义 TraderMate 项目文档的**标准目录结构、组织原则和维护规则**。所有贡献者必须遵循此规范，以确保文档易于发现、维护和扩展。

### 核心原则

1. **单一职责** - 每个目录有清晰的主题范围
2. **受众导向** - 按用户角色分组文档
3. **可发现性** - 新成员能在30秒内找到所需信息
4. **可扩展** - 新文档有明确归属，无需频繁调整结构
5. **唯一性** - 关键文件（如 TASKS.md）只存在于一个位置

---

## 📂 标准目录结构（最终版）

```
docs/
├── README.md                          # 📌 主入口（导航中心）
├── STRUCTURE_README.md               # 📌 本文件（结构规范）
│
├── design/                           # 🎨 设计文档 (Designer 负责)
│   ├── UI_DESIGN_SYSTEM.md          # 视觉设计系统
│   ├── WIREFRAMES.md                # 页面线框图
│   ├── COMPONENT_SPECS.md           # UI组件规格
│   ├── API_CONTRACT.md              # 前后端API契约
│   └── README.md                    # 设计文档导航
│
├── development/                      # 💻 开发指南 (Coder/Tester 负责)
│   ├── LOCAL_DEVELOPMENT_SETUP.md  # 本地环境搭建（完整版）
│   ├── DOCKER_DEVELOPMENT.md       # Docker开发模式
│   ├── ENV_VARIABLES_REFERENCE.md  # 环境变量参考
│   ├── TROUBLESHOOTING.md          # 开发环境故障排查
│   ├── API_README.md               # API概览和快速开始
│   ├── GETTING_STARTED.md          # 5分钟快速入门
│   │
│   ├── api/                        # API端点详细文档
│   │   ├── auth.md
│   │   ├── strategies.md
│   │   ├── backtest.md
│   │   ├── data.md
│   │   ├── queue.md
│   │   └── optimization.md
│   │
│   ├── frontend/                   # 前端技术文档
│   │   ├── FRONTEND_README.md     # 前端架构和技术栈
│   │   ├── E2E_README.md          # Playwright E2E测试
│   │   ├── TEST_SUMMARY.md        # 测试总结报告
│   │   └── PHASE7_FEATURES.md     # Phase 7特性规划
│   │
│   └── README.md                   # 开发指南导航
│
├── deployment/                      # 🚀 部署与运维 (Operator 负责)
│   ├── DEPLOYMENT_RUNBOOKS.md      # 部署流程和操作手册
│   ├── MONITORING_CONFIG.md        # 监控配置（Prometheus/Grafana）
│   ├── SECURITY_CHECKLIST.md       # 安全检查清单
│   ├── BACKUP_RECOVERY_PLAN.md     # 备份与灾难恢复
│   ├── INFRASTRUCTURE_DIAGRAM.md   # 基础设施架构图
│   ├── INFRASTRUCTURE_STATUS.md    # 基础设施状态报告
│   ├── docker-compose.prod.yml     # 生产环境Docker配置
│   └── README.md                   # 部署导航
│
├── architecture/                    # 🏗️ 架构与技术债务 (Coder 负责)
│   ├── DATABASE_ARCHITECTURE.md    # 双数据库架构设计
│   ├── DATA_SYNC_PLAN.md           # 数据同步守护进程
│   ├── CODE_ANALYSIS_REPORT.md     # 代码结构审查（C-002）
│   ├── TECH_DEBT_INVENTORY.md      # 技术债务清单
│   ├── PHASE7_IMPLEMENTATION_PLAN.md  # Phase 7实施计划
│   ├── CODE_REVIEWS.md             # 代码审查标准
│   ├── CODE_REVIEW_REPORT_*.md     # 代码审查报告（按日期）
│   └── README.md                   # 架构导航
│
├── testing/                        # 🧪 测试文档 (Tester 负责)
│   ├── TESTING.md                  # 测试概述和命令
│   ├── TEST_COVERAGE_REPORT.md     # 测试覆盖率报告
│   ├── validation-reports/         # 环境验证报告（按日期）
│   │   ├── VALIDATION_*.md
│   │   └── ...
│   └── README.md                   # 测试导航
│
├── standards/                      # 📐 文档标准 (Writer 负责)
│   ├── DOCUMENTATION_STANDARDS.md     # 文档撰写标准（S.T.A.N.D.）
│   ├── KNOWLEDGE_BASE_STRUCTURE.md   # 知识库长期结构规划
│   ├── DOCUMENTATION_GAPS.md         # 文档缺口清单（优先级）
│   ├── DOCUMENTATION_ASSESSMENT.md   # 文档现状评估报告
│   ├── DOCUMENTATION_TODO.md         # 详细执行计划
│   ├── OUTPUT/                         # Writer工作产出（临时）
│   │   ├── FINAL_REPORT.md
│   │   ├── TEMPLATES/
│   │   └── ...
│   ├── TEMPLATES/                     # 标准模板库（最终位置）
│   │   ├── how-to-guide.md
│   │   ├── troubleshooting-guide.md
│   │   ├── api-endpoint.md
│   │   ├── operational-runbook.md
│   │   ├── learning-tutorial.md
│   │   └── adr.md
│   ├── archive/                       # 已废弃文档存档
│   │   └── project/
│   └── README.md                      # 标准导航
│
├── project-management/             # 📋 项目管理 (Main 负责)
│   ├── TASKS.md                    # ⚠️ **任务跟踪唯一位置**
│   ├── PROJECT_MEMORY.md           # ⚠️ **项目记忆唯一位置**
│   ├── WORKFLOW.md                 # 多Agent工作流
│   ├── README.md                   # 项目管理导航
│   └── (其他Main产生的文档)
│
└── USER_GUIDE.md                  # 🎯 P0用户文档（根目录，优先创建）
                                    # 将来可能移动到 user-guides/ 或保留根目录
```

---

## 🔑 关键规则

### 1. 关键文件唯一性规则

⚠️ **最重要规则**：以下文件**禁止**出现在 `docs/` 根目录之外：

| 文件 | 唯一位置 | 用途 | 维护者 |
|------|----------|------|--------|
| `TASKS.md` | `docs/project-management/TASKS.md` | 所有任务跟踪 | @main |
| `PROJECT_MEMORY.md` | `docs/project-management/PROJECT_MEMORY.md` | 项目记忆和决策 | @main |

**违反后果**：
- 任务分散会导致跟踪失败
- 决策记录不一致
- CI检查可能失败（未来）

---

### 2. 新文档放置规则

创建新文档时，按**文档类型**选择目录：

| 文档类型 | 目标目录 | 示例 |
|---------|---------|------|
| UI设计、线框图、设计系统 | `design/` | UI_DESIGN_SYSTEM.md |
| 开发环境、API、前端技术 | `development/` | LOCAL_DEVELOPMENT_SETUP.md |
| 部署、监控、运维操作 | `deployment/` | DEPLOYMENT_RUNBOOKS.md |
| 架构图、技术债务、代码分析 | `architecture/` | CODE_ANALYSIS_REPORT.md |
| 测试策略、验证报告 | `testing/` | TESTING.md, validation-reports/ |
| 文档标准、模板、规划 | `standards/` | DOCUMENTATION_STANDARDS.md |
| 任务、工作流、项目记忆 | `project-management/` | TASKS.md, WORKFLOW.md |
| 用户操作指南（P0） | `USER_GUIDE.md` (根目录) | USER_GUIDE.md |
| 开发指南（参考） | 可选 `user-guides/` (未来) | DEVELOPER_GUIDE.md |

**如果 unsure**：先问 Writer (@writer) 或 Main (@main)

---

### 3. 引用规范（交叉链接）

#### 内部相对链接

使用**相对路径**而非绝对路径，确保文档移动后仍可工作：

```markdown
## 相关链接

- [任务跟踪](project-management/TASKS.md)
- [项目记忆](project-management/PROJECT_MEMORY.md)
- [API概览](development/API_README.md)
- [部署指南](deployment/DEPLOYMENT_RUNBOOKS.md)
- [设计契约](../design/API_CONTRACT.md)  # 从子目录往上
```

#### 外部链接

```markdown
[FastAPI官方文档](https://fastapi.tiangolo.com/)
```

#### 锚点链接（同一文档内）

```markdown
[Jump to Installation](#installation)
```

---

### 4. 文件名规范

- 使用 **kebab-case** (小写字母 + 连字符)
  - ✅ `getting-started.md`
  - ✅ `test-coverage-report.md`
  - ❌ `GettingStarted.md` (大写)
  - ❌ `test_coverage_report.md` (下划线)
  - ❌ `Test Coverage Report.md` (空格)

- 保持简洁但描述性
  - ✅ `LOCAL_DEVELOPMENT_SETUP.md` (大写可接受，已是项目标准)
  - ✅ `CODE_ANALYSIS_REPORT.md`
  - 避免: `MyVeryLongDocumentNameThatIsTooLong.md`

---

### 5. 目录层级限制

- **最大深度**: 3层（`a/b/c/d.md` 是极限）
- 过深会增加导航复杂度
- 如果需要更深，考虑重构或使用嵌套README导航

**示例 (可接受)**:
```
docs/
└── development/
    └── frontend/
        └── COMPONENT_LIBRARY.md  # 深度3，OK
```

**示例 (太深)**:
```
docs/
└── development/
    └── frontend/
        └── components/
            └── buttons/
                └── PRIMARY_BUTTON.md  # 深度4，避免
```

---

### 6. README.md 导航要求

每个子目录**必须**包含 `README.md`，提供：
- 目录目的和受众
- 内容列表（带链接）
- 相关链接（其他目录）
- 维护者信息

**示例模板** (`docs/design/README.md`):
```markdown
# Design 设计文档

此目录包含 TraderMate 项目的设计相关文档。

## 内容

- **UI_DESIGN_SYSTEM.md** - 视觉设计系统（色彩、字体、组件）
- **WIREFRAMES.md** - 页面线框图
- **COMPONENT_SPECS.md** - UI组件规格

## 开始

1. 阅读 UI_DESIGN_SYSTEM.md 了解设计令牌
2. 查看 WIREFRAMES.md 理解页面布局
3. 参考 COMPONENT_SPECS.md 实现组件

## 相关

- [开发指南](../development/README.md)
- [项目管理](../project-management/README.md)

---

**维护者**: @designer
**最后更新**: 2026-03-03
```

---

## 🔍 违反结构的常见场景

### ❌ 禁止: 在根目录创建新的 TASKS.md

**错误**:
```bash
touch docs/TASKS_v2.md  # 错！会混淆
```

**正确**:
- 如需新任务跟踪文件，添加到 `project-management/` 并命名 `TASKS_<SCOPE>.md`
- 或扩展现有 `TASKS.md`

---

### ❌ 禁止: 在根目录创建 PROJECT_MEMORY_v2.md

**错误**:
```bash
# 想把某个sub-project的记忆单独存放
touch docs/PROJECT_MEMORY_PHASE7.md  # 错！
```

**正确**:
- 所有项目级决策和记忆统一到 `project-management/PROJECT_MEMORY.md`
- 如需分区，使用章节标题：
  ```markdown
  ## Phase 7 决策
  ### 2026-03-03: 选择分阶段上线
  ...
  ```

---

### ❌ 禁止: 将开发文档放到 deployment/

**错误**:
```bash
# 把 LOCAL_DEVELOPMENT_SETUP.md 放到 deployment/
# 因为"都是基础设施"
```

**正确**:
- 开发环境设置 → `development/`
- 生产部署 → `deployment/`
- 两者用途不同，受众不同（开发 vs 运维）

---

### ❌ 禁止: 直接在根目录堆砌文档

**错误**:
```
docs/
├── MY_SPEC.md
├── ANOTHER_DOC.md
├── NOTES.md
└── TODO.md
```

**正确**:
根据文档类型移动到对应子目录，或创建新目录（需经Main批准）

---

## 📋 目录职责矩阵

| 目录 | 负责Agent | 文档类型 | 受众 |
|------|----------|----------|------|
| `design/` | @designer | UI/UX设计、API契约 | 设计师、前端开发 |
| `development/` | @coder, @tester | 开发环境、API、前端技术 | 开发者、测试 |
| `deployment/` | @operator | 部署、监控、安全、运维 | DevOps、运维 |
| `architecture/` | @coder | 系统架构、技术债务、代码审查 | 开发者、架构师 |
| `testing/` | @tester | 测试策略、验证报告、覆盖率 | 测试、开发 |
| `standards/` | @writer | 文档标准、模板、规划 | 所有agents |
| `project-management/` | @main | 任务、工作流、项目记忆 | 所有agents |
| `docs/` 根目录 | - | 用户文档（P0）、导航文件 | 用户、维护者 |

---

## 🔄 文档移动流程

如果需要移动现有文档：

1. **检查引用**: `grep -r "文件名.md" docs/` 找到所有引用
2. **更新引用**: 修改所有内部链接为新的相对路径
3. **移动文件**: `git mv old/path.md new/path.md`
4. **更新TASKS.md**: 如果移动影响了任务产出，更新任务描述
5. **提交**: `git commit -m "docs: 移动X文档到Y目录（遵循结构规范）"`
6. **通知**: 通过 `sessions_send` 通知可能受影响的其他agents

---

## 🚀 新文档创建检查清单

创建新文档前，按此流程：

```
[ ] 1. 确定文档类型（用户指南？API？部署？）
[ ] 2. 选择目标目录（参考"新文档放置规则"）
[ ] 3. 检查目录是否已有类似文档（避免重复）
[ ] 4. 选择合适模板（从 standards/TEMPLATES/ 复制）
[ ] 5. 编写内容（遵循 DOCUMENTATION_STANDARDS.md）
[ ] 6. 添加导航链接：
    - [ ] 链接到相关文档
    - [ ] 在目录README.md中添加条目（如果重要）
[ ] 7. 更新 TASKS.md 或相关任务状态
[ ] 8. 自检：
    - [ ] 文件名kebab-case
    - [ ] 所有内部链接使用相对路径
    - [ ] 无敏感信息（密码、密钥）
    - [ ] 代码示例可运行
[ ] 9. 提交PR（需要对应Agent审查）
```

---

## 📞 维护与变更

### 谁维护此结构？

- **Writer (Mia)** 是文档结构的主要维护者
- 结构变更需经 **Main** 和 **Writer** 共同批准
- 任何agent可提议变更，通过 `sessions_send` 讨论

### 何时需要调整结构？

- 新增文档类型，现有目录无法容纳
- 发现职责重叠或混淆
- 用户反馈难以找到文档
- 新增major功能模块（如Phase 7需要新目录？）

### 变更流程

1. 提议变更（邮件/Slack/sessions_send）
2. 讨论并达成共识
3. 更新 `docs/STRUCTURE_README.md`
4. 移动文档（如果需要）
5. 通知所有agents
6. 更新CI规则（如果自动化）

---

## 🧪 验证和CI

### 手动检查

定期（每月）运行：
```bash
# 1. 检查是否有多个TASKS.md
find docs/ -name "TASKS.md" -type f

# 2. 检查是否有多个PROJECT_MEMORY.md
find docs/ -name "PROJECT_MEMORY.md" -type f

# 3. 检查孤立文件（不在标准目录的.md）
find docs/ -name "*.md" -type f | grep -vE "(design|development|deployment|architecture|testing|standards|project-management|README|STRUCTURE)" | head
```

### 未来CI规则（待配置）

在GitHub Actions中添加：
- ✅ 检查 TASKS.md 唯一性
- ✅ 检查 PROJECT_MEMORY.md 唯一性
- ✅ 检查新文件是否在标准目录
- ✅ 检查相对链接有效性（markdown-link-check）
- ✅ 检查文件命名规范（kebab-case）

---

## 📚 示例场景

### 场景1: Coder需要添加新的API端点文档

```
question: my-new-api.md 应该放哪里？

answer:
1. API端点文档 → development/api/
2. 文件名: api/my-new-endpoint.md (如果本身在api/目录)
   或: development/api/my-new-endpoint.md
3. 更新 development/API_README.md 添加链接
```

---

### 场景2: Designer需要添加新的设计规范

```
question: 新的设计token定义，放哪里？

answer:
1. 设计规范 → design/
2. 文件名:
   - 如果是补充现有UI_DESIGN_SYSTEM.md，直接编辑该文件
   - 如果是新主题（如"暗色模式规范"），创建 design/DARK_MODE_SPEC.md
3. 更新 design/README.md
```

---

### 场景3: Operator需要添加新的监控配置

```
question: 新服务的监控配置，放哪里？

answer:
1. 监控配置 → deployment/
2. 文件名:
   - 如果是补充现有MONITORING_CONFIG.md，直接编辑
   - 如果是独立配置（如"Grafana仪表板配置"），创建 deployment/GRAFANA_DASHBOARDS.md
3. 更新 deployment/README.md
```

---

### 场景4: Writer需要添加文档标准更新记录

```
question: 标准变更文档，放哪里？

answer:
1. 文档标准相关 → standards/
2. 文件名: 如 standards/DOCUMENTATION_STANDARDS_CHANGELOG.md
   或直接更新 DOCUMENTATION_STANDARDS.md 的更新历史
3. standards/README.md 应引用标准文件本身（无需改动）
```

---

## 🗺️ 导航地图

快速查找文档：

```
我需要...                    查看
---------                    --------------------
✅ 如何开发？                development/README.md
✅ 如何部署？                deployment/README.md
✅ 系统架构？                architecture/README.md
✅ 设计规范？                design/README.md
✅ API文档？                 development/api/ 或 development/API_README.md
✅ 测试报告？                testing/validation-reports/
✅ 当前任务？                project-management/TASKS.md
✅ 项目决策？                project-management/PROJECT_MEMORY.md
✅ 写作规范？                standards/DOCUMENTATION_STANDARDS.md
✅ 文档结构？                STRUCTURE_README.md (本文件)
```

---

## 📊 当前状态（2026-03-03）

✅ **已完成整理**：
- ✅ 创建了8个子目录（design, development, deployment, architecture, testing, standards, project-management）
- ✅ 移动了30+文档到对应目录
- ✅ 为每个目录创建了README.md导航
- ✅ 更新了根目录 README.md 快速链接
- ✅ 修复了内部引用（API_CONTRACT.md路径）
- ✅ 项目关键文件已归位（TASKS.md, PROJECT_MEMORY.md）

⚠️ **待处理**：
- ⏳ 可能需要创建 `validation-reports/` 目录（如果存在报告文件）
- ⏳ `USER_GUIDE.md` 待创建（P0文档，Writer负责）
- ⏳ 可能将 `OUTPUT/` 和 `TEMPLATES/` 合并到 `standards/TEMPLATES/`

---

## 🔗 相关文档

- [DOCUMENTATION_STANDARDS.md](./standards/DOCUMENTATION_STANDARDS.md) - 文档编写标准
- [KNOWLEDGE_BASE_STRUCTURE.md](./standards/KNOWLEDGE_BASE_STRUCTURE.md) - 长期知识库规划
- [DOCUMENTATION_GAPS.md](./standards/DOCUMENTATION_GAPS.md) - 文档缺口清单
- [WORKFLOW.md](./project-management/WORKFLOW.md) - 多Agent工作流

---

**生效**: 2026-03-03 起所有新文档必须遵循此结构
**审查**: 每月审查一次，确保结构仍然有效
**修订**: 需要Main和Writer共同批准

---

_保持文档整洁，人人有责！📚✨_
