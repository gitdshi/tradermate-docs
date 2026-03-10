# Phase 7 V1 Database Schema Changes

**Version**: 1.0  
**Date**: 2026-03-09  
**Based on**: SYSTEM_ARCHITECTURE.md + PHASE7_IMPLEMENTATION_PLAN.md

---

## Table of Contents

1. [Overview](#overview)
2. [New Tables](#new-tables)
3. [Modified Tables](#modified-tables)
4. [Migration Strategy](#migration-strategy)
5. [Data Integrity Constraints](#data-integrity-constraints)
6. [Indexing Strategy](#indexing-strategy)
7. [Backward Compatibility](#backward-compatibility)

---

## Overview

Phase 7 V1 requires **two new tables** to support portfolio tracking and optimization jobs. No existing tables are modified (only new columns in `positions` table, which is created new).

### Summary of Changes

| Table | Action | Purpose |
|-------|--------|---------|
| `positions` | CREATE | Current holdings snapshot |
| `trades` | CREATE | Historical trade fills and position closes |
| `optimization_jobs` | CREATE | Async optimization job queue and results |
| (none) | ALTER | No alterations to existing tables in V1 |

All new tables use InnoDB engine with UTF8MB4 charset. Foreign keys reference existing `users(id)` table.

---

## New Tables

### 1. positions

Stores **current open positions** only. When a position is closed, it should be moved to `trades` table (application-level workflow).

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

    -- Foreign Keys
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,

    -- Indexes
    INDEX idx_user_symbol (user_id, symbol),
    INDEX idx_updated (updated_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Current open positions - one row per user/symbol';
```

#### Field Details

| Column | Type | Description |
|--------|------|-------------|
| `id` | INT | Surrogate primary key (not used externally) |
| `user_id` | INT | Owner of position (FK to users) |
| `symbol` | VARCHAR(50) | Trading symbol (e.g., '000001.SZ', 'AAPL.US') |
| `quantity` | INT | Number of shares/contracts held |
| `avg_price` | DOUBLE | Weighted average entry price |
| `current_price` | DOUBLE | Latest market price (cached, updated by price feed) |
| `market_value` | DOUBLE | `quantity * current_price` (cached) |
| `unrealized_pnl` | DOUBLE | `(current_price - avg_price) * quantity` (cached) |
| `opened_at` | DATETIME | Position open timestamp (when first buy executed) |
| `updated_at` | DATETIME | Last update timestamp (auto-updated) |

#### Notes
- `current_price` and `market_value` are **denormalized** for performance; refreshed every few minutes or on significant price movement.
- `unrealized_pnl` is **computed** and stored; avoids runtime calculation for list views.
- Position is *open* as long as no corresponding `trades.close_time` record exists. Application logic determines when to delete from `positions` and insert into `trades`.
- `symbol` does not have FK to `symbols` table because symbol master may not exist in early versions.

---

### 2. trades

Historical record of all **closed** trades and position lifecycles.

```sql
CREATE TABLE trades (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    symbol VARCHAR(50) NOT NULL,
    direction ENUM('long','short') NOT NULL,
    open_time DATETIME NOT NULL,
    close_time DATETIME,                   -- NULL = still open
    open_price DOUBLE NOT NULL,
    close_price DOUBLE,
    quantity INT NOT NULL,
    pnl DOUBLE,                           -- Realized P&L (NULL if position open)
    commission DOUBLE DEFAULT 0,
    fees DOUBLE DEFAULT 0,
    -- Foreign Keys
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,

    -- Indexes
    INDEX idx_user_time (user_id, open_time),
    INDEX idx_symbol (symbol),
    INDEX idx_close_time (close_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Trade history and closed positions';
```

#### Field Details

| Column | Type | Description |
|--------|------|-------------|
| `id` | INT | Primary key |
| `user_id` | INT | Trader ID (FK to users) |
| `symbol` | VARCHAR(50) | Traded symbol |
| `direction` | ENUM | 'long' (buy) or 'short' (sell) |
| `open_time` | DATETIME | Entry timestamp |
| `close_time` | DATETIME | Exit timestamp (NULL if position still open) |
| `open_price` | DOUBLE | Entry price |
| `close_price` | DOUBLE | Exit price (NULL if open) |
| `quantity` | INT | Shares/contracts |
| `pnl` | DOUBLE | Realized profit/loss (computed when closing) |
| `commission` | DOUBLE | Broker commission |
| `fees` | DOUBLE | Additional fees (taxes, exchange fees) |

#### Notes
- In V1, we only **read** from `trades` via `/portfolio/closed-trades`. Writes happen via broker sync or manual entry (outside V1 scope).
- `pnl` should be computed as: `(close_price - open_price) * quantity * direction_multiplier` (+1 for long, -1 for short) minus `commission + fees`.
- When position closes, create `trades` record and delete from `positions` in same transaction.

---

### 3. optimization_jobs

Stores optimization job metadata, parameters, and results.

```sql
CREATE TABLE optimization_jobs (
    job_id VARCHAR(64) PRIMARY KEY,
    user_id INT NOT NULL,
    strategy_class VARCHAR(100) NOT NULL,
    symbol VARCHAR(50) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    initial_capital DOUBLE NOT NULL,
    method ENUM('genetic', 'grid') NOT NULL DEFAULT 'genetic',
    param_ranges JSON NOT NULL,              -- {"param": {"min":x, "max":y, "step":z}}
    status ENUM('queued','running','completed','failed','cancelled') NOT NULL DEFAULT 'queued',
    progress INT DEFAULT 0,                  -- 0-100
    best_parameters JSON,                    -- Best param set (when completed)
    best_fitness DOUBLE,                     -- Fitness score (Sharpe ratio, etc.)
    all_results JSON,                        -- Full ranked results (top 100 or all)
    error_message TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Foreign Keys
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,

    -- Indexes
    INDEX idx_user_status (user_id, status),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='Optimization job queue and results';
```

#### Field Details

| Column | Type | Description |
|--------|------|-------------|
| `job_id` | VARCHAR(64) | Unique identifier (UUID/ULID) |
| `user_id` | INT | Job owner (FK to users) |
| `strategy_class` | VARCHAR(100) | Strategy class name in code |
| `symbol` | VARCHAR(50) | Backtest symbol |
| `start_date` / `end_date` | DATE | Backtest date range |
| `initial_capital` | DOUBLE | Starting cash |
| `method` | ENUM | 'genetic' or 'grid' |
| `param_ranges` | JSON | Parameter search space |
| `status` | ENUM | Job lifecycle state |
| `progress` | INT | 0-100% completion |
| `best_parameters` | JSON | Winning parameter set (if completed) |
| `best_fitness` | DOUBLE | Objective function value (higher = better) |
| `all_results` | JSON | Full result set (array of ranked results, may truncate) |
| `error_message` | TEXT | Error details if status in (failed, cancelled) |
| `created_at` / `updated_at` | DATETIME | Timestamps |

#### Notes
- Use **UUID v4** or **ULID** for `job_id` to avoid collisions across distributed workers.
- `param_ranges` JSON structure:
  ```json
  {
    "fast_window": {"min": 5, "max": 20, "step": 1},
    "slow_window": {"min": 20, "max": 60, "step": 5}
  }
  ```
- `all_results` can store full result set if small (<100 entries). For large sets, store top 100 and truncate to limit row size. Full raw data can be archived to S3 if needed (future enhancement).
- Worker process updates `status`, `progress`, `best_parameters`, `best_fitness`, `all_results` atomically (avoid race conditions).
- Cleanup strategy: Archive jobs older than 90 days to `optimization_jobs_archive` (future enhancement). V1 keeps all rows in main table.

---

## Modified Tables

**None in V1.** All changes are additive (new tables). No schema alterations to existing tables.

---

## Migration Strategy

### Migration Order

1. **Create `positions` table** (no data migration, table is new)
2. **Create `trades` table** (backfill existing closed positions from `positions` if any)
3. **Create `optimization_jobs` table** (empty)

### Handling Existing Data

If there are existing open positions in legacy storage (e.g., CSV, JSON, or custom tables), migrate them to `positions` table:

```sql
-- Example migration (adjust to actual source schema)
INSERT INTO positions (user_id, symbol, quantity, avg_price, current_price, market_value, unrealized_pnl, opened_at, updated_at)
SELECT
    user_id,
    symbol,
    quantity,
    avg_price,
    current_price,
    quantity * current_price,
    (current_price - avg_price) * quantity,
    open_date,
    NOW()
FROM legacy_positions
WHERE is_open = 1;
```

If no legacy structure exists (fresh install), skip backfill.

### Idempotent Migration

All `CREATE TABLE` statements use `IF NOT EXISTS` to support re-runs:

```sql
CREATE TABLE IF NOT EXISTS positions (...);
CREATE TABLE IF NOT EXISTS trades (...);
CREATE TABLE IF NOT EXISTS optimization_jobs (...);
```

### Rollback Plan

If migration fails:
1. Drop new tables: `DROP TABLE IF EXISTS optimization_jobs, trades, positions;`
2. Log error and alert operator
3. Restore from backup if data was already inserted

---

## Data Integrity Constraints

### Foreign Keys

| Table | Column | References | On Delete | On Update |
|-------|--------|------------|-----------|-----------|
| `positions` | `user_id` | `users(id)` | CASCADE | RESTRICT |
| `trades` | `user_id` | `users(id)` | CASCADE | RESTRICT |
| `optimization_jobs` | `user_id` | `users(id)` | CASCADE | RESTRICT |

**Rationale**: When a user is deleted, all their data should be removed for GDPR compliance.

### Check Constraints

- `positions.quantity > 0`
- `positions.avg_price > 0`
- `positions.current_price >= 0`
- `trades.quantity > 0`
- `trades.open_price > 0`
- `optimization_jobs.progress BETWEEN 0 AND 100`

(Implemented at application layer in V1; can move to DB constraints in future)

### Unique Constraints

- `optimization_jobs.job_id` is PRIMARY KEY (globally unique)
- No unique constraint on `(user_id, symbol)` for `positions` because a user can only have one position per symbol. However, application enforces this; DB constraint not added in V1 to allow flexibility for partial fills.

---

## Indexing Strategy

### positions

| Index | Columns | Purpose |
|-------|---------|---------|
| PRIMARY | `id` | Row lookup |
| `idx_user_symbol` | `(user_id, symbol)` | Fetch single position quickly (user's positions page) |
| `idx_updated` | `(updated_at)` | Identify stale positions for cache invalidation |

### trades

| Index | Columns | Purpose |
|-------|---------|---------|
| PRIMARY | `id` | Row lookup |
| `idx_user_time` | `(user_id, open_time)` | Query trade history by user with date range |
| `idx_symbol` | `(symbol)` | Cross-user symbol analytics (rare, but useful) |
| `idx_close_time` | `(close_time)` | Query closed positions by close date (reports) |

### optimization_jobs

| Index | Columns | Purpose |
|-------|---------|---------|
| PRIMARY | `job_id` | Direct lookup by job ID |
| `idx_user_status` | `(user_id, status)` | List user's jobs filtered by status (running/completed) |
| `idx_created` | `(created_at)` | Admin cleanup queries, time-based pagination |

---

## Backward Compatibility

- All V1 endpoints are **additive**; existing API endpoints remain unchanged
- Frontend must handle **empty arrays** for `positions` if no holdings
- `trades` table will be empty initially; closed-trades endpoint returns empty list
- `optimization_jobs` starts empty; history endpoint returns empty list
- No breaking changes to existing database schema

---

## Version History

- **1.0** (2026-03-09): Initial schema for Phase 7 V1 (Portfolio read-only, Analytics dashboard, Optimization submit/status)

---

## End of Data Model V1

**Next**: Review by Coder for implementation feasibility, then move to DEPLOYMENT_V1.md.
