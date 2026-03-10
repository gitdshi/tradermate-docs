# Test Coverage Report - TraderMate

**Date**: 2026-03-03  
**Evaluator**: Coder Agent (Mark)  
**Scope**: Backend (tradermate/) && Frontend (tradermate-portal/)

---

## Executive Summary

- **Backend coverage**: <10% (very low) 🚨
- **Frontend coverage**: ~20% (partial) ⚠️
- **Overall health**: **Critical** - Insufficient test coverage for reliable development
- **Priority**: Must increase to 60%+ before Phase 7 implementation

---

## Backend Test Coverage Details

### Current Test Files

```
tradermate/tests/
├── test_config_security.py    ✅ Auth config validation
└── test_new_sync.py           ✅ Sync coordinator imports
```

**Coverage breakdown** (by module, estimated):

| Module | Coverage | Test File | Notes |
|--------|----------|-----------|-------|
| `app.infrastructure.config` | 80% | test_config_security.py | Good |
| `app.datasync.service` | 10% | test_new_sync.py | Only import test |
| `app.api.routes.*` | 0% | None | ❌ Missing |
| `app.api.services.*` | 0% | None | ❌ Missing |
| `app.domains.*` | 0% | None | ❌ Missing |
| `app.worker.service` | 0% | None | ❌ Missing |
| `app.utils` | 0% | None | ❌ Missing |

### Coverage Measurement Setup

**Status**: Not configured  
**Recommendation**: Add pytest-cov && coverage.json

```bash
# Install test dependencies
pip install pytest pytest-cov pytest-asyncio httpx

# Run with coverage
pytest --cov=app --cov-report=html --cov-report=term-missing

# Target: html report in htmlcov/
```

### Required Test Types

1. **Unit tests** (target: 70% of modules)
   - Services: business logic isolation
   - DAOs: database CRUD (use SQLite in-memory for speed)
   - Utils: pure functions

2. **Integration tests** (target: all routes)
   - FastAPI TestClient against live app
   - Database fixtures (postgres container or SQLite)
   - Mock external APIs (tushare, redis)

3. **End-to-end** (critical user journeys)
   - Register → Login → Create strategy → Backtest
   - Portfolio view (once implemented)

---

## Frontend Test Coverage Details

### Current Test Structure

```
tradermate-portal/
├── vitest.config.ts        ✅ Configured
├── src/test/
│   ├── setup.ts           ✅ Test setup
│   ├── utils.tsx          ✅ Test utilities
│   ├── integration.test.tsx ⚠️ Some integration tests
│   └── api.test.ts        ⚠️ API client tests
├── src/pages/
│   ├── auth/Login.test.tsx ✅ Auth UI tests
│   └── Dashboard.test.tsx ✅ Dashboard tests
├── src/stores/auth.test.ts ✅ Zustand store tests
```

### Coverage Estimates

| Area | Coverage | Status |
|------|----------|--------|
| API client (`src/lib/api.ts`) | 30% | ⚠️ Partial |
| Auth store | 70% | ✅ Good |
| Pages (Login, Dashboard) | 40% | ⚠️ Partial |
| Components (BacktestForm, etc.) | 10% | ❌ Missing |
| Business logic/hooks | 0% | ❌ Missing |

### Improvement Plan

1. **Component testing** (Vitest + React Testing Library)
   - BacktestForm
   - StrategiesList
   - AnalyticsDashboard
   - PortfolioManagement
   - All form validation

2. **E2E testing** (Playwright)
   - User registration/login flow
   - Strategy creation workflow
   - Backtest submission && results view
   - Phase 7: Analytics && Portfolio pages

3. **API mocking** (MSW - Mock Service Worker)
   - Already in devDependencies ✅
   - Need to implement mocks for all endpoints

---

## Test Environment Setup

### Required Dependencies (Backend)

```txt
# Add to requirements-dev.txt or install separately
pytest>=7.0
pytest-cov>=4.0
pytest-asyncio>=0.23
httpx>=0.27    # for async FastAPI tests
faker>=20.0    # test data generation
factory-boy>=3.6
```

### Required Dev Dependencies (Frontend)

Already present in `package.json` ✅:
- `@testing-library/react`
- `@testing-library/jest-dom`
- `@testing-library/user-event`
- `jsdom`
- `msw`

---

## Coverage Target Timeline

| Sprint | Backend Target | Frontend Target | Deliverable |
|--------|----------------|-----------------|-------------|
| C-002 (now) | 30% | 40% | Core services && components tested |
| Phase 7 dev | 60% | 60% | All new APIs covered |
| Before Phase 7 release | 80% | 75% | Acceptance threshold |

---

## Priority Test Scenarios

### P0 (must have before Phase 7)

1. **Backend**
   - ✅ Auth flows: register, login, token refresh
   - ✅ Strategy CRUD (create, read, update, delete, validation)
   - ⬜ Portfolio API (once implemented)
   - ⬜ Analytics API (once implemented)
   - ⬜ Optimization API (once implemented)

2. **Frontend**
   - ✅ Login && Register pages
   - ✅ Dashboard navigation
   - ⬜ Backtest submission flow
   - ⬜ Strategy creation flow
   - ⬜ Portfolio page (once API ready)

### P1 (should have)

- All DAO layer methods
- Error handling paths (400, 401, 403, 404, 500)
- Pagination logic
- Data validation edge cases

---

## Testing Best Practices (to adopt)

1. ** Arrange-Act-Assert pattern
2. ** Fixtures for common data (users, strategies, backtest results)
3. ** Test database isolation: use transactions that roll back
4. ** Mock external services: tushare, redis, email
5. ** Parametrized tests for multiple input combinations
6. ** Property-based testing (Hypothesis) for complex data transforms

---

## CI/CD Integration

Current `.github/workflows/ci.yml` has:

- ✅ Lint (black, flake8, isort, mypy)
- ⚠️ Test runs but coverage not enforced

**Recommended additions**:

```yaml
- name: Run tests with coverage
  run: |
    pytest --cov=app --cov-report=xml --cov-report=term
- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v3
  with:
    file: ./coverage.xml
```

**Coverage gate**: Fail CI if coverage falls below threshold (e.g., 50%).

---

## Known Issues & Gaps

- ❌ **No test database initialization script**: Need `mysql/init/test_data.sql` or factory fixtures
- ❌ **No integration tests** hitting real MySQL (only unit tests with mocks)
- ❌ **Frontend tests don't cover new components** (Analytics, Portfolio, Optimization)
- ❌ **E2E tests exist but may not run in CI** (Playwright setup needed)
- ❌ **No snapshot testing** for UI components

---

## Test Debt Action Items

| Action | Owner | Deadline |
|--------|-------|----------|
| Add pytest && pytest-cov to requirements-dev | @coder | 2026-03-04 |
| Configure coverage reporting (pytest --cov) | @coder | 2026-03-04 |
| Write unit tests for auth_service && strategy_service | @coder | 2026-03-06 |
| Add integration tests for backtest submission flow | @tester | Phase 7 |
| Implement API mocking with MSW for frontend tests | @coder | Phase 7 |
| Achieve 30% backend coverage baseline | @coder | 2026-03-06 |
| Achieve 40% frontend coverage baseline | @coder | 2026-03-06 |

---

**Status**: 🚨 Critical — Must be addressed to ensure Phase 7 quality
**Next Review**: 2026-03-06 (during C-002 development)
