# Backtest API

Base path: `/api/backtest`

Auth: Bearer token required for all endpoints.

## Endpoints

- `POST /api/backtest`
  - Submit a single backtest (runs via FastAPI BackgroundTasks, in-process).
  - Body example:
    ```json
    {
      "strategy_id": 1,
      "strategy_class": "TripleMAStrategy",
      "vt_symbol": "000001.SZ",
      "start_date": "2024-01-01",
      "end_date": "2024-12-31",
      "parameters": {"fast_window": 5, "slow_window": 20},
      "capital": 100000.0,
      "rate": 0.0001,
      "slippage": 0.0,
      "size": 1,
      "benchmark": "399300.SZ"
    }
    ```
  - Response: `{ "job_id": "<uuid>", "status": "pending", "message": "..." }`

- `POST /api/backtest/batch`
  - Submit a batch backtest for multiple symbols.
  - Body: same as above, but `symbols: ["000001.SZ", "000002.SZ"]` and optional `top_n`.

- `GET /api/backtest/{job_id}`
  - Get single-job status/result.

- `GET /api/backtest/batch/{job_id}`
  - Get batch-job status/result.

- `DELETE /api/backtest/{job_id}`
  - Cancel a pending/running job.

- `GET /api/backtest/history/list`
  - List historical backtests from DB.
  - Query: `limit`, `offset` (defaults 50/0)

- `GET /api/backtest/history/{job_id}`
  - Detailed backtest result from DB.

## Notes

- `/api/backtest` uses in-process background tasks. For Redis/RQ-backed jobs, use `/api/queue` endpoints.
- Results are persisted to DB for history and may also be cached in Redis job storage.
