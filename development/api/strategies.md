# Strategies API

Base path: `/api/strategies`

Auth: Bearer token required unless otherwise noted.

## Endpoints

- `GET /api/strategies`
  - List all strategies for the current user.
  - Response: array of `StrategyListItem` (not paginated).

- `POST /api/strategies`
  - Create a new strategy.
  - Body: `{ "name": string, "class_name": string, "description": string, "parameters": object, "code": string }`

- `GET /api/strategies/{strategy_id}`
  - Fetch a strategy by ID.

- `PUT /api/strategies/{strategy_id}`
  - Update a strategy (partial fields supported).

- `DELETE /api/strategies/{strategy_id}`
  - Delete a strategy.

- `POST /api/strategies/{strategy_id}/validate`
  - Validate strategy code and class name.

- `GET /api/strategies/{strategy_id}/code-history`
  - List stored code-history entries.

- `GET /api/strategies/{strategy_id}/code-history/{history_id}`
  - Retrieve a specific code-history entry.

- `POST /api/strategies/{strategy_id}/code-history/{history_id}/restore`
  - Restore a code-history entry to the active strategy.

- `GET /api/strategies/builtin/list`
  - List built-in strategies from `app/strategies/` (no auth required).

## Notes

- Validation is performed server-side via `validate_strategy_code`.
- Built-in strategies are discovered dynamically from the Python files in `app/strategies/`.
