# Unified Auth Engine — Developer Documentation

> **This is public-facing documentation.** No credentials, secrets, or internal implementation details are included.
> Register at the portal to get your credentials.

---

## What This Service Does

A hosted, multi-tenant authentication engine. You call it from your backend — it validates users, issues tokens, and manages sessions. You do not need to build or host any auth infrastructure.

**What you get:**
- Phone or email OTP login (no passwords)
- Short-lived RS256 JWT access tokens
- Rotating refresh tokens with replay detection
- Multi-tenant role management embedded in JWTs
- Software license key issuance and device activation
- Immutable audit log of every auth event

**Your responsibilities:**
- Call these APIs from your **backend only** (not from frontend or mobile directly)
- Store the JWT access token and pass it as a `Bearer` header on your own API calls
- Store the refresh token securely (HttpOnly cookie recommended)

---

## How to Register

**[→ Open Developer Portal](https://developer-portal-8poz.onrender.com)**

1. Sign in with your phone or email (OTP)
2. Click **"Create App"** — give it a name and slug (e.g. `attendance-app`)
3. Copy your `APP_ID`, `APP_SECRET`, `APP_SLUG`, and `JWT_PUBLIC_KEY`
4. APP_SECRET is shown **once** — save it immediately

---

## Navigation

| Document | What It Covers |
|----------|---------------|
| [01 — Getting Started](./01-getting-started.md) | Registration, credentials, environment, middleware |
| [02 — OTP Authentication](./02-otp-authentication.md) | Phone and email login flows with full code examples |
| [03 — Token Management](./03-token-management.md) | JWT verification, `/auth/me`, refresh, logout |
| [04 — License Keys](./04-license-keys.md) | Generate, activate, verify, revoke license keys |
| [05 — Roles and Tenants](./05-roles-and-tenants.md) | Tenant model, role assignment, roles in JWT |
| [06 — Error Reference](./06-error-reference.md) | Every error code, HTTP status, and how to handle it |
| [07 — Security Checklist](./07-security-checklist.md) | Pre-launch checklist before going live |

---

## Request Flow

```
Your App Frontend / Mobile
        │
        │  Sends OTP input to YOUR backend
        ▼
Your App Backend
        │  POST /auth/send-otp    { app_id, app_secret, phone }
        │  POST /auth/verify-otp  { app_id, app_secret, otp_request_id, otp }
        │  POST /auth/refresh     { app_id, app_secret, refresh_token }
        ▼
Auth Engine  ──────────────────────────────▶  Returns JWT (RS256)
                                                      │
                                                      │  Bearer <token>
                                                      ▼
                                              Your App Backend
                                              (verifies with JWT_PUBLIC_KEY locally)
```

**Your credentials:**

| Credential | Held by | Purpose |
|-----------|---------|---------|
| `APP_ID` + `APP_SECRET` | Your backend server | Making auth API calls |
| `APP_SLUG` | Your backend server | Verifying JWT audience claim |
| `JWT_PUBLIC_KEY` | Your backend server | Verifying token signatures locally |

None of these ever go to the frontend or mobile app.

---

## Base URL

```
https://auth-engine-efb8.onrender.com
```

---

## API Conventions

- All requests: `Content-Type: application/json`
- All responses: JSON body
- Authentication: `app_id` + `app_secret` in the **request body** (not headers)
- Errors always include `error` (machine-readable code) and `message` (human readable)

```json
{
  "error": "OTP_EXPIRED",
  "message": "This OTP has expired. Please request a new one.",
  "status": 400
}
```
