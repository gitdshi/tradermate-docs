# Testing 测试文档

此目录包含 TraderMate 项目的测试策略、覆盖率报告和环境验证报告。测试质量是交付可靠软件的核心保障。

## 内容

### 📖 测试指南与策略

| 文档 | 描述 | 受众 |
|------|------|------|
| **[TESTING.md](./TESTING.md)** | 🧪 **测试完整指南**<br>- 测试金字塔 (单元/集成/E2E)<br>- 测试命令和配置<br>- 测试数据管理 (fixtures)<br>- Mocking策略 (外部API)<br>- CI/CD集成 | @tester, @coder |
| **[TEST_COVERAGE_REPORT.md](./TEST_COVERAGE_REPORT.md)** | 📊 **测试覆盖率报告**<br>- 当前覆盖率 (目标: ≥80%)<br>- 各模块覆盖率详情<br>- 未覆盖代码分析<br>- 改进建议 | @coder, @tester |

### 📋 验证报告 (按日期)

`validation-reports/` 目录包含自动化验证的详细输出，按日期命名:

```
validation-reports/
├── TEST_RESULTS_SUMMARY.md          # 测试结果摘要 (最新)
├── ENVIRONMENT_VALIDATION_REPORT.md # 环境验证 (数据库/Redis/API健康)
├── ISSUES.md                        # 发现的问题清单
├── 2026-03-02_VALIDATION.md         # 历史报告 (按日期存档)
└── ...
```

**报告类型**:

| 报告 | 生成时机 | 内容 |
|------|----------|------|
| **环境验证** | 每次环境初始化后 | MySQL/Redis/API `/health` 检查结果 |
| **测试结果摘要** | CI运行测试后 | pytest/Vitest/Playwright 测试结果 |
| **问题清单** | 发现缺陷时 | Bug描述、复现步骤、严重等级、责任人 |

---

## 测试金字塔

TraderMate采用标准测试金字塔，确保快速反馈和充分覆盖:

```
        /\
       /  \    E2E (Playwright) - 用户流程验证
      /    \   数量: ~10-20个核心场景
     /      \
    /________\  Integration (pytest) - API/DB集成
   /          \ 数量: ~50-100个
  /____________\ Unit (pytest + Vitest) - 函数/组件级
              数量: >200个 (目标)
```

**质量目标**:
- ✅ 单元测试覆盖率 ≥ 80%
- ✅ 所有API端点至少1个集成测试
- ✅ 核心用户旅程 (登录、策略创建、回测) 有E2E测试
- ✅ CI流水线必须通过所有测试才能合并

---

## 快速开始

### 首次设置

1. **阅读 [TESTING.md](./TESTING.md)** 了解测试框架和命令
2. **配置测试数据库**: 参考 `development/ENVIRONMENT_SETUP.md` 的数据库初始化
3. **安装测试依赖**:
   ```bash
   # 后端
   cd tradermate
   pip install -r requirements-dev.txt  # 或手动安装 pytest, pytest-cov, pytest-asyncio
   
   # 前端
   cd tradermate-portal
   npm ci
   ```

### 运行测试套件

```bash
# ============ 后端测试 ============
cd tradermate

# 单元测试 + 覆盖率
pytest tests/ -v --cov=app --cov-report=term --cov-report=xml

# 仅运行单元测试 (快速)
pytest tests/unit/ -v

# 集成测试 (需要DB+Redis)
pytest tests/integration/ -v

# 特定测试模块
pytest tests/unit/test_auth_service.py -v

# 查看覆盖率HTML报告
pytest --cov=app --cov-report=html
open htmlcov/index.html  # macOS
xdg-open htmlcov/index.html  # Linux

# ============ 前端测试 ============
cd tradermate-portal

# 单元测试 (Vitest)
npm run test:run

# 交互式测试 (watch模式)
npm run test:ui

# E2E测试 (Playwright) - 需要API+前端都运行
npm run test:e2e

# 代码检查 (Lint)
npm run lint
```

---

## 测试类型详解

### 1. 单元测试 (Unit Tests)

**目标**: 验证单个函数/类的行为

**示例**:
```python
# tests/unit/test_auth_service.py
def test_verify_password():
    hash = bcrypt.hashpw(b"password", bcrypt.gensalt())
    assert verify_password("password", hash) is True
    assert verify_password("wrong", hash) is False
```

**运行**: `pytest tests/unit/ -v`

### 2. 集成测试 (Integration Tests)

**目标**: 验证API端点、数据库交互、外部API集成

**示例**:
```python
# tests/integration/test_strategies_api.py
def test_create_strategy(client, db_session):
    response = client.post("/api/strategies", json={"name": "Test", ...})
    assert response.status_code == 201
    assert response.json()["id"] is not None
```

**运行**: `pytest tests/integration/ -v`

**要求**:
- 使用 `pytest.fixtures` 创建隔离测试数据库
- 每个测试事务回滚，避免数据污染
- 使用 Factory Boy 或类似生成测试数据

### 3. 端到端测试 (E2E Tests)

**目标**: 验证完整用户旅程，从UI到后端

**示例** (Playwright):
```typescript
// e2e/login.spec.ts
test('admin can login and see dashboard', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[data-testid=username]', 'admin')
  await page.fill('[data-testid=password]', 'initial-password-from-logs')
  await page.click('[data-testid=login-button]')
  await expect(page).toHaveURL('/dashboard')
})
```

**运行**: `npm run test:e2e`

---

## 测试数据管理

### Fixtures

后端使用 pytest fixtures 提供测试数据:

```python
# tests/conftest.py
@pytest.fixture
def db_session():
    """创建测试数据库会话，每个测试后回滚"""
    session = TestingSessionLocal()
    try:
        yield session
    finally:
        session.rollback()
        session.close()

@pytest.fixture
def admin_user(db_session):
    """创建测试管理员用户"""
    user = User(username="test_admin", email="admin@test.com")
    db_session.add(user)
    db_session.commit()
    return user
```

前端使用 Vitest fixtures:

```typescript
// tests/fixtures/mockApi.ts
export const mockStrategy = {
  id: 1,
  name: "Test Strategy",
  parameters: { ... }
}
```

### Mock外部依赖

- **Tushare API**: 使用 `tests/mocks/tushare_mock.py` 返回模拟市场数据
- **邮件发送**: 使用 `pytest-mock` 或 `aiosmtplib` 的 fake SMTP 服务器
- **Redis**: 使用 `fakeredis` 库模拟 Redis 行为

---

## 质量门禁

### CI/CD 要求

每次PR必须通过:

1. **Lint** (`flake8`, `black --check`, `eslint`)
2. **类型检查** (`mypy`, `tsc --noEmit`)
3. **单元测试** (覆盖率 ≥ 80%)
4. **安全扫描** (`bandit`, `safety check`)
5. **E2E测试** (仅当UI变更时)

### 覆盖率阈值

在 `pytest.ini` 或 `pyproject.toml` 配置:

```ini
[pytest]
addopts = --cov=app --cov-report=xml --cov-report=term --cov-fail-under=80
```

**当前覆盖率**: 见 `TEST_COVERAGE_REPORT.md` (C-002评估为<10%)

---

## 环境验证

每次环境初始化（Docker Compose up, 本地安装）后，运行验证脚本:

```bash
# 后端验证
cd tradermate
pytest tests/validation/ -v

# 输出示例:
# tests/validation/test_database.py::test_mysql_connection PASSED
# tests/validation/test_redis.py::test_redis_ping PASSED
# tests/validation/test_api_health.py::test_health_endpoint PASSED
```

验证报告自动记录到 `validation-reports/YYYY-MM-DD_VALIDATION.md`。

---

## 测试报告解读

### TEST_COVERAGE_REPORT.md

覆盖率报告说明:

| 指标 | 说明 | 目标 |
|------|------|------|
| **Overall Coverage** | 所有app代码的覆盖率 | ≥ 80% |
| **Missing Lines** | 未执行的行数 | 分析原因并补充测试 |
| **Branch Coverage** | 分支覆盖 (if/else) | ≥ 70% |
| **Module Coverage** | 按模块 (auth, strategies, backtest) | 每个模块 ≥ 75% |

**低覆盖率模块**:
- `app/api/routes/` - API端点需要更多集成测试
- `app/domains/strategies/` - 业务逻辑复杂，需补充单元测试
- `app/datasync/service/` - 数据同步daemon测试不足

### ENVIRONMENT_VALIDATION_REPORT.md

包含以下检查结果:

✅ **PASS**: MySQL连接、Redis连接、API `/health` 响应、Swagger文档可访问  
❌ **FAIL**: 具体失败的检查项和错误日志

---

## 常见问题

### Q: 测试数据库如何隔离？

**A**: 使用 `pytest` fixtures 在每个测试开始时创建独立会话，测试结束后回滚。生产数据库**绝不**用于测试。

### Q: E2E测试太慢，如何加速？

**A**: 
1. 使用 `--headed` 运行可见浏览器 (调试)
2. 并行化: `npm run test:e2e -- --workers=2`
3. 仅运行变更相关测试: `--grep "login"`
4. 使用 CI 缓存 Playwright 浏览器镜像

### Q: 如何mock Tushare API？

**A**: 使用 `tests/mocks/tushare_mock.py`，在测试中 `patch` `marketDataAPI` 的 `get_price` 方法返回模拟数据。

详细问题见 [TROUBLESHOOTING.md](../development/TROUBLESHOOTING.md#测试相关)。

---

## 测试改进计划 (Roadmap)

- [ ] **Q1 2026**: 覆盖率从 <10% 提升到 60%
  - [ ] 补充 `app/api/routes/` 集成测试 (20个端点 × 2测试 = 40个)
  - [ ] 补充 `app/domains/strategies/` 单元测试 (核心策略逻辑)
  - [ ] 补充 `app/worker/service/` 单元测试 (RQ worker任务)

- [ ] **Q2 2026**: 达到 ≥80% 覆盖率
  - [ ] 实现 `pytest-cov` 门禁 (CI自动检查)
  - [ ] 引入 Mutation Testing (mutmut) 验证测试质量
  - [ ] 建立测试数据工厂 (Factory Boy)

- [ ] **Q3 2026**: E2E测试覆盖核心用户旅程
  - [ ] 登录 + 忘记密码流程
  - [ ] 策略创建 + 回测运行
  - [ ] Portfolio查看 + 交易平仓

---

## 贡献指南

### 提交新测试

1. **命名**: `test_<feature>_<scenario>.py` (后端) 或 `<feature>.spec.ts` (前端E2E)
2. **位置**: 
   - 后端单元: `tests/unit/`
   - 后端集成: `tests/integration/`
   - 前端单元: `tradermate-portal/src/__tests__/`
   - E2E: `tradermate-portal/e2e/`
3. **覆盖率**: 新代码必须伴随测试 (TDD推荐)
4. **PR要求**: 测试通过 + 覆盖率不下降 + 类型检查通过

### 更新测试报告

验证脚本自动生成报告到 `validation-reports/`，手动更新仅用于:
- [ ] `TEST_RESULTS_SUMMARY.md` - CI运行后更新
- [ ] `ISSUES.md` - 发现新问题时更新

---

## 相关工具

| 工具 | 用途 | 文档 |
|------|------|------|
| **pytest** | 后端测试框架 | https://docs.pytest.org/ |
| **pytest-cov** | 覆盖率插件 | https://pytest-cov.readthedocs.io/ |
| **pytest-asyncio** | 异步测试 | https://pytest-asyncio.readthedocs.io/ |
| **Vitest** | 前端单元测试 | https://vitest.dev/ |
| **Playwright** | 前端E2E测试 | https://playwright.dev/ |
| **Factory Boy** | 测试数据工厂 | https://factoryboy.readthedocs.io/ |
| **fakeredis** | Redis mock | https://github.com/cunla/fakeredis-py |

---

**维护者**: @tester  
**质量目标**: 覆盖率 ≥80%, 零P0/P1缺陷漏测  
**最后更新**: 2026-03-03  
**下次全面评估**: 2026-04-01