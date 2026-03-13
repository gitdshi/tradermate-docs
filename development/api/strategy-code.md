# Strategy Code Utility API

Base path: `/api/strategy-code`

Auth: Not required.

## Endpoints

- `POST /api/strategy-code/parse`
  - Parse Python strategy code and return classes/parameters.
  - Body: `{ "content": "<python source>" }`

- `POST /api/strategy-code/lint`
  - Syntax + basic import checks.
  - Body: `{ "content": "<python source>" }`
  - Response: `{ "diagnostics": [{ line, col, severity, message }] }`

- `POST /api/strategy-code/lint/pyright`
  - Runs `pyright` if installed in PATH.
  - Body: `{ "content": "<python source>" }`
  - Response: `{ "pyright": <raw json>, "diagnostics": [...] }`

## Notes

- This API does not persist code; it is intended for the portal editor.
- `pyright` must be installed separately (e.g., `npm i -g pyright`).
