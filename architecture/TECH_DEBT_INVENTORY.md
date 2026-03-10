# Tech Debt Inventory - TraderMate

**Last Updated**: 2026-03-03  
**Scope**: tradermate && tradermate-portal codebases  
**Maintenance**: Update during each sprint retro

---

## 🔴 Critical Blockers (Immediate Action Required)

| ID | Debt | Impact | Owner | Target Fix |
|----|------|--------|-------|------------|
| TDB-001 | Analytics, Portfolio, Optimization APIs not implemented | Blocks Phase 7 UI completely | @coder | 2026-03-10 (V1) |
| TDB-002 | Missing `positions` and `trades` database tables | Cannot store live trading data | @coder | 2026-03-05 |
| TDB-003 | React 19 compatibility issues (potential) | Frontend stability risk | @coder | 2026-03-04 (downgrade) |
| TDB-004 | Data sync daemon TODO: missing date detection logic | Data integrity risk | @coder | 2026-03-06 |

---

## 🟡 Medium Priority (Code Quality & Maintainability)

### TDB-005: Hard-coded default values
- **Location**: `app/api/main.py` line with `DEFAULT_ADMIN_HASH`
- **Issue**: Security-sensitive default embedded in code
- **Recommendation**: Move to environment variable or config file
- **Effort**: 0.5 day

### TDB-006: Insufficient test coverage
- **Location**: Backend overall <10% coverage
- **Issue**: Regression risk, low confidence in changes
- **Recommendation**: Add unit tests for services/DAO (target 60%+)
- **Effort**: 5-7 days (ongoing)

### TDB-007: Missing type checking enforcement
- **Location**: No `mypy.ini` or strict mode in CI
- **Issue**: Type safety not guaranteed
- **Recommendation**: Add `mypy --strict` to CI pipeline
- **Effort**: 1-2 days

### TDB-008: Inconsistent project structure
- **Location**: `app/` vs `src/` symlink
- **Issue**: Confusion for new developers
- **Recommendation**: Standardize on `src/` and refactor imports
- **Effort**: 1 day

### TDB-009: Frontend API client may not match backend
- **Location**: `tradermate-portal/src/lib/api.ts`
- **Issue**: Some endpoints (analytics, portfolio) don't exist yet
- **Recommendation**: Implement backend APIs or adjust frontend
- **Effort**: Part of Phase 7

---

## 🟢 Low Priority (Nice to Have)

### TDB-010: Simple logging configuration
- **Location**: `app/infrastructure/logging/logging_setup.py`
- **Issue**: No structured logging (JSON), no log rotation
- **Recommendation**: Integrate `structlog` or similar
- **Effort**: 2 days

### TDB-011: No error monitoring / APM
- **Location**: Whole project
- **Issue**: Production issues hard to detect
- **Recommendation**: Integrate Sentry or Datadog
- **Effort**: 1-2 days

### TDB-012: CI lint rules too permissive
- **Location**: `.github/workflows/ci.yml`
- **Issue**: Flake8 exits with zero even with errors
- **Recommendation**: Fail CI on lint errors
- **Effort**: 0.5 day

### TDB-013: API versioning not planned
- **Location**: All routes under `/api`
- **Issue**: Breaking changes will affect all clients
- **Recommendation**: Versioned routes: `/api/v1/`
- **Effort**: 1-2 days (future)

### TDB-014: No database migration tool
- **Location**: `mysql/init/*.sql` only
- **Issue**: Schema changes hard to track and apply incrementally
- **Recommendation**: Adopt Alembic or similar
- **Effort**: 2-3 days

---

## 📊 Test Coverage Gap Analysis

### Current State
```
Backend:
  Config/Security:        1 test file (80%+)
  Data Sync:              1 test file (10%)
  All other modules:     0% coverage

Frontend:
  Auth store:             ✅ test exists
  Some components:        ✅ test exists
  API layer:              ❌ missing
  Business logic:        ❌ missing
```

### Target Coverage (by Sprint)
- **Sprint 1 (now)**: Backend 30%, Frontend 40%
- **Sprint 2**: Backend 60%, Frontend 60%
- **Sprint 3+**: Backend 80%, Frontend 80%

---

## 🎯 Quick Wins (Fast to Fix, High Value)

1. ⚡ Downgrade React 19 → 18 (1 hour, prevents breakage)
2. ⚡ Add `mypy` strict mode to CI (1 hour, catches type errors)
3. ⚡ Fix data sync daemon TODO (2 hours, prevents data loss)
4. ⚡ Make CI fail on lint errors (30 min, improves code quality)
5. ⚡ Add structured logs (2 hours, improves observability)

---

## 📈 Debt Trend Tracking

| Metric | Current | Target (Sprint End) |
|--------|---------|---------------------|
| Critical Blockers | 4 | 0 |
| Medium Debt Items | 5 | 2 |
| Low Debt Items | 5 | 3 |
| Backend Test Coverage | ~10% | 30% |
| Frontend Test Coverage | ~20% | 40% |

---

## 🔄 Resolution Workflow

1. New debt issues are added to this file with priority
2. Debt is addressed during sprint planning (capacity 20% per sprint)
3. When debt is resolved, mark as ✅ and move to `RESOLVED_DEBT.md`
4. Monthly review: check if new debt emerges faster than we fix it

---

**Keep this file up to date!**  
Tech debt grows silently — fight it continuously.
