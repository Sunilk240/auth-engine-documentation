# 01 — Getting Started

## Step 1 — Register at the Developer Portal

**[→ Open Developer Portal](https://developer-portal-8poz.onrender.com)**

1. Enter your phone number or email address
2. Enter the 6-digit OTP you receive
3. You are now signed in — no password needed

---

## Step 2 — Create Your App

Once signed in to the portal:

1. Click **"Create App"** on the dashboard
2. Fill in:
   - **App Name** — e.g. `My Attendance App` (display name)
   - **Slug** — e.g. `attendance-app` (URL-safe, lowercase, no spaces — this becomes your `APP_SLUG`)
3. Click **Create**
4. Your credentials appear **once** — copy them immediately:

```
APP_ID      = xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
APP_SECRET  = <64 hex characters — shown only once>
APP_SLUG    = attendance-app
```

> ⚠️ **APP_SECRET is shown only once.** If you lose it, go to your app in the portal and click **"Rotate Secret"** — the old secret is immediately invalidated and a new one is issued.

---

## Step 3 — Get Your JWT Public Key

In the portal, go to **🔑 Keys** in the sidebar.

You'll see a base64-encoded key — copy it with one click:

```
LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0t...  (long base64 string)
```

!!! info "Why base64?"
    The key is stored as a single-line base64 string — safe for env vars on any platform (Render, Railway, `.env` files). Your backend decodes it at startup.

---

## Step 4 — Add Credentials to Your Backend

Add these to your server's environment variables (`.env`, Render dashboard, Railway, etc.):

```env
# Auth Engine
AUTH_ENGINE_URL=https://auth-engine-efb8.onrender.com
AUTH_APP_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AUTH_APP_SECRET=<your 64-char secret>
AUTH_APP_SLUG=attendance-app
JWT_PUBLIC_KEY=LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0t...  # base64 from Keys page
```

!!! warning "Security rules"
    - ✅ Store in **server-side** environment variables only
    - ✅ Pass `AUTH_APP_ID` and `AUTH_APP_SECRET` only in backend-to-backend calls
    - ❌ Never put these in frontend JS, mobile bundles, or client-side code
    - ❌ Never commit to Git — add `.env` to `.gitignore`
    - ❌ Never log them

---

## Step 5 — Understand the Architecture

=== "Request Flow"

    ```
    Browser / Mobile App
         │  Calls YOUR backend (never the auth engine directly)
         ▼
    Your Backend                         Auth Engine
         │─── POST /auth/send-otp ──────▶│
         │◀─── { otp_request_id } ───────│
         │                               │ (sends OTP to user's phone/email)
         │
         │  (user enters OTP)
         │
         │─── POST /auth/verify-otp ────▶│
         │◀─── { access_token,           │
         │       refresh_token }  ────────│
         │
         │  Stores refresh_token in HttpOnly cookie
         │  Returns access_token to browser (in memory only)
    ```

=== "Token Verification"

    ```
    Browser / Mobile App
         │  Authorization: Bearer <access_token>
         ▼
    Your Backend
         │  jwt.verify(token, JWT_PUBLIC_KEY)  ← local, no network call
         │  Gets: user_id, roles, tenant_id from JWT payload
         ▼
    Protected resource returned
    ```

!!! tip "No DB hit per request"
    Once a token is issued, your backend verifies it **locally** using the public key.
    No call to the auth engine is needed on every request.


---

## Step 6 — Add Auth Middleware to Your Backend

Copy this into your Node.js/Express backend. Requires only the `jsonwebtoken` package.

```typescript
// authMiddleware.ts
import jwt from 'jsonwebtoken';
import type { Request, Response, NextFunction } from 'express';

// Decode base64 key at startup — zero overhead per request
const PUBLIC_KEY = Buffer.from(process.env.JWT_PUBLIC_KEY!, 'base64').toString('utf8');
const APP_SLUG   = process.env.AUTH_APP_SLUG!;

export function requireAuth(req: Request, res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'TOKEN_INVALID', message: 'Missing Bearer token.' });
  }

  const token = header.slice(7);
  try {
    const payload = jwt.verify(token, PUBLIC_KEY, {
      algorithms:     ['RS256'],
      clockTolerance: 30,
    }) as any;

    // Token must be issued for YOUR specific app
    if (payload.aud !== APP_SLUG) {
      return res.status(401).json({ error: 'TOKEN_INVALID', message: 'Token not issued for this app.' });
    }

    req.auth = {
      userId:   payload.sub,
      appId:    payload.app_id,
      tenantId: payload.tenant_id ?? null,
      roles:    payload.roles ?? [],
    };
    next();
  } catch (err: any) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'TOKEN_EXPIRED', message: 'Access token has expired.' });
    }
    return res.status(401).json({ error: 'TOKEN_INVALID', message: 'Invalid access token.' });
  }
}
```

**TypeScript declaration** — add to `types.d.ts` in your project:
```typescript
declare global {
  namespace Express {
    interface Request {
      auth?: {
        userId:   string;
        appId:    string;
        tenantId: string | null;
        roles:    string[];
      };
    }
  }
}
```

**Usage:**
```typescript
import { requireAuth } from './authMiddleware';

app.get('/api/profile', requireAuth, (req, res) => {
  res.json({ userId: req.auth!.userId });
});
```

---

## Credentials Summary

| Variable | Required | Description |
|----------|----------|-------------|
| `AUTH_ENGINE_URL` | ✅ | `https://auth-engine-efb8.onrender.com` |
| `AUTH_APP_ID` | ✅ | UUID — sent in every API call body |
| `AUTH_APP_SECRET` | ✅ | 64-char secret — sent in every API call body |
| `AUTH_APP_SLUG` | ✅ | Your app slug — used to verify JWT audience claim |
| `JWT_PUBLIC_KEY` | ✅ | RSA public key — used to verify JWT signatures locally |

---

## What's Next

- **[02 — OTP Authentication](./02-otp-authentication.md)** — Full login flow with complete code examples
- **[03 — Token Management](./03-token-management.md)** — JWT verification, token refresh, logout
