# Standards 文档标准

此目录包含 TraderMate 项目的文档编写标准、模板和规划。所有项目文档必须遵循这些规范，以确保一致性、可读性和可维护性。

## 内容

### 📐 核心标准

| 文档 | 描述 | 适用对象 |
|------|------|----------|
| **[DOCUMENTATION_STANDARDS.md](./DOCUMENTATION_STANDARDS.md)** | ✍️ **文档质量标准 (S.T.A.N.D. 原则)**<br>- Structure (结构清晰)<br>- Tone (语气一致)<br>- Accuracy (准确无误)<br>- Navigation (易于导航)<br>- Drafts (迭代改进) | 所有Agents (@writer, @coder, @designer, @tester, @operator) |
| **[KNOWLEDGE_BASE_STRUCTURE.md](./KNOWLEDGE_BASE_STRUCTURE.md)** | 🏗️ **长期知识库结构规划**<br>- 目录组织原则<br>- 文档分类矩阵<br>- 交叉引用规范<br>- 归档策略 | @writer (主导), 所有Agents |
| **[DOCUMENTATION_GAPS.md](./DOCUMENTATION_GAPS.md)** | 🕳️ **文档缺口清单**<br>- 识别缺失文档<br>- 优先级排序 (P0-P3)<br>- 责任人分配<br>- 完成日期 | @writer (维护) |
| **[DOCUMENTATION_ASSESSMENT.md](./DOCUMENTATION_ASSESSMENT.md)** | 📊 **文档现状评估报告**<br>- 覆盖率分析<br>- 质量问题统计<br>- 改进建议<br>- 基线指标 | @writer (生成) |
| **[DOCUMENT_CLEANUP_REPORT.md](./DOCUMENT_CLEANUP_REPORT.md)** | 🧹 **文档整理与净化报告**<br>- 整理行动记录<br>- 文件合并/删除清单<br>- 交叉引用修复<br>- 质量改进总结 | @writer (生成), 所有Agents (查阅) |

### 📄 模板库 (核心产出)

使用这些模板创建新文档，确保符合标准:

| 模板 | 用途 | 示例 |
|------|------|------|
| **[TEMPLATES/operational-runbook.md](./TEMPLATES/operational-runbook.md)** | 操作手册 (Operator日常运维) | [deployment/DEPLOYMENT_RUNBOOKS.md](../deployment/DEPLOYMENT_RUNBOOKS.md) |
| **[TEMPLATES/api-endpoint.md](./TEMPLATES/api-endpoint.md)** | API端点文档 (单个接口规格) | [development/api/strategies.md](../development/api/strategies.md) |
| **[TEMPLATES/adr.md](./TEMPLATES/adr.md)** | 架构决策记录 (技术决策) | 待创建 (建议放在 `architecture/DECISIONS/`) |
| **[TEMPLATES/troubleshooting-guide.md](./TEMPLATES/troubleshooting-guide.md)** | 故障排除指南 (常见问题) | [development/TROUBLESHOOTING.md](../development/TROUBLESHOOTING.md) |
| **[TEMPLATES/how-to-guide.md](./TEMPLATES/how-to-guide.md)** | 操作指南 (用户或开发者"如何做X") | 待创建 (如 [USER_GUIDE.md](../USER_GUIDE.md) 的章节) |
| **[TEMPLATES/learning-tutorial.md](./TEMPLATES/learning-tutorial.md)** | 学习教程 (分步教学) | 待创建 (LEARNING/系列教程) |

> 💡 **使用模板**: 复制对应模板文件到目标位置，重命名为实际文档，填充内容。模板中的注释和占位符提供了完整指导。

> 💡 **使用模板**: 复制对应模板文件到目标位置，重命名为实际文档，填充内容。模板中的注释和占位符提供了完整指导。

### 📁 归档

- **[archive/](./archive/)** - 已废弃或过时的文档
  - `cron/` - 临时cron进度报告
  - `status/` - 状态快照
  - `emergency/` - 紧急修复文档
  - `project/` - 项目历史文档

---

## 文档生命周期

### 1. 创建新文档

```bash
# 步骤:
# 1. 选择合适模板 (TEMPLATES/)
# 2. 复制到目标目录 (如 development/, deployment/)
# 3. 重命名并填写内容
# 4. 在目标目录 README.md 中添加链接
# 5. 更新相关 TASKS.md 或 DOCUMENTATION_GAPS.md
```

### 2. 评审与合并

文档提交PR前必须通过:

- [ ] **自查**: 遵循 `DOCUMENTATION_STANDARDS.md` 的 S.T.A.N.D. 原则
- [ ] **链接检查**: 所有内部链接有效 (使用 `markdown-link-check`)
- [ ] **同行评审**: 至少1名相关Agent审查 (根据内容领域)
- [ ] **更新索引**: 目标目录 README.md 已添加新文档链接
- [ ] **任务更新**: 相关任务状态已更新 (TASKS.md)

### 3. 维护与更新

- **定期审查**: 每季度审查一次文档时效性
- **重大变更**: 产品架构或API变更后2周内更新相关文档
- **小修小补**: 发现错别字或过时信息立即修复
- **版本标记**: 重要文档使用 `Last updated` 日期

### 4. 归档与删除

- **归档条件**:
  - 功能已废弃
  - 被新文档完全替代
  - 历史版本需要保留参考
- **归档流程**:
  1. 移动到 `standards/archive/<category>/`
  2. 在原始位置保留 `README.md` 指向归档位置
  3. 更新相关链接 (如需要)
- **删除**: 一般文档不删除，仅归档。敏感信息例外 (需彻底删除)

---

## 质量检查清单 (Linter)

在CI或本地运行文档检查:

```bash
# 1. Markdown 格式检查 (markdownlint)
markdownlint docs/**/*.md

# 2. 链接检查 (markdown-link-check)
npx markdown-link-check docs/**/*.md

# 3. 拼写检查 (codespell)
codespell docs/**/*.md

# 4. 死链检查 (lychee)
lychee docs/**/*.md
```

**CI集成**: 在 `.github/workflows/` 添加文档检查工作流，失败则阻止合并。

---

## 相关标准

### 命名规范

- 文件名: `kebab-case` (小写字母 + 连字符)
  - ✅ `getting-started.md`
  - ❌ `GettingStarted.md` (大写)
  - ❌ `get_started.md` (下划线)

- 标题层级: 最多 `###` (H3)，避免过深嵌套

### 引用规范

- **内部链接**: 使用相对路径
  ```markdown
  `../architecture/API_CONTRACT_V1.yaml`
  [部署手册](./DEPLOYMENT_RUNBOOKS.md)
  ```

- **外部链接**: 完整URL
  ```markdown
  [FastAPI官方文档](https://fastapi.tiangolo.com/)
  ```

- **锚点链接**: 同一文档内跳转
  ```markdown
  [安装步骤](#安装步骤)
  ```

### 代码块规范

- 指定语言以实现语法高亮
  ````markdown
  ```python
  def hello():
      print("Hello")
  ```
  ````

- 包含注释说明关键步骤
- 输出示例使用 `expected` 或 `text` 类型

---

## 文档结构最佳实践

### 标准文档模板 (通用)

```markdown
# 标题

> 一句话摘要 (可选)

## 目录 (可选，长文档建议)

- [TOC]

## 概述

背景、目的、适用范围。

## 主体内容

按章节组织，逻辑清晰。

## 快速参考

表格、速查表、常见问题。

## 相关链接

- [相关文档](链接)
- [外部资源](链接)

---

**维护者**: @role  
**最后更新**: YYYY-MM-DD
```

### 不同文档类型的特殊要求

| 类型 | 必需章节 | 特殊要求 | 示例 |
|------|----------|----------|------|
| 操作手册 (Runbook) | 前置条件、操作步骤、验证、回滚 | 强调确定性 ("输入X，输出Y") | `deployment/DEPLOYMENT_RUNBOOKS.md` |
| API文档 | 端点信息、请求/响应、错误码、数据模型 | 使用 OpenAPI/Swagger 注解 | `development/api/` |
| 架构决策 (ADR) | 上下文、决策、替代方案、后果 | 记录决策过程和权衡 | `TEMPLATES/adr.md` |
| 故障排除 | 症状→诊断→修复、日志查看 | 提供可直接复制的命令 | `development/TROUBLESHOOTING.md` |
| 测试文档 | 测试用例、覆盖率、数据管理 | 包含可运行的测试代码 | `testing/TESTING.md` |

---

## 更新与反馈

### 提交问题

发现文档问题?

1. 检查是否已有记录: `grep -r "关键词" docs/`
2. 如未记录，创建 Issue 或直接提交PR
3. PR描述清晰说明问题和建议修改

### 讨论与改进

文档标准本身也在迭代:

- 提出改进建议 → 在 `project-management/TASKS.md` 创建任务
- 重大变更 → 与 @writer 和 @main 讨论
- 模板更新 → 通知所有Agents

---

## 附录: 常用命令

### 文档检查

```bash
# 全局链接检查 (需node环境)
find docs/ -name "*.md" -exec npx markdown-link-check {} \;

# Markdown格式检查
markdownlint "docs/**/*.md"

# 搜索文档内容
grep -r "API_CONTRACT" docs/
```

### 文档生成 (未来)

- 使用 `mkdocs` 或 `docsify` 生成静态网站
- 自动API文档: 从 FastAPI OpenAPI schema 生成
- 版本化发布: 使用 Git tags 对应文档版本

---

**维护者**: @writer  
**文档版本**: 1.0 (2026-03-03)  
**生效日期**: 2026-03-03  
**下次审查**: 2026-06-03 或遇重大变更时