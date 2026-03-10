# Design 设计文档

此目录包含 TraderMate 项目的用户界面设计、视觉规范和前后端接口契约。所有设计决策和UI规格都在这里记录。

## 内容

### 🎨 视觉设计系统

| 文档 | 描述 | 状态 |
|------|------|------|
| **[UI_DESIGN_SYSTEM.md](./UI_DESIGN_SYSTEM.md)** | 视觉设计系统<br>- 色彩系统 (主色/辅助色/灰度)<br>- 字体排印 (字号/行高/字重)<br>- 间距规范 (spacing scale)<br>- 阴影/圆角/动效 | ✅ 完整 |
| **[COMPONENT_SPECS.md](./COMPONENT_SPECS.md)** | UI组件规格说明<br>- 按钮、输入框、表格、模态框<br>- 状态 (默认/悬停/禁用/错误)<br>- 尺寸变体 (sm/md/lg)<br>- 无障碍要求 | ✅ 完整 |

### 📐 页面设计

| 文档 | 描述 | 页面覆盖 |
|------|------|----------|
| **[WIREFRAMES.md](./WIREFRAMES.md)** | 页面线框图和布局<br>- 策略编辑器 (Strategy Editor)<br>- 回测界面 (Backtest Dashboard)<br>- 分析仪表板 (Analytics)<br>- 投资组合 (Portfolio)<br>- 优化界面 (Optimization) | 📱 响应式 |

### 🔄 前后端协作

| 文档 | 描述 | 更新频率 |
|------|------|----------|
| **[API_CONTRACT.md](./API_CONTRACT.md)** | ⚡ **前后端API契约** (单点真相)<br>- 端点定义 (Method + Path)<br>- 请求/响应 Schema<br>- 错误码和消息<br>- 认证要求 | 🔄 随API实现同步更新 |

---

## 设计工作流

### 1. 新功能设计流程

```
需求 → 线框图 (WIREFRAMES.md) → 组件规格 (COMPONENT_SPECS.md) → API契约 (API_CONTRACT.md) → 实现
```

### 2. 设计系统演进

- 新增设计令牌 → 更新 `UI_DESIGN_SYSTEM.md`
- 新增组件 → 补充 `COMPONENT_SPECS.md`
- 重大变更 (如暗色模式) → 创建独立文档 `DARK_MODE_SPEC.md` 并关联

### 3. API开发协作

**前端 ↔ 后端协作协议**:

1. **由后端 Coder 发起**: 需要新API时，首先更新 `API_CONTRACT.md` 定义接口
2. **前端 Designer 审查**: 确认API满足UI需求，字段、错误处理完备
3. **后端实现**: 按契约实现API端点
4. **前端集成**: 基于契约开发UI，无需等待后端完成 (可mock)
5. **集成测试**: 对照契约验证API实际行为

⚠️ **契约变更纪律**:
- API一旦确定，禁止破坏性变更 (删除字段、改类型)
- 新增字段 ✅ 允许
- 废弃字段 ⚠️ 标记 `deprecated`，保留至少1个迭代
- 重大变更需 Main 批准并通知所有相关方

---

## 快速查找

| 我需要... | 应该查看 |
|-----------|----------|
| 按钮的默认颜色和圆角 | `UI_DESIGN_SYSTEM.md` |
| 表格组件的所有状态样式 | `COMPONENT_SPECS.md` |
| 策略编辑器的页面布局 | `WIREFRAMES.md` |
| 登录API的请求/响应格式 | `API_CONTRACT.md` |
| Portfolio API需要哪些字段 | `API_CONTRACT.md` (搜索 "portfolio") |
| 设计系统的颜色变量名 | `UI_DESIGN_SYSTEM.md` → 颜色系统 |

---

## 与开发文档集成

设计文档与开发文档紧密关联：

- **API契约实现**: 开发者在 `development/api/` 中编写详细端点文档，必须**引用本目录的API_CONTRACT.md** 作为设计依据
- **组件实现**: 前端开发在 `tradermate-portal/src/components/` 实现组件时，需遵循 `COMPONENT_SPECS.md` 的规格
- **主题定制**:  Tailwind CSS配置应映射 `UI_DESIGN_SYSTEM.md` 的设计令牌

---

## 相关链接

- **[开发环境设置](../development/ENVIRONMENT_SETUP.md)** - 配置开发环境
- **[前端技术文档](../development/frontend/FRONTEND_README.md)** - React/Vite架构
- **[API端点文档](../development/api/)** - 详细API实现文档 (按契约编写)
- **[项目任务](../project-management/TASKS.md)** - 设计相关任务

---

## 贡献指南

### 修改设计文档

1. 确保更改已与 Coder 和 Designer 团队共识
2. 更新对应文档，保持与实现一致
3. 如果影响API，同时更新 `API_CONTRACT.md` 和相关实现文档
4. 提交PR时，@designer 和 @coder 必须批准

### 添加新文档

- 新页面设计 → 补充到 `WIREFRAMES.md`
- 新组件 → 补充到 `COMPONENT_SPECS.md`
- 重大设计决策 → 可创建独立文档（如 `ACCESSIBILITY_GUIDELINES.md`）并更新本README

---

**维护者**: @designer  
**相关Agent**: @coder (implementation), @tester (validation)  
**最后更新**: 2026-03-03  
**下次审查**: 2026-04-03 或 Phase 7 UI评审后