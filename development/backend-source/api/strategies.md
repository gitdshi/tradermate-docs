# Strategies API

Base path: `/api/strategies`

Endpoints

- `GET /api/strategies`
  - Description: List user strategies
  - Query params: None (currently returns full list; pagination not implemented yet)
  - Response: JSON array of strategy list items

- `POST /api/strategies`
  - Description: Create a new strategy
  - Auth: Bearer token required
  - Body: `{ "name": string, "description": string, "code": string, "params": object }`
  - Response: Created strategy object with `id`

- `GET /api/strategies/{id}`
  - Description: Get strategy details
  - Response: Strategy object including source code and parameter schema

- `PUT /api/strategies/{id}`
  - Description: Update strategy
  - Auth: Bearer token required
  - Body: Partial strategy fields
  - Response: Updated strategy

- `DELETE /api/strategies/{id}`
  - Description: Delete user strategy
  - Auth: Bearer token required
  - Response: 204 No Content

- `GET /api/strategies/builtin`
  - Description: List built-in strategies available for quick import (e.g., Turtle Trading, Triple MA)

Notes

- Strategy code validation: server-side AST validation and safe execution checks are performed before saving or running.
- Parameter schema: strategies expose parameter names, types, min/max/default for UI builders and optimization.

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
- `404` (RESOURCE_NOT_FOUND: Strategy not found)
- `422` (VALIDATION_ERROR)

## Pagination

This endpoint is not paginated today (returns full list). If pagination is added, it should follow `PAGINATION.md`.

## Rate Limiting

No explicit global API rate limit documented.

## References

- `ERROR_CODES.md`
- `PAGINATION.md`
