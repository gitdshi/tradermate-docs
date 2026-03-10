# API Pagination Standard

This document defines the pagination standard for API endpoints that return lists.

## 1. Chosen Scheme

Use `limit/offset` pagination.

Rationale:
- Simple to implement and understand.
- Matches common SQL query patterns.
- Works well for admin-style list views.

## 2. Request Parameters

When an endpoint is paginated, it SHOULD accept:

- `limit` (int)
  - Default: 20
  - Min: 1
  - Max: 100

- `offset` (int)
  - Default: 0
  - Min: 0

Example:

```http
GET /api/strategies?limit=20&offset=0
```

## 3. Response Shape

Preferred response shape:

```json
{
  "items": [
    { "...": "..." }
  ],
  "limit": 20,
  "offset": 0,
  "total": 123
}
```

Notes:
- Some existing endpoints may currently return a bare JSON array. When updating documentation, call out the current shape and the recommended shape.

## 4. Sorting

If an endpoint supports ordering, use:
- `sort_by=<field>`
- `sort_dir=asc|desc`

If sorting is not supported, document it explicitly.
