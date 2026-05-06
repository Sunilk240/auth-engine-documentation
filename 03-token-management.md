# 03 — Token Management

After a user logs in, your backend holds two tokens. This document explains how to use them.

---

## The Two Tokens

| Token | Type | Expires | Stored Where | Purpose |
|-------|------|---------|-------------|---------|
| `access_token` | JWT (RS256) | 30 minutes | Frontend memory or short-lived storage | Sent on every API call as `Bearer` header |
| `refresh_token` | Opaque string | 30 days (absolute: 1 year) | HttpOnly cookie or secure server-side store | Used only to get a new access token |

---

## Verifying the Access Token (In Your Backend)

Your backend verifies the JWT on every protected API call using the `JWT_PUBLIC_KEY`. See [01 — Getting Started](./01-getting-started.md) for the copy-paste middleware.

**JWT payload structure:**
```json
{
  "sub":       "user-uuid",
  "app_id":    "my-product",
  "iss":       "https://auth-engine-efb8.onrender.com",
  "aud":       "my-product",
  "iat":       1746268800,
  "exp":       1746270600,
  "tenant_id": "tenant-uuid-or-null",
  "roles":     ["admin", "viewer"]
}
```

| Claim | Description |
|-------|-------------|
| `sub` | The user's stable UUID — use this as your user identifier |
| `app_id` | Your app slug (not the UUID) — verify this matches your `AUTH_APP_SLUG` |
| `tenant_id` | The tenant this user belongs to (null if no role assigned) |
| `roles` | Array of role name strings for this tenant (e.g. `["admin"]`) |
| `aud` | Your app slug — must match `AUTH_APP_SLUG` in middleware |

---

## GET /auth/me — Fetch User Claims from Token

Use this to get user details from a valid access token without decoding it yourself.

```
GET /auth/me
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "user_id":   "user-uuid",
  "app_id":    "my-product",
  "tenant_id": "tenant-uuid-or-null",
  "roles":     ["admin"]
}
```

> **Note:** `app_id` is the app **slug** (e.g. `my-product`), not the UUID.
> `roles` is an array of role name strings. It reflects the state at the last login or token refresh — role changes assigned via `/auth/roles/assign` appear on the **next** token refresh.
> There are no `phone`, `email`, or timestamp fields — all data is read directly from the JWT claims.

**cURL:**
```bash
curl https://auth-engine-efb8.onrender.com/auth/me \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

**PowerShell:**
```powershell
$me = Invoke-RestMethod `
  -Uri "https://auth-engine-efb8.onrender.com/auth/me" `
  -Headers @{ Authorization = "Bearer $($verifyRes.access_token)" }

$me | Format-List
```

---

## POST /auth/refresh — Renew Access Token

Call this when the access token expires (you receive a `TOKEN_EXPIRED` error on a protected call).

```
POST /auth/refresh
```

**Request body:**
```json
{
  "app_id":        "YOUR_APP_ID",
  "app_secret":    "YOUR_APP_SECRET",
  "refresh_token": "CURRENT_REFRESH_TOKEN"
}
```

**Response (200 OK):**
```json
{
  "access_token":  "<new JWT>",
  "refresh_token": "<new refresh token>"
}
```

> **Token Rotation:** The old `refresh_token` is **immediately invalidated** after this call.
> Always replace both tokens in your storage. If you replay an old refresh token, the entire
> session family is revoked as a security measure.

**Node.js Example:**
```typescript
app.post('/api/auth/refresh', async (req, res) => {
  const refreshToken = req.cookies.refresh_token; // from HttpOnly cookie
  if (!refreshToken) return res.status(401).json({ error: 'No refresh token' });

  const authRes = await fetch(`${process.env.AUTH_SERVICE_URL}/auth/refresh`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      app_id:        process.env.AUTH_APP_ID,
      app_secret:    process.env.AUTH_APP_SECRET,
      refresh_token: refreshToken,
    }),
  });

  const data = await authRes.json();
  if (!authRes.ok) {
    // Clear cookie and force re-login
    res.clearCookie('refresh_token');
    return res.status(401).json({ error: 'SESSION_EXPIRED' });
  }

  // Set the new refresh token
  res.cookie('refresh_token', data.refresh_token, {
    httpOnly: true, secure: true, sameSite: 'strict',
    maxAge: 30 * 24 * 60 * 60 * 1000,
  });

  res.json({ access_token: data.access_token });
});
```

**cURL:**
```bash
curl -X POST https://auth-engine-efb8.onrender.com/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{
    "app_id":        "YOUR_APP_ID",
    "app_secret":    "YOUR_APP_SECRET",
    "refresh_token": "CURRENT_REFRESH_TOKEN"
  }'
```

**PowerShell:**
```powershell
$refreshRes = Invoke-RestMethod `
  -Uri "https://auth-engine-efb8.onrender.com/auth/refresh" `
  -Method POST `
  -ContentType "application/json" `
  -Body (@{
    app_id        = "YOUR_APP_ID"
    app_secret    = "YOUR_APP_SECRET"
    refresh_token = $verifyRes.refresh_token
  } | ConvertTo-Json)

Write-Host "New Access Token:  $($refreshRes.access_token)"
Write-Host "New Refresh Token: $($refreshRes.refresh_token)"
```

---

## POST /auth/revoke — Logout

Invalidates the refresh token (and its entire rotation family) immediately.
Call this when the user logs out.

```
POST /auth/revoke
```

**Request body:**
```json
{
  "app_id":        "YOUR_APP_ID",
  "app_secret":    "YOUR_APP_SECRET",
  "refresh_token": "REFRESH_TOKEN_TO_REVOKE"
}
```

**Response (200 OK):**
```json
{ "success": true }
```

This call is **idempotent** — calling it with an already-revoked token still returns `200 OK`.

**Node.js Logout Route:**
```typescript
app.post('/api/auth/logout', async (req, res) => {
  const refreshToken = req.cookies.refresh_token;
  if (refreshToken) {
    await fetch(`${process.env.AUTH_SERVICE_URL}/auth/revoke`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        app_id:        process.env.AUTH_APP_ID,
        app_secret:    process.env.AUTH_APP_SECRET,
        refresh_token: refreshToken,
      }),
    });
  }
  res.clearCookie('refresh_token');
  res.json({ success: true });
});
```

**cURL:**
```bash
curl -X POST https://auth-engine-efb8.onrender.com/auth/revoke \
  -H "Content-Type: application/json" \
  -d '{
    "app_id":        "YOUR_APP_ID",
    "app_secret":    "YOUR_APP_SECRET",
    "refresh_token": "REFRESH_TOKEN_TO_REVOKE"
  }'
```

**PowerShell:**
```powershell
Invoke-RestMethod `
  -Uri "https://auth-engine-efb8.onrender.com/auth/revoke" `
  -Method POST `
  -ContentType "application/json" `
  -Body (@{
    app_id        = "YOUR_APP_ID"
    app_secret    = "YOUR_APP_SECRET"
    refresh_token = $verifyRes.refresh_token
  } | ConvertTo-Json)
```

---

## Recommended Token Storage Strategy

| Scenario | Access Token | Refresh Token |
|---------|-------------|--------------|
| Web app (recommended) | In-memory JS variable | HttpOnly cookie (set by your backend) |
| Web app (alternative) | `sessionStorage` | HttpOnly cookie |
| Mobile app | Secure storage (iOS Keychain / Android Keystore) | Secure storage |
| Server-to-server | Not needed — use app_secret directly | Not needed |

**Never store tokens in:**
- `localStorage` (vulnerable to XSS)
- URL parameters (appear in logs)
- `document.cookie` from JavaScript (use HttpOnly)

---

## Common Errors

| Error Code | Status | Cause |
|-----------|--------|-------|
| `TOKEN_INVALID` | 401 | Malformed JWT or wrong `app_secret` |
| `TOKEN_EXPIRED` | 401 | Access token is older than 30 minutes — refresh it |
| `TOKEN_REVOKED` | 401 | Refresh token was explicitly revoked (user logged out) |
| `TOKEN_REUSE` | 401 | Old rotated refresh token was replayed — full family revoked |
| `USER_DELETED` | 403 | User account was soft-deleted |

---

## Next Steps

- **[04 — License Keys](./04-license-keys.md)** — Software license management
- **[05 — Roles and Tenants](./05-roles-and-tenants.md)** — Multi-tenant role assignment
