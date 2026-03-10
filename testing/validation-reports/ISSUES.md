# TraderMate 环境验证发现的问题清单

**日期**: 2026-03-02
**验证人**: Sarah (Tester)
**任务**: T-001

---

## 阻塞性问题 (Blocking)

### B001 - 数据库服务完全不可用

**模块**: Infrastructure
**严重性**: 🔴 Critical - Blocker
**状态**: Open

**描述**:
项目依赖MySQL和Redis数据库，但当前环境中：
- 远程MySQL (10.0.0.73:3306) 网络不可达
- Docker/Docker Compose未安装
- 本地MySQL/Redis服务未运行
- 无sudo权限安装系统包

**影响**:
- 后端API无法启动
- 所有集成测试无法执行
- 开发者无法进行任何需要数据库的工作
- 整个开发进程阻塞

**重现步骤**:
1. 尝试连接数据库: `python -c "import pymysql; pymysql.connect(host='10.0.0.73', port=3306, user='root', password='tradermate01@A')"`
2. 尝试启动docker: `docker compose up -d`
3. 检查MySQL服务: `systemctl status mysql`

**预期**: 数据库连接成功，服务运行
**实际**: 连接超时，docker命令不存在，服务未运行

**修复方案**:
**方案A (推荐 - 快速)**: 安装Docker并启动docker-compose服务
```bash
# 安装Docker (需要sudo)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo systemctl start docker
sudo systemctl enable docker

# 启动服务
cd /home/ubuntu/.openclaw/workspace/projects/TraderMate/tradermate
docker compose up -d mysql redis
```

**方案B**: 在宿主机安装MySQL和Redis
```bash
# MySQL
sudo apt update && sudo apt install -y mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'tradermate01@A';"
mysql -u root -p'tradermate01@A' -e "CREATE DATABASE tradermate CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
# 同样创建tushare, vnpy, akshare数据库

# Redis
sudo apt install -y redis-server
sudo systemctl start redis
sudo systemctl enable redis
```

**方案C (应急)**: 修改代码使用SQLite（仅开发），但需要评估工作量

**责任人**: @operator
**优先级**: Highest
**截止日期**: 2026-03-03 18:00 UTC

---

### B002 - 测试套件完全无法运行

**模块**: Testing
**严重性**: 🔴 Critical - Blocker
**状态**: Open

**描述**:
执行 `pytest tests/` 时发生内部错误，测试无法收集：
1. Python路径未正确配置，`app` 模块无法导入
2. 测试依赖不存在的模块 `app.datasync.service.sync_coordinator`
3. 测试文件在导入失败时调用 `sys.exit(1)`，导致pytest INTERNALERROR而非FAIL

**重现步骤**:
```bash
cd /home/ubuntu/.openclaw/workspace/projects/TraderMate/tradermate
source .venv/bin/activate
pytest tests/ -v
```

**错误日志**:
```
ModuleNotFoundError: No module named 'app'
ModuleNotFoundError: No module named 'app.datasync.service.sync_coordinator'
INTERNALERROR: SystemExit: 1
```

**修复方案**:
1. **添加pytest配置** (在tradermate/根目录创建pytest.ini):
```ini
[pytest]
testpaths = tests
pythonpath = .
addopts = -v --tb=short --strict-markers
```

2. **实现缺失的sync_coordinator模块** 或 **更新测试文件**:
   - 检查 `tests/test_new_sync.py` 是否与实际代码结构匹配
   - 如果 `sync_coordinator` 功能未实现，应删除或重构该测试

3. **修复测试代码结构**:
   - 移除 `sys.exit(1)`，改用 `pytest.fail("message")` 或标准断言
   - 确保测试可被pytest正确发现和收集

4. **验证测试是否可以mock数据库**:
   - 使用 `pytest-mock` 或 `unittest.mock`
   - 避免测试依赖真实数据库连接

**责任人**: @coder (修复代码) + @tester (修复测试)
**优先级**: Highest
**截止日期**: 2026-03-03 20:00 UTC

---

### B003 - 前端TypeScript编译失败

**模块**: Frontend (tradermate-portal)
**严重性**: 🔴 Critical - Blocker
**状态**: Open

**描述**:
`npm run build` 执行失败，TypeScript编译器报告50+个错误：
- 测试中缺少 `@testing-library/jest-dom` 的 `toBeInTheDocument` matcher
- 多个组件有未使用的变量
- 函数调用参数数量不匹配
- 类型不兼容 (`string | undefined` 赋值给 `string`)
- 缺少模块 `../test/utils`
- 类型导入需要使用 `type` 关键字

**错误统计**:
- `TS2339` (属性不存在): 12次
- `TS6133` (未使用变量): 20次
- `TS2554` (参数数量错误): 8次
- `TS2307` (找不到模块): 2次
- `TS2345` (类型不兼容): 2次
- 其他类型错误: 6次

**重现步骤**:
```bash
cd /home/ubuntu/.openclaw/workspace/projects/TraderMate/tradermate-portal
npm run build
```

**Top 5错误**:
1. `Property 'toBeInTheDocument' does not exist on type 'Assertion<any>'`
2. `'xxx' is declared but its value is never read.`
3. `Cannot find module '../test/utils'`
4. `Expected 0 arguments, but got 1/2/3`
5. `Argument of type 'string | undefined' is not assignable to parameter of type 'SetStateAction<string>'`

**修复方案**:
1. **安装缺失的测试依赖**:
```bash
npm install --save-dev @testing-library/jest-dom @testing-library/react @testing-library/user-event
```
在 `src/test/setup.ts` 或 `vitest.config.ts` 中添加:
```typescript
import '@testing-library/jest-dom/vitest'
```

2. **修复TypeScript严格模式问题**:
   - 在 `tsconfig.json` 中，临时放宽 `strict` 或 `noUnusedLocals` (仅应急)
   - 或修复所有未使用变量警告（推荐）

3. **导入类型使用 `type` 关键字**:
```typescript
// 修改前
import { RenderOptions } from '@testing-library/react'
// 修改后
import type { RenderOptions } from '@testing-library/react'
```

4. **修复函数签名**:
   - 检查 `Strategies.tsx` 中的函数调用，确保传递正确数量的参数
   - 检查 `useState` 的初始值类型

5. **补充缺失的 `src/test/utils.tsx`**:
   参考标准测试工具函数结构，提供：
   - `renderWithProviders(ui, options)`
   - `createMockStrategy()`
   - `mockAuth()`

6. **修复全局变量问题**: 在 `src/test/setup.ts` 中添加:
```typescript
declare global {
  var global: typeof globalThis
}
global = globalThis
```

**责任人**: @coder (前端)
**优先级**: Highest
**截止日期**: 2026-03-03 22:00 UTC

---

## 主要问题 (Major)

### M001 - JWT SECRET_KEY安全性不足

**模块**: Security
**严重性**: 🟡 Medium
**状态**: Open

**描述**:
`.env` 文件中的 `SECRET_KEY=your-jwt-secret-key-here` 长度仅27字符，且是默认值。

**风险**:
- JWT签名可能被暴力破解
- 生产环境绝对禁止使用默认密钥

**修复**:
```bash
openssl rand -hex 32  # 生成32字节(64字符)
# 更新 .env: SECRET_KEY=<生成的64字符hex>
```

**责任人**: @operator / @main
**优先级**: Medium
**截止日期**: 2026-03-03 12:00 UTC

---

### M002 - npm安全漏洞

**模块**: Frontend Dependencies
**严重性**: 🟡 Medium
**状态**: Open

**描述**:
`npm audit` 报告3个漏洞 (1 moderate, 2 high)

**受影响包** (需运行 `npm audit` 查看详情)

**修复**:
```bash
cd tradermate-portal
npm audit fix  # 自动修复
# 如果失败:
npm audit fix --force  # 谨慎，可能引入破坏性更新
```

**责任人**: @coder
**优先级**: Medium
**截止日期**: 2026-03-04 12:00 UTC

---

## 次要问题 (Minor)

### S001 - 虚拟环境创建非标准方法

**模块**: DevOps
**严重性**: 🟢 Low
**状态**: Info

**描述**:
系统Python受保护（PEP 668），无法使用标准 `python -m venv`。使用了 `virtualenv` 作为替代。

这本身不是问题，但理想情况下应安装系统包 `python3.12-venv` 以使用标准方法。

**建议**:
```bash
sudo apt install python3.12-venv
# 之后可用: python -m venv .venv
```

---

## 问题统计

| 严重性 | 数量 | 已修复 |
|--------|------|--------|
| Critical (阻塞) | 3 | 0 |
| High | 0 | 0 |
| Medium | 2 | 0 |
| Low | 1 | 0 |
| **总计** | **6** | **0** |

---

## 质量门禁状态

| 检查项 | 要求 | 实际 | 通过 |
|--------|------|------|------|
| pytest 100%通过 | ✅ | ❌ 无法运行 | ❌ |
| 测试覆盖率 ≥80% | ✅ | ❌ 未执行 | ❌ |
| 前端构建成功 | ✅ | ❌ 50+错误 | ❌ |
| 后端启动健康 | ✅ | ❌ DB不可用 | ❌ |
| API文档可访问 | ✅ | ❌ API未运行 | ❌ |
| 无Critical漏洞 | ✅ | ❌ 3阻塞问题 | ❌ |
| **质量门禁** | **全部通过** | | **❌ 未通过** |

---

**结论**: **环境未就绪** - 需优先解决3个Critical阻塞问题，然后重新验证。
