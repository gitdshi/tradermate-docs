# Architecture 架构文档

此目录包含 TraderMate 系统的架构设计、技术决策和代码质量评估文档。

---

## 📋 文档列表

### 系统架构

| 文档 | 描述 | 维护者 |
|------|------|--------|
| **[DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md)** | 双数据库架构设计 (tushare/vnpy)<br>- 数据流向<br>- 表结构设计<br>- 同步策略 | @coder |
| **[DATA_SYNC_PLAN.md](./DATA_SYNC_PLAN.md)** | 数据同步守护进程详细设计<br>- 同步流程<br>- 错误处理<br>- 监控指标 | @coder |

### 代码评估与技术债务

| 文档 | 描述 | 维护者 |
|------|------|--------|
| **[CODE_ANALYSIS_REPORT.md](./CODE_ANALYSIS_REPORT.md)** | C-002 代码结构熟悉报告<br>- 后端/前端架构分析<br>- 设计模式评估<br>- 改进建议 | @coder |
| **[TECH_DEBT_INVENTORY.md](./TECH_DEBT_INVENTORY.md)** | 技术债务清单<br>- 分类：高/中/低优先级<br>- 修复建议<br>- 影响评估 | @coder |
| **[PHASE7_IMPLEMENTATION_PLAN.md](./PHASE7_IMPLEMENTATION_PLAN.md)** | Phase 7功能实施计划<br>- 工作量评估 (12.5人日)<br>- 分阶段策略 (V1/V2)<br>- API实现清单<br>- 技术风险 | @coder |

### 代码审查

| 文档 | 描述 | 维护者 |
|------|------|--------|
| **[CODE_REVIEWS.md](./CODE_REVIEWS.md)** | 代码审查标准和流程<br>- 审查清单<br>- 质量门禁<br>- 最佳实践 | @coder, @operator |
| **[CODE_REVIEW_REPORT_*.md](./CODE_REVIEW_REPORT_2026-03-03_INFRASTRUCTURE.md)** | 具体审查报告 (按日期)<br>- 发现问题清单<br>- 修复建议 | @operator |

---

## 🎯 按场景阅读

### "我想理解系统整体架构"
1. 阅读 [DATABASE_ARCHITECTURE.md](./DATABASE_ARCHITECTURE.md) 了解数据层
2. 阅读 [DATA_SYNC_PLAN.md](./DATA_SYNC_PLAN.md) 了解数据流
3. 阅读 [CODE_ANALYSIS_REPORT.md](./CODE_ANALYSIS_REPORT.md) 了解代码组织

### "评估技术债务"
1. 阅读 [TECH_DEBT_INVENTORY.md](./TECH_DEBT_INVENTORY.md) 查看债务清单
2. 参考 [PHASE7_IMPLEMENTATION_PLAN.md](./PHASE7_IMPLEMENTATION_PLAN.md) 了解债务影响

### "准备开始Phase 7开发"
1. 通读 [PHASE7_IMPLEMENTATION_PLAN.md](./PHASE7_IMPLEMENTATION_PLAN.md) 完整计划
2. 参考 [CODE_ANALYSIS_REPORT.md](./CODE_ANALYSIS_REPORT.md) 的架构建议
3. 查看 `API_CONTRACT_V1.yaml` 了解接口设计（OpenAPI 3.0）

### "进行代码审查"
1. 阅读 [CODE_REVIEWS.md](./CODE_REVIEWS.md) 审查标准
2. 查看最新的 `CODE_REVIEW_REPORT_*.md` 了解历史问题

---

## 📊 文档维护

| 文档 | 更新频率 | 触发条件 |
|------|----------|----------|
| CODE_ANALYSIS_REPORT.md | 一次性的架构评估 | 架构重大变更时 |
| TECH_DEBT_INVENTORY.md | 每月 | 每次迭代后更新 |
| PHASE7_IMPLEMENTATION_PLAN.md | 按Phase | Phase 7启动/完成时 |
| CODE_REVIEWS.md | 持续 | 审查流程变更时 |
| CODE_REVIEW_REPORT_*.md | 每次审查 | 代码审查完成后 |

---

## 🔗 相关文档

- UI/UX 设计资料：已从 public docs repo 收敛删除（如需保留请放到独立设计仓库或私有空间）
- [开发指南](../development/README.md) - 环境设置和API
- [项目管理](../project-management/TASKS.md) - 任务跟踪

---

## 📝 架构决策记录 (ADRs)

重要技术决策应记录为 **ADR** (Architecture Decision Record)，模板见 `../standards/TEMPLATES/adr.md`。

建议创建的ADRs:
- ADR-001: 选择 FastAPI 作为后端框架
- ADR-002: 选择 React + Vite 作为前端框架
- ADR-003: 双数据库架构 (tushare/vnpy)
- ADR-004: 使用 Redis + RQ 进行异步任务
- ADR-005: 多Agent协作工作流
- ADR-006: 数据库选型 (MySQL)
- ADR-007: React 19 → React 18 降级决策 (如有)

ADR应放置在 `architecture/DECISIONS/` 目录（待创建）。

---

** Maintainer**: @coder (主要), @operator (代码审查相关)
** last Updated**: 2026-03-03
** 相关标准**: [DOCUMENTATION_STANDARDS.md](../../standards/DOCUMENTATION_STANDARDS.md)
