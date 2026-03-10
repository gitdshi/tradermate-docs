# API Error Responses and Error Codes

This document is the single reference for API error responses and documented error codes.

## 1. Current Error Response Format (as implemented)

Most endpoints currently raise FastAPI `HTTPException`, which produces this JSON body:

```json
{
  "detail": "Human-readable error message"
}
```

Notes:
- Clients should primarily branch on HTTP status codes.
- The "error codes" below are *documented semantics* for consistency. Not all endpoints currently return a machine-readable `code` field.

## 2. Error Code Catalog (Documented Semantics)

Each entry maps to:
- HTTP status
- Typical `detail` message (or pattern)
- Meaning and recommended client action

### 2.1 Authentication / Authorization

- `AUTH_INVALID_CREDENTIALS`
  - HTTP: `401 Unauthorized`
  - detail: e.g. "Invalid username or password"
  - Meaning: credentials are incorrect
  - Client action: prompt user to re-authenticate

- `AUTH_USER_DISABLED`
  - HTTP: `403 Forbidden`
  - detail: message contains "disabled"
  - Meaning: account disabled
  - Client action: show account disabled message; do not retry

- `AUTH_MISSING_TOKEN`
  - HTTP: `401 Unauthorized`
  - detail: e.g. "Not authenticated" / missing bearer token
  - Meaning: auth required
  - Client action: redirect to login

- `AUTH_TOKEN_EXPIRED_OR_INVALID`
  - HTTP: `401 Unauthorized`
  - detail: e.g. "Could not validate credentials" / token invalid
  - Meaning: access token expired or invalid
  - Client action: attempt refresh; otherwise re-login

### 2.2 Resource / Domain

- `RESOURCE_NOT_FOUND`
  - HTTP: `404 Not Found`
  - detail: e.g. "User not found" / "Strategy not found"
  - Meaning: resource id does not exist
  - Client action: show not found; stop retry

- `VALIDATION_ERROR`
  - HTTP: `422 Unprocessable Entity`
  - detail: FastAPI validation payload (field errors)
  - Meaning: request schema invalid
  - Client action: fix client-side request; do not retry without changes

- `BAD_REQUEST`
  - HTTP: `400 Bad Request`
  - detail: domain validation failure (ValueError / PermissionError in business rules)
  - Meaning: request is understood but invalid
  - Client action: show message; fix input

### 2.3 Server / Infrastructure

- `INTERNAL_ERROR`
  - HTTP: `500 Internal Server Error`
  - detail: may vary
  - Meaning: unexpected server-side failure
  - Client action: show generic error; allow retry with backoff

## 3. Recommended Future-Compatible Format (Optional)

If/when the backend adds structured error codes, this format is recommended:

```json
{
  "error": {
    "code": "AUTH_INVALID_CREDENTIALS",
    "message": "Invalid username or password",
    "request_id": "<optional>"
  }
}
```

Until then, treat the code catalog in this document as the SSOT for error semantics.
