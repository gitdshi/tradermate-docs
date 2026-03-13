# Authentication API

Base path: `/api/auth`

Authentication is JWT-based. Most protected endpoints require an `Authorization: Bearer <access_token>` header.

## Endpoints

- `POST /api/auth/register`
  - Register a new user.
  - Body: `{ "username": string, "email": string, "password": string }`
  - Response: `User`

- `POST /api/auth/login`
  - Login and obtain access + refresh tokens.
  - Body: `{ "username": string, "password": string }`
  - Response: `{ "access_token", "refresh_token", "token_type", "expires_in", "must_change_password" }`

- `POST /api/auth/refresh`
  - Exchange refresh token for a new access token.
  - Body: `{ "refresh_token": string }`
  - Response: `{ "access_token", "refresh_token", "token_type", "expires_in", "must_change_password" }`

- `GET /api/auth/me`
  - Get current user profile.

- `POST /api/auth/change-password`
  - Change password for current user (required if `must_change_password=true`).
  - Body: `{ "current_password": string, "new_password": string }`
  - Response: `{ "detail": "Password changed successfully" }`

## Notes

- Token TTL is configured via `ACCESS_TOKEN_EXPIRE_MINUTES` and `REFRESH_TOKEN_EXPIRE_DAYS`.
- If `must_change_password=true`, the portal will force a password change before other pages work.
