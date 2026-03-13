# Queue & Job Management API

Base path: `/api/queue`

Auth: Bearer token required.

## Endpoints

- `GET /api/queue/stats`
  - Queue statistics from Redis job storage.

- `GET /api/queue/jobs`
  - List jobs for the current user.
  - Query: `status`, `limit` (default 50).

- `GET /api/queue/jobs/{job_id}`
  - Get job detail and result (if available).

- `POST /api/queue/jobs/{job_id}/cancel`
  - Cancel a running job.

- `DELETE /api/queue/jobs/{job_id}`
  - Delete a job and its results.

- `POST /api/queue/backtest`
  - Submit a backtest to the Redis/RQ queue.
  - Body: similar to `/api/backtest`, accepts `symbol` or `vt_symbol`.

- `POST /api/queue/bulk-backtest`
  - Submit a bulk backtest (one strategy, many symbols).

- `GET /api/queue/bulk-jobs/{job_id}/results`
  - Paginated child results for a bulk job.
  - Query: `page`, `page_size`, `sort_order` (asc|desc).

- `GET /api/queue/bulk-jobs/{job_id}/summary`
  - Aggregated summary statistics for a bulk job.

## Notes

- `/api/queue/*` uses Redis + job storage and is intended for longer-running workloads.
- `/api/backtest` uses in-process background tasks and is best for smaller local runs.
