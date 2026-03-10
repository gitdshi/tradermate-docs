# TraderMate API Contract

**Version**: 1.0  
**Last Updated**: 2026-03-02  
**Base URL**: `http://localhost:8000/api` (development)  
**Authentication**: JWT Bearer token (except where noted)

---

## Table of Contents

1. [Authentication](#authentication)
2. [Strategies](#strategies)
3. [Strategy Code Utilities](#strategy-code-utilities)
4. [Backtesting](#backtesting)
5. [Queue & Jobs](#queue--jobs)
6. [Market Data](#market-data)
7. [Optimization](#optimization)
8. [Analytics](#analytics)
9. [Portfolio](#portfolio)
10. [System](#system)
11. [Error Handling](#error-handling)
12. [Data Models](#data-models)

---

## Authentication

All protected endpoints require:
```
Authorization: Bearer <access_token>
```

Access tokens are short-lived (~30 min). Use the refresh token endpoint to obtain new tokens.

### POST `/api/auth/register`

Register a new user account.

**Request**:
```json
{
  "username": "string, 3-50 chars, unique",
  "email": "valid email address",
  "password": "string, min 8 chars"
}
```

**Response** (201 Created):
```json
{
  "id": 1,
  "username": "user1",
  "email": "user1@example.com",
  "created_at": "2026-03-02T12:34:56Z"
}
```

### POST `/api/auth/login`

Authenticate and receive tokens.

**Request**:
```json
{
  "username": "string",
  "password": "string"
}
```

**Response** (200 OK):
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

### POST `/api/auth/refresh`

Exchange refresh token for new access token.

**Request**:
```json
{
  "refresh_token": "string"
}
```

**Response** (200 OK):
```json
{
  "access_token": "new_token_here",
  "refresh_token": "new_refresh_token_here",
  "token_type": "bearer",
  "expires_in": 1800
}
```

### GET `/api/auth/me`

Get current user profile.

**Response** (200 OK):
```json
{
  "id": 1,
  "username": "admin",
  "email": "admin@example.com",
  "must_change_password": false
}
```

---

## Strategies

### GET `/api/strategies`

List all strategies for the authenticated user.

**Response** (200 OK):
```json
[
  {
    "id": 1,
    "name": "Triple MA Strategy",
    "class_name": "TripleMAStrategy",
    "description": "Moving average crossover strategy",
    "version": 2,
    "is_active": true,
    "created_at": "2026-03-01T10:00:00Z",
    "updated_at": "2026-03-02T14:30:00Z"
  }
]
```

### POST `/api/strategies`

Create a new strategy.

**Request**:
```json
{
  "name": "string, required",
  "class_name": "string, required, PascalCase",
  "description": "string, optional",
  "parameters": {
    "fast_window": 10,
    "slow_window": 20
  } | "JSON string",
  "code": "Python source code, required"
}
```

**Validation**:
- Class name must match a class defined in the code
- Code must be valid Python and safe (AST validation)
- Parameters must be a JSON object

**Response** (201 Created):
Returns full strategy object with `id`, `version: 1`, timestamps.

### GET `/api/strategies/{id}`

Get full strategy details including code.

**Response** (200 OK):
```json
{
  "id": 1,
  "name": "Triple MA",
  "class_name": "TripleMAStrategy",
  "description": "...",
  "parameters": { "fast_window": 10, "slow_window": 20 },
  "code": "from vnpy_ctastrategy import CtaTemplate\n\nclass TripleMAStrategy(CtaTemplate):\n    ...",
  "version": 2,
  "is_active": true,
  "created_at": "2026-03-01T10:00:00Z",
  "updated_at": "2026-03-02T14:30:00Z"
}
```

### PUT `/api/strategies/{id}`

Update strategy. Partial updates supported.

**Request** (all fields optional):
```json
{
  "name": "string",
  "class_name": "string",
  "description": "string",
  "parameters": { ... } | string,
  "code": "string (complete code replacement)",
  "is_active": boolean
}
```

**Rules**:
- If `code` is provided, it replaces the entire code (version increment)
- If `parameters` provided, replaces stored defaults
- Version auto-increments on code change

**Response** (200 OK): Updated strategy object.

### DELETE `/api/strategies/{id}`

Delete a strategy.

**Response**: 204 No Content

### POST `/api/strategies/{id}/validate`

Validate strategy code without executing.

**Response** (200 OK):
```json
{
  "valid": true,
  "errors": [],
  "warnings": []
}
```

### GET `/api/strategies/{id}/code-history`

List stored code history entries (versions, most recent first).

**Response** (200 OK):
```json
[
  {
    "id": 3,
    "version": 3,
    "created_at": "2026-03-02T14:30:00Z",
    "strategy_name": "Triple MA",
    "class_name": "TripleMAStrategy",
    "parameters": { "fast_window": 10, "slow_window": 20 },
    "size": 2048
  }
]
```

### GET `/api/strategies/{strategy_id}/code-history/{history_id}`

Get a specific historical code version.

**Response** (200 OK):
```json
{
  "id": 3,
  "strategy_id": 1,
  "code": "full Python source here",
  "version": 3,
  "created_at": "2026-03-02T14:30:00Z",
  "strategy_name": "Triple MA",
  "class_name": "TripleMAStrategy",
  "parameters": { "fast_window": 10, "slow_window": 20 }
}
```

### POST `/api/strategies/{strategy_id}/code-history/{history_id}/restore`

Restore a historical code version to become current.

**Response** (200 OK):
```json
{
  "message": "Code history restored successfully",
  "strategy_id": 1,
  "history_id": 3
}
```

### GET `/api/strategies/builtin/list`

List built-in strategy templates (read-only, available to all users).

**Response** (200 OK):
```json
[
  {
    "id": 0,
    "name": "Triple MA",
    "class_name": "TripleMAStrategy",
    "description": "...",
    "version": 0,
    "is_active": true,
    "created_at": "2026-01-01T00:00:00Z",
    "updated_at": "2026-01-01T00:00:00Z"
  }
]
```

---

## Strategy Code Utilities

**Route Prefix**: `/api/strategy-code` (no JWT required for lint/parse)

### POST `/api/strategy-code/parse`

Parse Python code to extract strategy classes and default parameters.

**Request**:
```json
{
  "content": "Python source code string"
}
```

**Response** (200 OK):
```json
{
  "classes": [
    {
      "name": "TripleMAStrategy",
      "base_classes": ["CtaTemplate"],
      "doc": "Triple MA strategy docstring",
      "defaults": {
        "fast_window": 10,
        "slow_window": 20,
        "fixed_size": 1
      }
    }
  ]
}
```

### POST `/api/strategy-code/lint`

Basic linting (syntax + import checks).

**Response** (200 OK):
```json
{
  "diagnostics": [
    {
      "line": 15,
      "col": 4,
      "severity": "error" | "warning",
      "message": "Invalid syntax"
    }
  ]
}
```

### POST `/api/strategy-code/lint/pyright`

Advanced linting using Pyright (if installed).

**Response** (200 OK):
```json
{
  "pyright": { /* full pyright JSON output */ },
  "diagnostics": [ ... ]
}
```

---

## Backtesting

### POST `/api/backtest`

Submit a single backtest job (asynchronous).

**Request**:
```json
{
  "strategy_id": 1,
  "strategy_class": "TripleMAStrategy",
  "strategy_name": "Triple MA (optional)",
  "symbol": "000001.SZ",
  "symbol_name": "平安银行",
  "start_date": "2020-01-01",
  "end_date": "2023-12-31",
  "initial_capital": 100000.0,
  "rate": 0.0003,
  "slippage": 0.0,
  "benchmark": "000300.SH",
  "parameters": {
    "fast_window": 10,
    "slow_window": 20
  }
}
```

**Response** (202 Accepted):
```json
{
  "job_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "queued"
}
```

### POST `/api/backtest/batch`

Submit multi-symbol backtest (bulk job).

**Request**:
```json
{
  "strategy_id": 1,
  "strategy_class": "TripleMAStrategy",
  "symbols": ["000001.SZ", "000002.SZ"],
  "start_date": "2020-01-01",
  "end_date": "2023-12-31",
  "initial_capital": 100000.0,
  "parameters": { ... }
}
```

**Response**: Same format as single backtest, but `job_id` refers to bulk job.

### GET `/api/backtest/{job_id}`

Get backtest job status and result.

**Response** (200 OK):
```json
{
  "job_id": "a1b2...",
  "status": "queued" | "running" | "completed" | "failed",
  "progress": 75,
  "created_at": "2026-03-02T12:00:00Z",
  "started_at": "2026-03-02T12:00:05Z",
  "completed_at": null,
  "result_url": "/api/backtest/results/a1b2...",
  "error_message": null
}
```

### GET `/api/backtest/{job_id}/results`

Get full backtest results (only if status=completed).

**Response** (200 OK):
```json
{
  "job_id": "a1b2...",
  "statistics": {
    "total_return": 0.1542,
    "annual_return": 0.128,
    "sharpe_ratio": 1.42,
    "max_drawdown": -0.082,
    "win_rate": 0.58,
    "total_trades": 147,
    "profit_factor": 1.85,
    "average_win": 1250.50,
    "average_loss": -800.25
  },
  "equity_curve": [
    { "date": "2020-01-02", "equity": 100000, "benchmark": 100000 },
    ...
  ],
  "drawdown_curve": [
    { "date": "2020-02-15", "drawdown": -0.05 },
    ...
  ],
  "trades": [
    {
      "id": 1,
      "datetime": "2020-02-15T10:30:00",
      "direction": "long" | "short",
      "offset": "open" | "close",
      "price": 12.5,
      "volume": 100,
      "pnl": 1250.00
    }
  ]
}
```

### GET `/api/backtest/history`

List historical backtests (with pagination).

**Query Params**: `?page=1&limit=20`

**Response** (200 OK):
```json
{
  "items": [ /* BacktestJob objects */ ],
  "total": 45,
  "page": 1,
  "page_size": 20
}
```

### POST `/api/backtest/{job_id}/cancel`

Cancel a running backtest.

**Response** (200 OK):
```json
{ "message": "Cancellation requested" }
```

---

## Queue & Jobs

### GET `/api/queue/stats`

Get Redis queue statistics.

**Response** (200 OK):
```json
{
  "queues": {
    "backtest": {
      "queued": 5,
      "started": 2,
      "finished": 150,
      "failed": 3
    },
    "optimization": { ... }
  },
  "redis_info": {
    "connected_clients": 12,
    "used_memory": "1.2M"
  }
}
```

### GET `/api/queue/jobs`

List user's jobs with optional filter.

**Query Params**: `?status=queued|running|completed|failed&limit=20&page=1`

**Response** (200 OK):
```json
{
  "items": [ /* job objects with job_id, status, created_at, type */ ],
  "total": 50
}
```

### GET `/api/queue/jobs/{job_id}`

Get job details including logs.

**Response** (200 OK):
```json
{
  "job_id": "...",
  "status": "running",
  "created_at": "2026-03-02T12:00:00Z",
  "started_at": "2026-03-02T12:00:05Z",
  "type": "backtest",
  "progress": 65,
  "logs": [
    { "timestamp": "12:00:05", "message": "Starting backtest..." },
    ...
  ],
  "result_url": "/api/backtest/{job_id}/results"
}
```

### POST `/api/queue/jobs/{job_id}/cancel`

Cancel a queued or running job.

**Response** (200 OK):
```json
{ "message": "Cancellation requested" }
```

### POST `/api/queue/backtest` (deprecated)

**Use** `POST /api/backtest` instead.

### GET `/api/queue/bulk-jobs/{job_id}/results`

Get paginated results for a bulk backtest job.

**Query Params**: `?page=1&page_size=10&sort_order=desc`

**Response** (200 OK):
```json
{
  "items": [
    {
      "job_id": "child_job_id",
      "symbol": "000001.SZ",
      "status": "completed",
      "metrics": { ... }
    }
  ],
  "total": 100
}
```

### GET `/api/queue/bulk-jobs/{job_id}/summary`

Summary statistics for bulk job (mean Sharpe, best/worst performers).

**Response** (200 OK):
```json
{
  "job_id": "...",
  "total_jobs": 100,
  "completed": 95,
  "failed": 5,
  "average_sharpe": 1.32,
  "best_job": { "job_id": "...", "sharpe": 2.15, "symbol": "000001.SZ" },
  "worst_job": { "job_id": "...", "sharpe": 0.45, "symbol": "000005.SZ" }
}
```

---

## Market Data

### GET `/api/data/symbols`

Search/list symbols.

**Query Params**:
- `exchange` (string): Filter by exchange code (e.g., `SZ`, `SH`, `BJ`, `US`, `HK`)
- `keyword` (string): Free-text search in symbol/name
- `limit` (int): Max results (default 100)
- `offset` (int): Pagination offset

**Response** (200 OK):
```json
{
  "items": [
    {
      "symbol": "000001.SZ",
      "name": "平安银行",
      "exchange": "SZ",
      "industry": "银行"
    }
  ],
  "total": 5000
}
```

### GET `/api/data/history`

Get OHLCV historical data.

**Query Params**:
- `symbol` (required): e.g., `000001.SZ`
- `start_date` (YYYY-MM-DD)
- `end_date` (YYYY-MM-DD)
- `interval` (optional): `daily` (default), `1min`, `5min`, etc.

**Response** (200 OK):
```json
[
  {
    "date": "2020-01-02",
    "open": 12.5,
    "high": 12.8,
    "low": 12.4,
    "close": 12.7,
    "volume": 1500000
  }
]
```

### GET `/api/data/indicators`

Compute technical indicators.

**Query Params**:
- `symbol` (required)
- `start_date`, `end_date`
- `indicators` (comma-separated): `ma,ema,rsi,macd,bbands`

**Response** (200 OK):
Time series with indicator columns:
```json
[
  {
    "date": "2020-01-02",
    "close": 12.7,
    "ma_20": 12.5,
    "rsi_14": 58.2
  }
]
```

### GET `/api/data/overview`

Market overview (indices snapshot, market breadth).

**Response** (200 OK):
```json
{
  "indices": [
    { "name": "上证指数", "code": "000001.SH", "close": 3089, "change": 0.52 },
    ...
  ],
  "breadth": {
    "advancers": 1200,
    "decliners": 800,
    "unchanged": 200
  },
  "timestamp": "2026-03-02T15:00:00Z"
}
```

### GET `/api/data/sectors`

Sector performance and aggregation.

**Response** (200 OK):
```json
[
  {
    "sector": "银行",
    "change": 0.85,
    "turnover": 15000000000,
    "leading_symbol": "000001.SZ"
  }
]
```

### GET `/api/data/exchanges`

List available exchanges with human-readable names.

**Response** (200 OK):
```json
[
  { "code": "SH", "name": "上海证券交易所" },
  { "code": "SZ", "name": "深圳证券交易所" },
  { "code": "BJ", "name": "北京证券交易所" },
  { "code": "US", "name": "美股 (NYSE/NASDAQ)" }
]
```

---

## Optimization

### POST `/api/optimization`

Submit a parameter optimization job.

**Request**:
```json
{
  "strategy_class": "TripleMAStrategy",
  "strategy_name": "Triple MA Optimization",
  "symbol": "000001.SZ",
  "start_date": "2020-01-01",
  "end_date": "2023-12-31",
  "initial_capital": 100000.0,
  "method": "genetic" | "grid",
  "population_size": 100,
  "generations": 50,
  "param_ranges": {
    "fast_window": { "min": 5, "max": 20, "step": 1 },
    "slow_window": { "min": 20, "max": 60, "step": 5 }
  },
  "constraints": {
    "fast_window < slow_window": true
  }
}
```

**Response** (202 Accepted):
```json
{
  "job_id": "opt-12345",
  "status": "queued"
}
```

### GET `/api/optimization/{job_id}`

Get optimization status and results (when complete).

**Response** (200 OK):
```json
{
  "job_id": "opt-12345",
  "status": "queued" | "running" | "completed" | "failed",
  "progress": 100,
  "generation": 50,
  "best_fitness": 1.85,
  "results": [
    {
      "rank": 1,
      "parameters": { "fast_window": 12, "slow_window": 28 },
      "metrics": {
        "sharpe_ratio": 1.85,
        "total_return": 0.42,
        "max_drawdown": -0.08
      }
    },
    ...
  ]
}
```

### GET `/api/optimization/history`

List past optimization jobs.

**Query Params**: `?limit=20&offset=0`

**Response** (200 OK):
List of optimization job summaries.

### POST `/api/optimization/{job_id}/cancel`

Cancel running optimization.

**Response** (200 OK):
```json
{ "message": "Cancellation requested" }
```

---

## Analytics

> **Note**: Analytics endpoints are planned for Phase 7, implementation status: **Backend TBD**

### GET `/api/analytics/dashboard`

Portfolio overview metrics.

**Expected Response** (200 OK):
```json
{
  "summary": {
    "total_market_value": 1500000.00,
    "total_pnl": 125000.00,
    "pnl_percentage": 0.0912,
    "day_pnl": 3500.00,
    "sharpe_ratio": 1.45
  },
  "allocation": [
    { "sector": "科技", "value": 500000, "percentage": 0.33 },
    ...
  ],
  "performance": [
    { "date": "2024-01-01", "portfolio": 100000, "benchmark": 100000 },
    ...
  ]
}
```

### GET `/api/analytics/risk-metrics`

Risk analysis metrics.

**Expected Response** (200 OK):
```json
{
  "volatility": 0.185,
  "max_drawdown": -0.123,
  "var_95": -8765.00,
  "beta": 1.15,
  "alpha": 0.0082,
  "concentration_top3": 0.67,
  "rolling_volatility": [
    { "date": "2024-01", "vol_30": 0.18, "vol_60": 0.17, "vol_90": 0.16 }
  ],
  "drawdown_curve": [
    { "date": "2024-02-15", "drawdown": -0.085 }
  ]
}
```

### GET `/api/analytics/compare`

Compare performance of multiple backtest results.

**Query Params**: `?ids=job_id_1,job_id_2,job_id_3`

**Expected Response** (200 OK):
```json
{
  "jobs": [
    {
      "job_id": "job_1",
      "strategy_name": "Triple MA",
      "metrics": { "sharpe_ratio": 1.42, "total_return": 0.15 },
      "equity_curve": [ ... ]
    },
    ...
  ],
  "comparison_table": {
    "columns": ["Metric", "Triple MA", "RSI Strategy"],
    "rows": [
      { "metric": "Sharpe", "values": [1.42, 1.18] },
      ...
    ]
  }
}
```

---

## Portfolio

> **Note**: Portfolio endpoints are planned for Phase 7, implementation status: **Backend TBD**

### GET `/api/portfolio/positions`

Get current open positions.

**Expected Response** (200 OK):
```json
{
  "positions": [
    {
      "symbol": "000001.SZ",
      "symbol_name": "平安银行",
      "quantity": 1000,
      "avg_price": 12.50,
      "current_price": 13.20,
      "market_value": 13200.00,
      "unrealized_pnl": 700.00,
      "unrealized_pnl_pct": 0.056
    }
  ],
  "total_market_value": 1500000.00,
  "total_unrealized_pnl": 125000.00
}
```

### GET `/api/portfolio/closed-trades`

Get historical closed trades.

**Query Params**: `?page=1&limit=50&start_date=...&end_date=...`

**Expected Response** (200 OK):
```json
{
  "items": [
    {
      "id": 1,
      "symbol": "000001.SZ",
      "direction": "long",
      "open_time": "2024-02-01T10:30:00Z",
      "close_time": "2024-02-15T14:00:00Z",
      "open_price": 12.5,
      "close_price": 13.2,
      "quantity": 100,
      "pnl": 700.00
    }
  ],
  "total": 1500
}
```

### POST `/api/portfolio/positions/{position_id}/close`

Close a position (market order).

**Request**:
```json
{
  "quantity": 500,  // optional, defaults to full position
  "price": 13.25    // optional, market order if omitted
}
```

**Response** (200 OK):
```json
{
  "trade_id": 12345,
  "message": "Position closed successfully"
}
```

---

## System

### GET `/api/system/sync-status`

Get data sync status (Tushare/vnpy).

**Response** (200 OK):
```json
{
  "tushare": {
    "last_sync": "2026-03-02T03:00:00Z",
    "next_sync": "2026-03-03T03:00:00Z",
    "status": "idle" | "running" | "error",
    "symbols_ingested": 5000
  },
  "vnpy": {
    "last_sync": "2026-03-02T04:00:00Z",
    "status": "idle"
  }
}
```

---

## Error Handling

### Standard Error Response

```json
{
  "detail": "Human-readable error message",
  "code": "ERROR_CODE",  // optional
  "fields": {            // for validation errors
    "field_name": ["error1", "error2"]
  }
}
```

### HTTP Status Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful GET/PUT |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Validation error, invalid input |
| 401 | Unauthorized | Missing/invalid token |
| 403 | Forbidden | Insufficient permissions, must change password |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Duplicate resource (e.g., username) |
| 422 | Unprocessable Entity | Semantic validation fail |
| 429 | Too Many Requests | Rate limit |
| 500 | Internal Server Error | Unexpected server error |
| 503 | Service Unavailable | Maintenance, DB down |

### Rate Limiting

Currently not enforced. Future implementations may add:
- `X-RateLimit-Limit`: requests per window
- `X-RateLimit-Remaining`: remaining quota
- `X-RateLimit-Reset`: window reset timestamp

---

## Data Models

### Common Types

```typescript
// Frontend TypeScript types (src/types/index.ts)

export type Strategy = {
  id: number;
  name: string;
  class_name: string;
  description?: string;
  parameters: Record<string, any> | string;
  code?: string;
  version: number;
  is_active: boolean;
  created_at: string; // ISO
  updated_at: string; // ISO
};

export type BacktestJob = {
  job_id: string;
  strategy_name: string;
  symbol: string;
  start_date: string;
  end_date: string;
  status: 'queued' | 'running' | 'completed' | 'failed';
  created_at: string;
};

export type BacktestResult = {
  statistics: {
    total_return: number;
    annual_return: number;
    sharpe_ratio: number;
    max_drawdown: number;
    win_rate: number;
    total_trades: number;
  };
  equity_curve: Array<{ date: string; equity: number; benchmark?: number }>;
  drawdown_curve: Array<{ date: string; drawdown: number }>;
  trades: Trade[];
};

export type Position = {
  symbol: string;
  symbol_name?: string;
  quantity: number;
  avg_price: number;
  current_price: number;
  market_value: number;
  unrealized_pnl: number;
  unrealized_pnl_pct: number;
};

export type Trade = {
  id?: number;
  datetime: string;
  direction: 'buy' | 'sell' | 'long' | 'short';
  offset: 'open' | 'close';
  price: number;
  volume: number;
  pnl?: number;
};
```

---

## Versioning & Pagination

### Pagination Standard

List endpoints support:
- `?page=1` (default 1)
- `?limit=20` (default 20, max 100)

Response includes:
```json
{
  "items": [ ... ],
  "total": 150,
  "page": 1,
  "page_size": 20,
  "total_pages": 8
}
```

### API Versioning

Current version: **v1** (implied by `/api/` prefix).

If breaking changes are needed in the future, prefix with `/api/v2/`.

---

## Implementation Status (Backend vs Frontend)

| Endpoint | Backend | Frontend |
|----------|---------|----------|
| Auth | ✅ Complete | ✅ Complete |
| Strategies CRUD | ✅ Complete | ✅ Complete |
| Strategy Code Utils | ✅ Complete | ✅ Complete |
| Backtest (single/batch) | ✅ Complete | ✅ Complete |
| Queue Management | ✅ Complete | ✅ Complete |
| Market Data | ✅ Complete | ✅ Complete |
| Optimization | ✅ Complete (doc) | ✅ Component ready |
| Analytics | ⚠️ Partial/Incomplete | ✅ Component ready |
| Portfolio | ⚠️ Partial/Incomplete | ✅ Component ready |
| System Sync Status | ✅ Complete | ✅ Complete |

**Action Required**:
- Implement `/api/analytics/*` endpoints (dashboard, risk-metrics, compare)
- Implement `/api/portfolio/*` endpoints (positions, closed-trades, close position)
- These are needed for Phase 7 full functionality

---

## Notes for Developers

### Code Generation

Strategy code submitted via API is validated before execution:
1. Syntax check via AST parse
2. Import checks (no dangerous modules like `os`, `sys`, `subprocess`)
3. Class inheritance check (`CtaTemplate` required)
4. Parameter schema extraction from `parameters` class attribute

### Async Job Processing

Long-running operations (backtest, optimization) use Redis + RQ:
- Job ID returned immediately
- Client polls `GET /api/backtest/{job_id}` or uses WebSocket (future)
- Results stored in DB, available via result URL

### Date Format

All dates in request/response: **ISO 8601** (UTC), e.g., `"2026-03-02T14:30:00Z"`. Query parameters may use `YYYY-MM-DD` for convenience.

---

## Revision History

- 2026-03-02: Initial API contract document created
- Comprehensive endpoint documentation
- Added data models and error handling specs
- Marked Phase 7 endpoints as needing implementation
