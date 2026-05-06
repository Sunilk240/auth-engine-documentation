# 06 — Error Reference

All errors follow this structure:

```json
{
  "error":   "ERROR_CODE",
  "message": "Human-readable description.",
  "status":  400
}
```

Some errors include additional fields (noted below).

---

## Authentication & App Credentials

| Code | Status | Cause | Action |
|------|--------|-------|--------|
| `TOKEN_INVALID` | 401 | Wrong `app_id`, wrong `app_secret`, or malformed JWT in `Authorization` header | Verify credentials; ensure JWT is passed correctly |
| `TOKEN_EXPIRED` | 401 | Access token is older than 30 minutes | Call `/auth/refresh` to get a new access token |
| `TOKEN_REVOKED` | 401 | Refresh token was explicitly revoked (user logged out) | Ask user to log in again |
| `TOKEN_REUSE` | 401 | An already-rotated refresh token was replayed | Session family revoked — user must log in again |
| `USER_DELETED` | 403 | User account has been soft-deleted | Do not allow session; show appropriate message |

---

## OTP Errors

| Code | Status | Cause | Action |
|------|--------|-------|--------|
| `OTP_RATE_LIMITED` | 429 | Too many OTPs sent to this contact in 10 minutes | Check `Retry-After` header; wait before allowing another send |
| `OTP_INVALID` | 400 | OTP is incorrect | Let user try again (up to 3 attempts total) |
| `OTP_LOCKED` | 429 | 3 incorrect attempts — OTP is permanently locked | Ask user to request a new OTP via `send-otp` |
| `OTP_EXPIRED` | 400 | OTP is older than 5 minutes | Ask user to request a new OTP |
| `OTP_ALREADY_USED` | 400 | OTP has already been successfully verified | Redirect user to log in again |
| `OTP_WRONG_APP` | 403 | `otp_request_id` was issued for a different `app_id` | Ensure `app_id` matches across send and verify calls |
| `OTP_WRONG_PURPOSE` | 403 | `purpose` in verify doesn't match the one used in send | Send and verify must use the same `purpose` |
| `OTP_NOT_FOUND` | 404 | `otp_request_id` does not exist | Verify the ID is correct and not cleaned up |

**Extra fields on `OTP_RATE_LIMITED` and `OTP_LOCKED`:**
```json
{
  "error":       "OTP_RATE_LIMITED",
  "message":     "Too many OTP requests. Please wait before trying again.",
  "status":      429,
  "retry_after": 342
}
```
`retry_after` is in seconds. Use this to show a countdown in your UI.

---

## License Key Errors

| Code | Status | Cause | Action |
|------|--------|-------|--------|
| `LICENSE_NOT_FOUND` | 404 | Key doesn't exist or belongs to a different app | Ask user to double-check the key they entered |
| `LICENSE_INVALID` | 400 | Key format is malformed (wrong length, bad characters) | Validate format before sending; guide user to re-enter |
| `LICENSE_REVOKED` | 403 | Key was revoked by an admin | Show "License revoked" message; contact support |
| `LICENSE_EXPIRED` | 403 | Key is past its `expires_at` date | Show expiry message; offer renewal |
| `LICENSE_NOT_ACTIVATED` | 403 | Verify called but this device was never activated | Send user through the activation flow first |
| `LICENSE_MAX_ACTIVATIONS` | 403 | All activation slots are used by other devices | Show "Activation limit reached"; offer deactivation of another device |
| `LICENSE_ALREADY_REVOKED` | 409 | Revoke called on a key that was already revoked | Idempotent — treat as success |
| `LICENSE_ALREADY_ACTIVE` | 409 | Activation slot already exists for this device | Treat as success (idempotent re-activation) |

---

## Role Management Errors

| Code | Status | Cause | Action |
|------|--------|-------|--------|
| `TENANT_NOT_FOUND` | 404 | Tenant doesn't exist or belongs to a different app | Verify `tenant_id` is correct for your `app_id` |
| `USER_NOT_FOUND` | 404 | `user_id` does not exist in this app | Verify user has completed OTP login first |
| `ROLE_NOT_FOUND` | 404 | No role exists for this user/tenant combination (on revoke) | Safe to ignore — idempotent |

---

## Validation Errors

| Code | Status | Cause |
|------|--------|-------|
| `VALIDATION_ERROR` | 400 | Missing required field or wrong format |

```json
{
  "error":   "VALIDATION_ERROR",
  "message": "Invalid request body.",
  "status":  400,
  "details": [
    { "field": "phone", "message": "Phone must be in E.164 format (e.g. +919876543210)" },
    { "field": "app_id", "message": "Must be a valid UUID" }
  ]
}
```

`details` lists every field that failed validation so you can surface them in your UI.

---

## Infrastructure Errors

| Code | Status | Cause | Action |
|------|--------|-------|--------|
| `APP_NOT_FOUND` | 404 | `app_id` UUID is not registered | Check you are using the correct `app_id` |
| `INTERNAL_ERROR` | 500 | Unexpected server-side error | Retry with exponential backoff; contact service owner if persistent |

---

## HTTP Status Code Summary

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `201` | Created (new resource) |
| `400` | Bad request (validation error, wrong OTP, etc.) |
| `401` | Authentication required or credentials are wrong |
| `403` | Authenticated but not authorized |
| `404` | Resource not found |
| `409` | Conflict (duplicate, already exists) |
| `429` | Rate limited — check `Retry-After` header |
| `500` | Server error — retry with backoff |
| `503` | Service unavailable — retry with backoff |

---

## Recommended Error Handling Pattern

```typescript
async function callAuthService(url: string, body: object) {
  const res = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });

  const data = await res.json();

  if (!res.ok) {
    switch (data.error) {
      case 'TOKEN_EXPIRED':
        // Trigger refresh flow
        await refreshAccessToken();
        // Retry original request
        break;
      case 'OTP_RATE_LIMITED':
        // Show countdown using data.retry_after
        showRetryCountdown(data.retry_after);
        break;
      case 'TOKEN_REUSE':
      case 'TOKEN_REVOKED':
      case 'USER_DELETED':
        // Force re-login
        clearSession();
        redirectToLogin();
        break;
      default:
        // Surface message to user
        showError(data.message);
    }
    return null;
  }

  return data;
}
```
