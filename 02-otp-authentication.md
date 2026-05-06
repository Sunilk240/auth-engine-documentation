# 02 — OTP Authentication

Users log in by receiving a one-time code on their phone or email and submitting it. Your backend orchestrates both steps.

---

## The Flow

```
User enters phone/email on your UI
        │
        ▼
Your Backend  ──POST /auth/send-otp──▶  Auth Service
                                              │ sends OTP to user's phone/email
                                              │ returns otp_request_id
        ◀─────────────────────────────────────┘
        │
        │  pass otp_request_id to your frontend
        ▼
User sees OTP input, enters code
        │
        ▼
Your Backend  ──POST /auth/verify-otp─▶  Auth Service
                                              │ validates OTP
                                              │ returns access_token + refresh_token
        ◀─────────────────────────────────────┘
        │
        │  store tokens, return session to user
        ▼
     User is logged in
```

---

## Step 1 — Send OTP

### Phone (SMS)

```
POST /auth/send-otp
```

**Request body:**
```json
{
  "app_id":     "YOUR_APP_ID",
  "app_secret": "YOUR_APP_SECRET",
  "phone":      "+919876543210",
  "purpose":    "LOGIN"
}
```

**Response (200 OK):**
```json
{
  "otp_request_id": "req-uuid-here",
  "expires_at":     "2026-05-03T10:10:00.000Z"
}
```

### Email

```
POST /auth/send-otp
```

**Request body:**
```json
{
  "app_id":     "YOUR_APP_ID",
  "app_secret": "YOUR_APP_SECRET",
  "email":      "user@example.com",
  "purpose":    "LOGIN"
}
```

> Provide **either** `phone` or `email` — never both, never neither.

### Phone format
Phones must be in **E.164 format**: `+` followed by country code and number, no spaces.
- ✅ `+919876543210`
- ✅ `+14155552671`
- ❌ `09876543210`
- ❌ `+91 98765 43210`

### Purpose values

| Purpose | When to use |
|---------|-------------|
| `LOGIN` | Standard login / signup |
| `PHONE_CHANGE` | Verifying a new phone number |
| `EMAIL_VERIFY` | Verifying an email address |
| `PASSWORD_RESET` | Initiating a password reset |

---

## Step 2 — Verify OTP

```
POST /auth/verify-otp
```

**Request body:**
```json
{
  "app_id":         "YOUR_APP_ID",
  "app_secret":     "YOUR_APP_SECRET",
  "otp_request_id": "req-uuid-from-step-1",
  "otp":            "483920",
  "purpose":        "LOGIN"
}
```

**Response (200 OK):**
```json
{
  "access_token":  "<JWT>",
  "refresh_token": "<opaque token>",
  "user_id":       "user-uuid",
  "is_new_user":   true
}
```

- `is_new_user: true` → this is the user's first login. Use this to trigger onboarding in your product.
- `is_new_user: false` → returning user.
- Store `refresh_token` securely. Recommended: HttpOnly cookie, server-side session store.
- `access_token` expires in 30 minutes. Use the refresh flow to renew it.

---

## OTP Rules

| Rule | Value |
|------|-------|
| OTP length | 6 digits |
| OTP expiry | 5 minutes |
| Max wrong attempts | 3 (locked after 3rd wrong attempt) |
| Lock cooldown | Returned as `retry_after` (seconds) in the `OTP_LOCKED` error |
| Rate limit (send) | 3 OTPs per phone/email per 10 minutes |

---

## Code Examples

### Node.js / Express Backend (Phone Login)

```typescript
// POST /api/auth/send-otp — your backend route
app.post('/api/auth/send-otp', async (req, res) => {
  const { phone } = req.body;

  const authRes = await fetch(`${process.env.AUTH_SERVICE_URL}/auth/send-otp`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      app_id:     process.env.AUTH_APP_ID,
      app_secret: process.env.AUTH_APP_SECRET,
      phone,
      purpose: 'LOGIN',
    }),
  });

  const data = await authRes.json();
  if (!authRes.ok) return res.status(authRes.status).json(data);

  // Return otp_request_id to your frontend
  res.json({ otp_request_id: data.otp_request_id, expires_at: data.expires_at });
});
```

```typescript
// POST /api/auth/verify-otp — your backend route
app.post('/api/auth/verify-otp', async (req, res) => {
  const { otp_request_id, otp } = req.body;

  const authRes = await fetch(`${process.env.AUTH_SERVICE_URL}/auth/verify-otp`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      app_id:         process.env.AUTH_APP_ID,
      app_secret:     process.env.AUTH_APP_SECRET,
      otp_request_id,
      otp,
      purpose: 'LOGIN',
    }),
  });

  const data = await authRes.json();
  if (!authRes.ok) return res.status(authRes.status).json(data);

  // Set refresh token as HttpOnly cookie (recommended)
  res.cookie('refresh_token', data.refresh_token, {
    httpOnly: true,
    secure:   true,
    sameSite: 'strict',
    maxAge:   30 * 24 * 60 * 60 * 1000, // 30 days
  });

  // Return access token to frontend
  res.json({
    access_token: data.access_token,
    user_id:      data.user_id,
    is_new_user:  data.is_new_user,
  });
});
```

---

### cURL Examples

**Send OTP (phone):**
```bash
curl -X POST https://auth-engine-efb8.onrender.com/auth/send-otp \
  -H "Content-Type: application/json" \
  -d '{
    "app_id":     "YOUR_APP_ID",
    "app_secret": "YOUR_APP_SECRET",
    "phone":      "+919876543210",
    "purpose":    "LOGIN"
  }'
```

**Send OTP (email):**
```bash
curl -X POST https://auth-engine-efb8.onrender.com/auth/send-otp \
  -H "Content-Type: application/json" \
  -d '{
    "app_id":     "YOUR_APP_ID",
    "app_secret": "YOUR_APP_SECRET",
    "email":      "user@example.com",
    "purpose":    "LOGIN"
  }'
```

**Verify OTP:**
```bash
curl -X POST https://auth-engine-efb8.onrender.com/auth/verify-otp \
  -H "Content-Type: application/json" \
  -d '{
    "app_id":         "YOUR_APP_ID",
    "app_secret":     "YOUR_APP_SECRET",
    "otp_request_id": "PASTE_OTP_REQUEST_ID_HERE",
    "otp":            "483920",
    "purpose":        "LOGIN"
  }'
```

---

### PowerShell Examples

**Send OTP:**
```powershell
$sendRes = Invoke-RestMethod `
  -Uri "https://auth-engine-efb8.onrender.com/auth/send-otp" `
  -Method POST `
  -ContentType "application/json" `
  -Body (@{
    app_id     = "YOUR_APP_ID"
    app_secret = "YOUR_APP_SECRET"
    phone      = "+919876543210"
    purpose    = "LOGIN"
  } | ConvertTo-Json)

Write-Host "OTP Request ID: $($sendRes.otp_request_id)"
Write-Host "Expires At:     $($sendRes.expires_at)"
```

**Verify OTP:**
```powershell
$verifyRes = Invoke-RestMethod `
  -Uri "https://auth-engine-efb8.onrender.com/auth/verify-otp" `
  -Method POST `
  -ContentType "application/json" `
  -Body (@{
    app_id         = "YOUR_APP_ID"
    app_secret     = "YOUR_APP_SECRET"
    otp_request_id = $sendRes.otp_request_id
    otp            = "483920"
    purpose        = "LOGIN"
  } | ConvertTo-Json)

Write-Host "Access Token:  $($verifyRes.access_token)"
Write-Host "Refresh Token: $($verifyRes.refresh_token)"
Write-Host "User ID:       $($verifyRes.user_id)"
Write-Host "Is New User:   $($verifyRes.is_new_user)"
```

---

## Common Errors

| Error Code | Status | Cause | What To Do |
|-----------|--------|-------|-----------|
| `TOKEN_INVALID` | 401 | Wrong `app_id` or `app_secret` | Check your credentials |
| `OTP_RATE_LIMITED` | 429 | Too many OTPs sent to this contact | Wait `retry_after` seconds then retry |
| `OTP_INVALID` | 400 | Wrong OTP entered | Let user try again (up to 3 attempts) |
| `OTP_LOCKED` | 429 | 3 wrong attempts — OTP is locked | Show countdown using `retry_after` seconds; ask user to request a new OTP |
| `OTP_EXPIRED` | 400 | OTP is older than 5 minutes | Ask user to request a new OTP |
| `OTP_WRONG_APP` | 403 | `otp_request_id` belongs to a different app | Verify you're sending the correct `app_id` |
| `OTP_ALREADY_USED` | 400 | OTP already successfully verified | Redirect user to login again |
| `VALIDATION_ERROR` | 400 | Missing or malformed field | Check phone format (E.164), all required fields |

---

## Next Steps

- **[03 — Token Management](./03-token-management.md)** — Handle JWT verification, refresh, and logout
