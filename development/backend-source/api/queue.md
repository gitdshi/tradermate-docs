# Queue & Job Management API

Base path: `/api/queue`

Endpoints

- `GET /api/queue/stats`
  - Description: Get queue statistics (counts per queue, processing rates)
  - Response: `{ queues: { backtest: { queued: 10, started: 2, failed: 1 }, optimization: {...} }, redis_info: {...} }`

- `GET /api/queue/jobs`
  - Description: List jobs for the authenticated user
  - Query params: `?status=queued&limit=20&offset=0` (if pagination is implemented)

- `GET /api/queue/jobs/{job_id}`
  - Description: Get job details, logs, and progress

- `POST /api/queue/jobs/{job_id}/cancel`
  - Description: Cancel a queued or running job

- `DELETE /api/queue/jobs/{job_id}`
  - Description: Remove a job from storage (user action)

Notes

- Queue backend: Redis + RQ workers. Worker processes consume job queues and update status/progress in job storage.
- Result TTL: job results may be kept for a configured TTL (default 7 days) before automatic cleanup.

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

If `GET /api/queue/jobs` is paginated, it should follow `PAGINATION.md` (`limit/offset`).

## Rate Limiting

No explicit global API rate limit documented.

## References

- `ERROR_CODES.md`
- `PAGINATION.md`
