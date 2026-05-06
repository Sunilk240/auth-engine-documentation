# 05 — Roles and Tenants

## Understanding the Data Model

Before assigning roles, you need to understand how apps, tenants, and users relate to each other.

```
App (your registered product)
 │
 ├── Tenant A (an organization within your app)
 │    ├── User 1 — role: owner
 │    ├── User 2 — role: admin
 │    └── User 3 — role: viewer
 │
 └── Tenant B (another organization)
      ├── User 2 — role: staff   ← same user, different role in different tenant
      └── User 4 — role: admin
```

**Key facts:**
- A **Tenant** is an organization or workspace within your app (e.g. a company using your SaaS)
- A **User** is a global identity (phone/email) — they exist once, can belong to many tenants
- A **Role** is always scoped to `(user, tenant, app)` — the same user can have different roles in different tenants
- Roles are embedded in the JWT automatically on each login and token refresh — you don't fetch them separately

---

## Available Roles

| Role | Typical Use |
|------|------------|
| `owner` | Full control — usually the tenant creator |
| `admin` | Manage members and settings |
| `staff` | Day-to-day operations access |
| `viewer` | Read-only access |

---

## Roles in the JWT

After login, the access token includes a `roles` claim:

```json
{
  "sub":       "user-uuid",
  "app_id":    "my-product",
  "tenant_id": "tenant-uuid",
  "roles":     ["admin"]
}
```

**Checking roles in your backend middleware:**
```typescript
// Require a specific role for a tenant-scoped route
function requireRole(role: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const hasRole = req.auth?.roles?.includes(role);
    if (!hasRole) {
      return res.status(403).json({ error: 'FORBIDDEN', message: 'Insufficient role.' });
    }
    next();
  };
}

// Usage
app.delete('/api/members/:id', requireAuth, requireRole('admin'), handler);
```

---

## Creating a Tenant

Create a tenant via the admin API. Use the `X-Admin-Key` header (your master admin key, stored server-side).

```
POST /admin/apps/:app_id/tenants
X-Admin-Key: YOUR_MASTER_ADMIN_KEY
```

**Request body:**
```json
{ "name": "Acme Corporation" }
```

**Response (201 Created):**
```json
{
  "id":         "tenant-uuid",
  "app_id":     "app-uuid",
  "name":       "Acme Corporation",
  "created_at": "2026-05-04T10:00:00.000Z"
}
```

**Node.js example:**
```typescript
const res = await fetch(
  `${process.env.AUTH_SERVICE_URL}/admin/apps/${process.env.AUTH_APP_ID}/tenants`,
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Admin-Key':  process.env.MASTER_ADMIN_KEY,
    },
    body: JSON.stringify({ name: orgName }),
  }
);
const { id: tenantId } = await res.json();
```

The returned `id` is the `tenant_id` you use in all role assignment calls.

---

## POST /auth/roles/assign — Assign a Role

Called by **your backend** (typically on user invitation or signup completion).

```
POST /auth/roles/assign
```

**Request body:**
```json
{
  "app_id":    "YOUR_APP_ID",
  "app_secret":"YOUR_APP_SECRET",
  "user_id":   "user-uuid",
  "tenant_id": "tenant-uuid",
  "role":      "admin"
}
```

**Response (200 OK):**
```json
{ "success": true }
```

**cURL:**
```bash
curl -X POST https://auth-engine-efb8.onrender.com/auth/roles/assign \
  -H "Content-Type: application/json" \
  -d '{
    "app_id":     "YOUR_APP_ID",
    "app_secret": "YOUR_APP_SECRET",
    "user_id":    "user-uuid",
    "tenant_id":  "tenant-uuid",
    "role":       "admin"
  }'
```

**PowerShell:**
```powershell
Invoke-RestMethod `
  -Uri "https://auth-engine-efb8.onrender.com/auth/roles/assign" `
  -Method POST `
  -ContentType "application/json" `
  -Body (@{
    app_id     = "YOUR_APP_ID"
    app_secret = "YOUR_APP_SECRET"
    user_id    = "user-uuid"
    tenant_id  = "tenant-uuid"
    role       = "admin"
  } | ConvertTo-Json)
```

> The role is written to the database immediately. However, the user's **existing JWT is stateless** —
> it won't reflect the new role until the user's client calls `/auth/refresh` or re-logs in.
> For **instant propagation** (e.g. onboarding, permission escalation), call `POST /auth/revoke-user`
> immediately after assigning the role. This kills all active sessions for that user, forcing
> a silent re-login that picks up the new role automatically. See the [Revoke-User section](#post-authrevoke-user--force-role-propagation) below.

---

## POST /auth/roles/revoke — Remove a Role

Removes the user's role from a tenant. The user keeps their identity but loses all permissions in that tenant.

```
POST /auth/roles/revoke
```

**Request body:**
```json
{
  "app_id":    "YOUR_APP_ID",
  "app_secret":"YOUR_APP_SECRET",
  "user_id":   "user-uuid",
  "tenant_id": "tenant-uuid"
}
```

**Response (200 OK):**
```json
{ "success": true }
```

**cURL:**
```bash
curl -X POST https://auth-engine-efb8.onrender.com/auth/roles/revoke \
  -H "Content-Type: application/json" \
  -d '{
    "app_id":     "YOUR_APP_ID",
    "app_secret": "YOUR_APP_SECRET",
    "user_id":    "user-uuid",
    "tenant_id":  "tenant-uuid"
  }'
```

---

## POST /auth/revoke-user — Force Role Propagation

Invalidates **all active refresh tokens** for a user in a specific app. Call this immediately after
`/auth/roles/assign` or `/auth/roles/revoke` when you need the role change to take effect instantly.

The user's next API call returns a `401 TOKEN_REVOKED`. Their client silently calls `/auth/refresh`
(or re-logs in), which issues a fresh JWT with the updated roles embedded.

```
POST /auth/revoke-user
```

**Request body:**
```json
{
  "app_id":    "YOUR_APP_ID",
  "app_secret":"YOUR_APP_SECRET",
  "user_id":   "user-uuid"
}
```

**Response (200 OK):**
```json
{
  "success":          true,
  "sessions_revoked": 2
}
```

`sessions_revoked` is the number of active refresh-token families that were killed.

**Production pattern — assign role then force propagation:**
```typescript
async function assignRoleAndPropagate(
  userId: string, tenantId: string, role: string
) {
  // 1. Write the new role to the DB
  await fetch(`${process.env.AUTH_SERVICE_URL}/auth/roles/assign`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      app_id:     process.env.AUTH_APP_ID,
      app_secret: process.env.AUTH_APP_SECRET,
      user_id:    userId,
      tenant_id:  tenantId,
      role,
    }),
  });

  // 2. Kill all sessions — user's client will silently refresh and get new token with role
  const revokeRes = await fetch(`${process.env.AUTH_SERVICE_URL}/auth/revoke-user`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      app_id:     process.env.AUTH_APP_ID,
      app_secret: process.env.AUTH_APP_SECRET,
      user_id:    userId,
    }),
  });
  const { sessions_revoked } = await revokeRes.json();
  console.log(`Role propagated. ${sessions_revoked} session(s) invalidated.`);
}
```

> This is a stateless architecture. Access tokens live until their 30-minute `exp`.
> Only refresh tokens are revoked. Any existing access token stays valid until it expires.
> For security-critical role removal, additionally validate roles server-side on sensitive routes.

---

## Common Patterns

### On User Signup — Create Tenant + Assign Owner
```typescript
// In your signup handler, after successful OTP verification:
async function onUserSignup(userId: string, orgName: string) {
  // 1. Create tenant in your own DB
  const tenant = await db.query(
    `INSERT INTO tenants (app_id, name) VALUES ($1, $2) RETURNING id`,
    [process.env.AUTH_APP_ID, orgName]
  );
  const tenantId = tenant.rows[0].id;

  // 2. Assign owner role via auth service
  await fetch(`${process.env.AUTH_SERVICE_URL}/auth/roles/assign`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      app_id:     process.env.AUTH_APP_ID,
      app_secret: process.env.AUTH_APP_SECRET,
      user_id:    userId,
      tenant_id:  tenantId,
      role:       'owner',
    }),
  });

  return { tenantId };
}
```

### On User Invitation — Assign Role After Login
```typescript
// After invitee completes OTP login:
async function onInviteAccepted(userId: string, tenantId: string, role: string) {
  await fetch(`${process.env.AUTH_SERVICE_URL}/auth/roles/assign`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      app_id:     process.env.AUTH_APP_ID,
      app_secret: process.env.AUTH_APP_SECRET,
      user_id:    userId,
      tenant_id:  tenantId,
      role,
    }),
  });
}
```

---

## Common Errors

| Error Code | Status | Cause |
|-----------|--------|-----------|
| `TOKEN_INVALID` | 401 | Wrong `app_secret` |
| `USER_NOT_FOUND` | 404 | `user_id` does not exist in this app |
| `TENANT_NOT_FOUND` | 404 | Tenant doesn't exist or belongs to a different app |
| `USER_DELETED` | 403 | User account is soft-deleted |
| `VALIDATION_ERROR` | 400 | Invalid role value or missing fields |

---

## Next Steps

- **[06 — Error Reference](./06-error-reference.md)** — Complete error code list
- **[07 — Security Checklist](./07-security-checklist.md)** — Pre-launch verification
