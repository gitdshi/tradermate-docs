# Optimization API

Base path: `/api/optimization`

Purpose

- Submit parameter optimization jobs (genetic algorithm or grid search) and retrieve ranked results.

Endpoints

- `POST /api/optimization`
  - Description: Submit an optimization job
  - Auth: Bearer token required
  - Body example:
    ```json
    {
      "strategy_class": "TripleMAStrategy",
      "symbol": "000001.SZ",
      "start_date": "2020-01-01",
      "end_date": "2023-12-31",
      "population_size": 100,
      "generations": 50,
      "param_ranges": {
        "fast_window": {"min": 2, "max": 20},
        "slow_window": {"min": 10, "max": 60}
      }
    }
    ```
  - Response: `{ "job_id": "<uuid>", "status": "queued" }`

- `GET /api/optimization/{job_id}`
  - Description: Get optimization status and results (ranked list of parameter sets with metrics)

- `GET /api/optimization/history`
  - Description: Get past optimization jobs and top results

- `POST /api/optimization/{job_id}/cancel`
  - Description: Cancel optimization

Notes

- Optimization jobs are long-running and typically executed on the `optimization` RQ queue with longer timeouts.
- Results include performance metrics: total return, annualized return, Sharpe, max drawdown, win rate, and trades count.

## Error Responses

Current error body:

```json
{ "detail": "..." }
```

See:
- `ERROR_CODES.md`

Common statuses:
- `401` (AUTH_MISSING_TOKEN / AUTH_TOKEN_EXPIRED_OR_INVALID)
- `400` (BAD_REQUEST)
- `404` (RESOURCE_NOT_FOUND)
- `422` (VALIDATION_ERROR)
- `500` (INTERNAL_ERROR)

## Pagination

`GET /api/optimization/history` SHOULD be paginated when implemented. Follow `PAGINATION.md`.

## Rate Limiting

No explicit global API rate limit documented.
Queue capacity may effectively rate-limit job submissions.

## References

- `ERROR_CODES.md`
- `PAGINATION.md`
