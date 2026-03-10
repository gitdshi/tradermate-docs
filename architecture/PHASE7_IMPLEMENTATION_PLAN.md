# Phase 7 Implementation Plan - TraderMate

**Prepared by**: Coder Agent (Mark)  
**Date**: 2026-03-03  
**Based on**: `API_CONTRACT_V1.yaml` (OpenAPI 3.0)
**Status**: Draft - Awaiting Approval

---

## Executive Summary

Implementing the **full Phase 7** (Analytics + Portfolio + Optimization APIs) requires **approximately 12.5 person-days** (≈2.5 weeks).

Given the current codebase state (APIs missing, DB tables missing), this represents a **significant development effort** that could block UI development if not staged.

### Recommendation

**Split into Phase 7 V1 (5 days) + Phase 7 V2 (remaining 7.5 days)**

Phase 7 V1 delivers minimal viable endpoints to unblock frontend integration:
- Portfolio: `GET /positions` only (read-only)
- Analytics: `GET /dashboard` only (basic metrics from backtest results)
- Optimization: `POST /optimization` + `GET /{job_id}` (submit & status only)

Full details below.

---

## 1. Missing API Endpoints

| Module | Endpoint | Status | Priority |
|--------|----------|--------|----------|
| Analytics | `GET /api/analytics/dashboard` | ❌ Missing | P0 |
| Analytics | `GET /api/analytics/risk-metrics` | ❌ Missing | P1 |
| Analytics | `GET /api/analytics/compare` | ❌ Missing | P1 |
| Portfolio | `GET /api/portfolio/positions` | ❌ Missing | P0 |
| Portfolio | `GET /api/portfolio/closed-trades` | ❌ Missing | P1 |
| Portfolio | `POST /api/portfolio/positions/{id}/close` | ❌ Missing | P1 |
| Optimization | `POST /api/optimization` | ❌ Missing | P0 |
| Optimization | `GET /api/optimization/{job_id}` | ❌ Missing | P0 |
| Optimization | `GET /api/optimization/history` | ❌ Missing | P2 |
| Optimization | `POST /api/optimization/{job_id}/cancel` | ❌ Missing | P2 |

**Total**: 10 endpoints to implement

---

## 2. Database Schema Additions

### 2.1 positions table

```sql
CREATE TABLE positions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    symbol VARCHAR(50) NOT NULL,
    quantity INT NOT NULL,
    avg_price DOUBLE NOT NULL,
    current_price DOUBLE,
    market_value DOUBLE,
    unrealized_pnl DOUBLE,
    opened_at DATETIME NOT NULL,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_symbol (user_id, symbol)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 2.2 trades table

```sql
CREATE TABLE trades (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    symbol VARCHAR(50) NOT NULL,
    direction ENUM('long','short') NOT NULL,
    open_time DATETIME NOT NULL,
    close_time DATETIME,
    open_price DOUBLE NOT NULL,
    close_price DOUBLE,
    quantity INT NOT NULL,
    pnl DOUBLE,
    commission DOUBLE DEFAULT 0,
    fees DOUBLE DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_time (user_id, open_time),
    INDEX idx_symbol (symbol)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 2.3 account_balance (optional, can be computed on fly)

```sql
CREATE TABLE account_balance (
    user_id INT PRIMARY KEY,
    cash_balance DOUBLE DEFAULT 0,
    total_equity DOUBLE,
    last_updated DATETIME,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Migration**: Add SQL scripts to `mysql/init/` and update `docker-compose.yml` to execute them.

---

## 3. Detailed Implementation Plan

### Phase 7a: Foundation & Data Layer (3.5 days)

#### Day 1: Schema Design & Migration
- [ ] Finalize DB schema review with team
- [ ] Create `mysql/init/phase7.sql` (or separate files)
- [ ] Update docker-compose to run new init scripts on container creation
- [ ] Write schema validation tests (ensure tables exist, constraints work)

#### Day 2-3: DAO & Service Implementation
- [ ] **PositionsDao**
  - `get_positions(user_id)` → list of positions with current_price
  - `get_position_by_id(position_id)` → single position
  - `upsert_position(...)` → create or update
  - `close_position(position_id, quantity, price)` → close and record trade
- [ ] **TradesDao**
  - `open_trade(...)` → insert open trade
  - `close_trade(trade_id, close_price)` → update close fields and calculate P&L
  - `get_closed_trades(user_id, filters)` → paginated history
- [ ] **PositionsService**
  - `calculate_unrealized_pnl()` using latest market data (from `tushare` or cached)
  - `sync_positions_with_market()` to update `current_price` and `market_value`
- [ ] **TradesService**
  - `execute_close_transaction()`: atomic update positions → trades
  - P&L calculation logic (commission, fees included)

**Tests**: Mock DB tests for DAO methods; service unit tests with fixtures.

---

### Phase 7b: Portfolio API (2 days)

#### Day 4: Read Endpoints
- [ ] `GET /api/portfolio/positions`
  - Returns: `{ positions: [...], total_market_value: float, total_unrealized_pnl: float }`
  - Query: Positions joined with latest price (from data cache or external API)
- [ ] `GET /api/portfolio/closed-trades`
  - Query params: `?page=1&limit=50&start_date=&end_date=`
  - Returns: `{ items: [...], total: int }`
- [ ] Add pagination support
- [ ] Write endpoint tests (FastAPI TestClient)
- [ ] Update API documentation

#### Day 5: Write Operation
- [ ] `POST /api/portfolio/positions/{position_id}/close`
  - Request: `{ quantity?: int, price?: float }`
  - If quantity omitted → full close; if price omitted → market order (fetch latest)
  - Use DB transaction: update position → insert trade
  - Return closed trade record
- [ ] Edge case tests: partial close, insufficient quantity, price validation
- [ ] Integration test with real DB (docker-compose)

---

### Phase 7c: Analytics API (2.5 days)

#### Day 6: Data Layer (DAO + Service)
- [ ] **AnalyticsDao**
  - `get_portfolio_aggregates(user_id)` → total MVC, total P&L, sector allocation
  - `get_performance_curve(user_id, start_date, end_date)` → equity curve data
  - `get_risk_metrics(user_id)` → volatility, max drawdown, VaR, beta, alpha
  - `get_backtest_comparison_data(job_ids)` → equity curves for multiple backtests
- [ ] **AnalyticsService**
  - Risk calculations: 
    - Volatility: standard deviation of daily returns
    - Max drawdown: from equity curve
    - VaR: historical or parametric (95% confidence)
    - Beta/Alpha: against benchmark (require market index data)
- [ ] Ensure `pandas`/`numpy` usage is efficient (vectorized)

#### Day 7-8: API Endpoints
- [ ] `GET /api/analytics/dashboard`
  - Returns summary + allocation + performance curve
  - If no live positions, return from backtest_history.result fallback
- [ ] `GET /api/analytics/risk-metrics`
  - Returns all risk metrics (may compute on fly or cache)
- [ ] `GET /api/analytics/compare?ids=job1,job2`
  - Compare backtest results (metrics table + equity curves)
- [ ] Add caching layer (Redis) if calculations heavy
- [ ] Tests + API docs

---

### Phase 7d: Optimization API (2.5 days)

#### Day 9: Algorithm Implementation
- [ ] **OptimizationService**
  - Support two methods:
    - **Genetic Algorithm**: Use `deap` (already in requirements.txt)
    - **Grid Search**: Exhaustive parameter grid
  - `run_optimization(strategy_class, param_ranges, symbol, dates, initial_capital)`
  - Returns best parameters and corresponding metrics
- [ ] **Worker Task**: `app/worker/service/tasks.py`
  - `run_optimization_job(job_id, params)` - long-running, updates job status in DB
- [ ] Job storage integration (use existing `JobStorageService`)

#### Day 10: API + Queue Integration
- [ ] `POST /api/optimization`
  - Request includes `method: "genetic" | "grid"`, `param_ranges`
  - Creates job in DB, enqueues to RQ
  - Returns `{ job_id, status: "queued" }`
- [ ] `GET /api/optimization/{job_id}`
  - Returns status, progress, results if completed
- [ ] `GET /api/optimization/history`
  - List user's optimization jobs (pagination)
- [ ] `POST /api/optimization/{job_id}/cancel`
  - Cancel queued/running job (RQ cancel)
- [ ] Tests

---

### Phase 7e: Integration & Testing (2 days)

#### Day 11-12: System Validation
- [ ] **API Contract Validation**: Verify all implemented endpoints match `API_CONTRACT_V1.yaml`
- [ ] **End-to-End Tests**: Frontend pages (Analytics, Portfolio, Optimization) can successfully call APIs
- [ ] **Performance**: 
  - Add DB indexes: `positions(user_id, symbol)`, `trades(user_id, open_time)`
  - Test with large datasets (100k+ trades)
  - Implement Redis caching for expensive aggregations
- [ ] **Documentation**: Update `docs/API_README.md` with new endpoints
- [ ] **Migration Scripts**: Ensure existing users' data (if any) migrates cleanly

---

## 4. Effort Estimation Summary

| Phase | Tasks | Person-Days |
|-------|-------|-------------|
| 7a: Data Layer | Schema + DAO + Service | 3.5 |
| 7b: Portfolio API | 3 endpoints | 2.0 |
| 7c: Analytics API | 3 endpoints + risk calcs | 2.5 |
| 7d: Optimization API | GA/Grid + 4 endpoints | 2.5 |
| 7e: Integration | Testing + docs + perf | 2.0 |
| **Total (Full)** | | **12.5 days** |

**Calendar estimate**: ~2.5 weeks (assuming 5-day sprints, some parallelization possible).

---

## 5. Phase 7 V1 - Minimum Viable Product (5 days)

If full scope is too large, implement **only the following**:

### V1 Scope

1. **Portfolio API** (1.5 days)
   - ✅ `GET /api/portfolio/positions` (read-only)
   - ❌ No `closed-trades` or `close` endpoint

2. **Analytics API** (2 days)
   - ✅ `GET /api/analytics/dashboard` (basic metrics only)
   - ❌ No `risk-metrics`, no `compare`

3. **Optimization API** (1.5 days)
   - ✅ `POST /api/optimization` (submit)
   - ✅ `GET /api/optimization/{job_id}` (status only)
   - ❌ No `history` or `cancel`

**Total**: 5 person-days

**Rationale**: 
- Unblocks frontend UI development (pages can render with mock/minimal data)
- Provides core value: view positions, see portfolio analytics, run optimizations
- Defers lower-priority features (trading closure, advanced risk metrics, job management)

---

## 6. Technical Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Real-time risk metrics performance degradation | Medium | High | Use Redis caching; precompute nightly |
| Positions table remains empty (no live trading yet) | High | Medium | Analytics fallback to backtest_history.result for mock data |
| Symbol name/industry missing | Low | Medium | Join with tushare.stock_basic |
| Portfolio close transaction inconsistency | Medium | Medium | DB transaction; proper error handling; rollback on failure |
| Complex aggregations slow on large datasets | Medium | Medium | Add indexes; consider materialized views or summary tables |
| React 19 incompatibility | High | Medium | **Downgrade to React 18 LTS** immediately (per Designer recommendation) |
| DEAP library learning curve | Medium | Low | Use simple grid search first; GA as enhancement |

---

## 7. Dependencies

- **Database**: MySQL (tradermate + tushare + vnpy) - ✅ already in place
- **Cache**: Redis - ✅ available (RQ uses it)
- **Algorithms**: `deap` (genetic), `scipy` (optimization) - ✅ in requirements.txt
- **Market data**: tushare/akshare APIs - ✅ existing tokens/config
- **Frontend**: Already built (React 19) - ⚠️ may need React 18 downgrade

---

## 8. Recommendations to Main

### 8.1 Decision Required by 2026-03-04

**Choose one**:

**Option A (Recommended)**: Approve **Phase 7 V1** (5 days)
- Pros: Fast unblock, minimal risk, delivers usable UI
- Cons: Only read-only positions; no trade closure, no advanced analytics
- Timeline: Start 2026-03-05, finish 2026-03-11

**Option B**: Approve **Full Phase 7** (12.5 days)
- Pros: Complete functionality, aligned with `API_CONTRACT_V1.yaml`
- Cons: Blocks frontend for ~2.5 weeks; higher complexity
- Timeline: Start 2026-03-05, finish ~2026-03-21

**Option C**: Defer Phase 7 to after MVP launch (re-prioritize)
- Pros: Focus on core trading features first
- Cons: Analytics/Portfolio UI remains placeholder; user feedback delayed

### 8.2 Parallel Work

While Phase 7 is in development, Designer can:
- Finalize UI mockups for V2 features (risk metrics, trade closing)
- Prepare help content for optimization parameter configuration

Tester can:
- Prepare test fixtures for positions/trades
- Draft E2E test scenarios for the new APIs

---

## 9. Appendices

### A. API Endpoint Summary

| Endpoint | Method | Purpose | V1? |
|----------|--------|---------|-----|
| `/api/portfolio/positions` | GET | List open positions | ✅ |
| `/api/portfolio/closed-trades` | GET | Trade history | ❌ |
| `/api/portfolio/positions/{id}/close` | POST | Close position | ❌ |
| `/api/analytics/dashboard` | GET | Portfolio overview | ✅ |
| `/api/analytics/risk-metrics` | GET | Risk statistics | ❌ |
| `/api/analytics/compare` | GET | Compare backtests | ❌ |
| `/api/optimization` | POST | Submit optimization | ✅ |
| `/api/optimization/{job_id}` | GET | Get optimization status | ✅ |
| `/api/optimization/history` | GET | List optimization jobs | ❌ |
| `/api/optimization/{job_id}/cancel` | POST | Cancel optimization | ❌ |

### B. Database Tables Needed

| Table | Primary Use | Required for V1? |
|-------|-------------|------------------|
| positions | Open positions | ✅ (Portfolio V1) |
| trades | Trade history & P&L | ❌ (V1 uses read-only) |
| account_balance | Account snapshot | ❌ (optional) |

### C. Files to Create/Modify

```
tradermate/
├── app/
│   ├── domains/
│   │   ├── portfolio/dao/        # new
│   │   │   ├── positions_dao.py
│   │   │   └── trades_dao.py
│   │   ├── portfolio/service/    # new
│   │   │   ├── positions_service.py
│   │   │   └── trades_service.py
│   │   ├── analytics/dao/        # new
│   │   │   └── analytics_dao.py
│   │   ├── analytics/service/    # new
│   │   │   └── analytics_service.py
│   │   └── optimization/dao/     # new (or reuse queue)
│   │       └── optimization_dao.py
│   ├── api/routes/               # new files
│   │   ├── portfolio.py
│   │   ├── analytics.py
│   │   └── optimization.py
│   └── worker/service/tasks.py   # modify: add run_optimization_job
├── mysql/init/
│   └── phase7.sql                # new schema
└── docs/
    └── API_README.md             # update
```

---

**End of Plan**
