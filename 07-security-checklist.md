# 07 — Security Checklist

Use this checklist before going live. Each item is a mandatory check, not a suggestion.

---

## Credentials

- [ ] `APP_ID` and `APP_SECRET` are stored in server-side environment variables (not in code)
- [ ] `APP_SECRET` is **not** committed to any Git repository (check `.gitignore`)
- [ ] `APP_SECRET` is **not** present in any log file, error trace, or monitoring dashboard
- [ ] `JWT_PUBLIC_KEY` is stored in a server-side environment variable
- [ ] No credentials are passed from your frontend or mobile app to the auth service — all calls go through your backend
- [ ] If `APP_SECRET` was ever logged or exposed, it has been rotated via the service owner

---

## Token Storage

- [ ] `refresh_token` is stored in an **HttpOnly cookie** (not accessible to JavaScript)
- [ ] `access_token` is stored in memory only (not in `localStorage` or `sessionStorage`)
- [ ] Both tokens are transmitted only over HTTPS
- [ ] Cookie flags are set: `Secure`, `HttpOnly`, `SameSite=Strict` (or `Lax` for cross-site flows)
- [ ] Token refresh is handled server-side — not triggered from the browser/client directly

---

## API Usage

- [ ] Your backend verifies the JWT signature using `JWT_PUBLIC_KEY` on every protected route
- [ ] Your backend checks `aud` claim matches your auth service URL (prevents token misuse from other services)
- [ ] Your backend checks `app_id` claim matches your own `AUTH_APP_ID`
- [ ] Role checks are done in your backend — never trust role data sent from the client
- [ ] Your backend returns `401` (not `403`) when a token is missing or expired, so clients know to refresh

---

## User-Facing Security

- [ ] OTP entry UI has a countdown showing time remaining (5 minutes)
- [ ] After 3 wrong OTP attempts, UI prompts user to request a new OTP (don't let them keep trying)
- [ ] Logout calls `POST /auth/revoke` — not just clearing the local token
- [ ] Re-login is required after password change, email change, or suspicious activity detection

---

## License Key Security (if using license keys)

- [ ] License key generation is an admin-only action in your backend — not accessible from client apps
- [ ] License keys are delivered to customers securely (email with access control, not plain URL parameter)
- [ ] `device_id` is generated once on first install and stored persistently — not derived from changeable hardware IDs
- [ ] Verify is called on every app launch — not cached client-side to skip the check

---

## Infrastructure

- [ ] Auth service is accessible only over HTTPS (HTTP redirects to HTTPS or is blocked)
- [ ] `HEALTH_SECRET` header is set so `/health` endpoint details are not publicly visible
- [ ] In production: `ADMIN_PORT` is set and the admin port (3001) is NOT publicly exposed
- [ ] Database connection uses SSL (`rejectUnauthorized: true` in production)
- [ ] All environment variables are set in your deployment platform (Railway, Render, etc.) — not hardcoded

---

## Operational

- [ ] You have the `JWT_PUBLIC_KEY` backed up — if the auth service key is rotated, you must update your backends
- [ ] You know how to rotate `APP_SECRET` if it is compromised (contact service owner → `rotate-secret` → update env vars → redeploy)
- [ ] Monitoring is set up on your auth service URL (uptime check with `HEALTH_SECRET` header)
- [ ] A maintenance schedule exists for cleaning up expired tokens and OTP records

---

## Pre-Launch Test Scenarios

Run these manually before going live:

| Scenario | Expected Result |
|---------|----------------|
| Send OTP with valid phone | 200 + `otp_request_id` |
| Verify with wrong OTP 3 times | 3× `OTP_INVALID`, then `OTP_LOCKED` |
| Verify with expired OTP | `OTP_EXPIRED` |
| Call protected route with valid JWT | 200 |
| Call protected route with expired JWT | 401 `TOKEN_EXPIRED` |
| Refresh with valid token | 200 + new tokens |
| Refresh with old (rotated) token | 401 `TOKEN_REUSE` |
| Logout then refresh | 401 `TOKEN_REVOKED` |
| Activate license on 2 devices with `max_activations: 1` | 2nd device gets `LICENSE_MAX_ACTIVATIONS` |
| Verify license on unactivated device | `LICENSE_NOT_ACTIVATED` |

---

## If Something Goes Wrong

| Situation | Immediate Action |
|-----------|----------------|
| `APP_SECRET` exposed in logs/code | Contact service owner → rotate-secret immediately |
| Suspicious login activity | Revoke all refresh tokens for the user; require re-login |
| JWT public key lost | Contact service owner for the public key — private key is never shared |
| Mass token invalidation needed | Contact service owner — all app tokens can be invalidated by deactivating the app and re-registering |
