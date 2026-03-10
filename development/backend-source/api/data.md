# Market Data API

Base path: `/api/data`

Endpoints

- `GET /api/data/symbols`
  - Description: List available symbols (supports basic filtering + pagination)
  - Query params:
    - `exchange` (optional): `SZSE|SSE|...`
    - `keyword` (optional): search by symbol or name
    - `limit` (default 100, max 1000)
    - `offset` (default 0)
  - Response: JSON array of symbol objects

- `GET /api/data/history/{vt_symbol}`
  - Description: Retrieve OHLCV historical data
  - Query params: `?start_date=2020-01-01&end_date=2023-12-31&interval=daily`
  - Response: JSON array of OHLC bars

- `GET /api/data/indicators/{vt_symbol}`
  - Description: Compute indicators for a symbol
  - Query params: `?start_date=2020-01-01&end_date=2023-12-31&indicators=ma_10,ma_20,ma_60`
  - Response: JSON payload of computed indicator series

- `GET /api/data/overview`
  - Description: Market overview statistics

- `GET /api/data/sectors`
  - Description: Sector information

- `GET /api/data/exchanges`
  - Description: Exchange-level stock groupings (SSE, SZSE, BSE)

Notes

- Data source: Tushare DB or configured market data DB. Ensure ingestion is up-to-date for accurate backtests.
- `GET /symbols` supports `limit/offset`.

## Error Responses

Current error body:

```json
{ "detail": "..." }
```

See:
- `ERROR_CODES.md`

Common statuses:
- `400` (BAD_REQUEST)
- `422` (VALIDATION_ERROR)
- `500` (INTERNAL_ERROR)

## Pagination

- `GET /api/data/symbols` supports `limit/offset`.
- Other endpoints return time-series arrays and are not paginated.

Standard reference:
- `PAGINATION.md`

## Rate Limiting

No explicit global API rate limit documented.
Note: upstream Tushare API has its own rate limits (see DataSync docs).

## References

- `ERROR_CODES.md`
- `PAGINATION.md`
