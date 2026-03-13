# Market Data API

Base path: `/api/data`

Auth: Optional. These endpoints accept a token if provided, but do not require login.

## Endpoints

- `GET /api/data/symbols`
  - List available symbols.
  - Query: `exchange` (SZSE/SSE/BSE), `keyword`, `limit` (<=1000), `offset`.

- `GET /api/data/history/{vt_symbol}`
  - Historical OHLC bars.
  - Query: `start_date`, `end_date`, `interval` (daily|weekly|monthly).

- `GET /api/data/indicators/{vt_symbol}`
  - Computed indicators.
  - Query: `start_date`, `end_date`, `indicators` (comma-separated, e.g. `ma_10,ma_20,ma_60`).

- `GET /api/data/overview`
  - Market overview stats.

- `GET /api/data/sectors`
  - Sector summaries.

- `GET /api/data/exchanges`
  - Exchange-level groupings.

- `GET /api/data/indexes`
  - Available benchmark index codes (AkShare).

- `GET /api/data/symbols-by-filter`
  - Filtered symbol list for bulk backtests.
  - Query: `industry`, `exchange`, `limit` (<=2000).

## Notes

- Data comes from Tushare and AkShare sync pipelines. Ensure data sync has been initialized.
- `vt_symbol` uses the format like `000001.SZ`.
