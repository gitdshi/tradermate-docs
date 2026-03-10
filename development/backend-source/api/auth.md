# Authentication API

Base path: `/api/auth`

Authentication is JWT-based. Most protected endpoints require an `Authorization: Bearer <access_token>` header.

Endpoints

- `POST /api/auth/register`
  - Description: Register a new user
  - Body (application/json): `{ "username": string, "email": string, "password": string }`
  - Response: 201 Created, user profile (id, username, email)

- `POST /api/auth/login`
  - Description: Login and obtain access + refresh tokens
  - Body: `{ "username": string, "password": string }`
  - Response: `{ "access_token": string, "refresh_token": string, "token_type": "bearer" }`

- `POST /api/auth/refresh`
  - Description: Exchange refresh token for a new access token
  - Body: `{ "refresh_token": string }`
  - Response: `{ "access_token": string, "refresh_token": string }`

- `GET /api/auth/me`
  - Description: Get current authenticated user profile
  - Auth: Bearer token required
  - Response: `{ "id": number, "username": string, "email": string }`

Notes

- Tokens: `access_token` short-lived (~30 minutes), `refresh_token` longer (days). Configure via environment variables.
- Use HTTPS in production to protect credentials and tokens.

## Error Responses

This API currently uses FastAPI default error bodies:

```json
{ "detail": "..." }
```

Documented error semantics and codes:
- See `ERROR_CODES.md`

Common statuses for this module:
- `400` (BAD_REQUEST): invalid input (e.g. username/email already exists)
- `401` (AUTH_INVALID_CREDENTIALS / AUTH_TOKEN_EXPIRED_OR_INVALID)
- `403` (AUTH_USER_DISABLED)
- `404` (RESOURCE_NOT_FOUND)
- `422` (VALIDATION_ERROR)

## Pagination

Not applicable.

## Rate Limiting

No explicit global API rate-limit is documented at the application layer yet.
If a gateway / proxy adds rate limiting, it should be documented here.

## References

- `ERROR_CODES.md`
- `PAGINATION.md`
