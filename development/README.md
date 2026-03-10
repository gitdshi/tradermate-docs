# Development 开发指南

此目录包含 TraderMate 开发和测试相关的完整指南。

## 内容

### 🚀 快速开始
- **[ENVIRONMENT_SETUP.md](./ENVIRONMENT_SETUP.md)** - ⭐ **开发环境配置完整指南** (必读)
  - Docker开发模式 (推荐)
  - 本地原生安装
  - 数据库初始化
  - 环境验证
  - 故障排除

### 📖 核心文档
- **[API_README.md](./API_README.md)** - API概览、认证、快速示例
- **[GETTING_STARTED.md](./GETTING_STARTED.md)** - 5分钟快速入门 (新贡献者首选)
- **[ENV_VARIABLES_REFERENCE.md](./ENV_VARIABLES_REFERENCE.md)** - 所有环境变量详细说明
- **[TROUBLESHOOTING.md](./TROUBLESHOOTING.md)** - 开发环境常见问题与解决方案

### 💻 技术文档
- **[api/](./api/)** - API端点详细文档 (按功能模块)
  - `auth.md` - 认证授权
  - `strategies.md` - 策略管理
  - `backtest.md` - 回测系统
  - `data.md` - 市场数据
  - `queue.md` - 任务队列
  - `optimization.md` - 参数优化
- **[frontend/](./frontend/)** - 前端技术文档
  - `FRONTEND_README.md` - 前端架构、技术栈、项目结构
  - `E2E_README.md` - Playwright E2E测试指南
  - `TEST_SUMMARY.md` - 测试总结报告
  - `PHASE7_FEATURES.md` - Phase 7特性规划

---

## 开始开发

1. **配置环境**: 阅读 [ENVIRONMENT_SETUP.md](./ENVIRONMENT_SETUP.md)，选择Docker或本地方式启动开发环境
2. **了解API**: 查看 [API_README.md](./API_README.md) 和 [api/](./api/) 详细端点
3. **前端开发**: 阅读 [frontend/FRONTEND_README.md](./frontend/FRONTEND_README.md)
4. **遇到问题**: 参考 [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)

---

## 相关链接

- [部署运维](../deployment/README.md) - 生产环境部署和运维
- [架构设计](../architecture/README.md) - 系统架构和技术决策
- API契约与架构: `../architecture/API_CONTRACT_V1.yaml`（OpenAPI） + `../architecture/SYSTEM_ARCHITECTURE.md`
- [测试文档](../testing/README.md) - 测试策略和验证报告
- [项目任务](../project-management/TASKS.md) - 开发任务跟踪

---

**维护者**: @coder, @tester  
**最后更新**: 2026-03-03