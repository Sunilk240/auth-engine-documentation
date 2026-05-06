# 04 — License Keys

The license key system lets you distribute software to paying customers and enforce activation limits. Each key is tied to your app, optionally to a tenant, and can be locked to a specific number of devices.

---

## Concepts

| Term | Meaning |
|------|---------|
| **License Key** | A unique code (e.g. `MYAPP-XXXX-XXXX-XXXX-XXXX`) given to a customer |
| **Activation** | The act of a device claiming one slot on a license key |
| **max_activations** | How many different devices can activate one key |
| **Device Fingerprint** | A server-computed hash of device_id + user-agent — used to identify unique devices |
| **Verification** | An ongoing check on every app launch that the key is still valid |

---

## Flow Overview

```
Service Owner (Admin)
  │
  │  POST /license/generate   ← needs app_secret (admin action)
  │  Returns license_key
  ▼
Customer receives license_key (email, purchase confirmation, etc.)
  │
  │  Customer's app (via your backend)
  │  POST /license/activate   ← needs app_secret, device_id
  │  Returns activation_id
  ▼
On every app launch:
  POST /license/verify        ← needs app_secret, device_id
  Returns { valid: true/false }
```

---

## Step 1 — Generate a License Key (Admin Action)

This is called by **your backend** when a customer purchases or is provisioned access. Not called by the end user.

```
POST /license/generate
```

**Request body:**
```json
{
  "app_id":          "YOUR_APP_ID",
  "app_secret":      "YOUR_APP_SECRET",
  "label":           "Pro License — Acme Corp",
  "issued_to":       "Acme Corporation",
  "max_activations": 5,
  "expires_at":      "2027-01-01T00:00:00.000Z"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `app_id` | ✅ | Your app UUID |
| `app_secret` | ✅ | Your app secret |
| `label` | Optional | Internal label for your records |
| `issued_to` | Optional | Customer name for your records |
| `max_activations` | Optional | How many devices can activate (default: 1) |
| `tenant_id` | Optional | Lock this key to a specific tenant |
| `expires_at` | Optional | ISO 8601 — key invalid after this date |

**Response (201 Created):**
```json
{
  "license_key":     "MYAPP-A1B2-C3D4-E5F6-G7H8",
  "license_id":      "license-uuid",
  "max_activations": 5,
  "expires_at":      "2027-01-01T00:00:00.000Z"
}
```

**cURL:**
```bash
curl -X POST https://auth-engine-efb8.onrender.com/license/generate \
  -H "Content-Type: application/json" \
  -d '{
    "app_id":          "YOUR_APP_ID",
    "app_secret":      "YOUR_APP_SECRET",
    "label":           "Pro License",
    "issued_to":       "Customer Name",
    "max_activations": 1
  }'
```

**PowerShell:**
```powershell
$licRes = Invoke-RestMethod `
  -Uri "https://auth-engine-efb8.onrender.com/license/generate" `
  -Method POST `
  -ContentType "application/json" `
  -Body (@{
    app_id          = "YOUR_APP_ID"
    app_secret      = "YOUR_APP_SECRET"
    label           = "Pro License"
    issued_to       = "Customer Name"
    max_activations = 1
  } | ConvertTo-Json)

Write-Host "License Key: $($licRes.license_key)"
```

---

## Step 2 — Activate a License Key

Called when the end user installs your software and enters the license key for the first time.

```
POST /license/activate
```

**Request body:**
```json
{
  "license_key":  "MYAPP-A1B2-C3D4-E5F6-G7H8",
  "app_id":       "YOUR_APP_ID",
  "app_secret":   "YOUR_APP_SECRET",
  "device_id":    "unique-device-identifier",
  "device_label": "John's MacBook Pro"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `license_key` | ✅ | The key the customer entered |
| `app_id` | ✅ | Your app UUID |
| `app_secret` | ✅ | Your app secret |
| `device_id` | Optional | A stable unique ID for this device (see note below) |
| `device_label` | Optional | Human-readable device name for admin UI |

> **Device ID:** Generate a stable UUID on first install and persist it in local storage (file, registry, OS keychain). Do not use the MAC address or device serial — those can change. The server computes a fingerprint from `device_id + user-agent` — same device always gets the same fingerprint.

**Response (201 Created — first activation):**
```json
{
  "activation_id":    "activation-uuid",
  "already_activated": false,
  "activations_used": 1,
  "max_activations":  5
}
```

**Response (200 OK — same device activating again, idempotent):**
```json
{
  "activation_id":    "same-activation-uuid",
  "already_activated": true,
  "activations_used": 1,
  "max_activations":  5
}
```

**cURL:**
```bash
curl -X POST https://auth-engine-efb8.onrender.com/license/activate \
  -H "Content-Type: application/json" \
  -H "User-Agent: MyApp/1.0 (Windows)" \
  -d '{
    "license_key":  "MYAPP-A1B2-C3D4-E5F6-G7H8",
    "app_id":       "YOUR_APP_ID",
    "app_secret":   "YOUR_APP_SECRET",
    "device_id":    "stable-device-uuid",
    "device_label": "My PC"
  }'
```

---

## Step 3 — Verify a License Key (Every App Launch)

Call this every time the application starts to confirm the key is still valid and not revoked.

```
POST /license/verify
```

**Request body:**
```json
{
  "license_key": "MYAPP-A1B2-C3D4-E5F6-G7H8",
  "app_id":      "YOUR_APP_ID",
  "app_secret":  "YOUR_APP_SECRET",
  "device_id":   "stable-device-uuid"
}
```

**Response (200 OK — valid):**
```json
{
  "valid":          true,
  "license_id":     "license-uuid",
  "activation_id":  "activation-uuid",
  "expires_at":     "2027-01-01T00:00:00.000Z",
  "last_verified_at": "2026-05-03T10:00:00.000Z"
}
```

**Response (valid field = false cases):**

| Scenario | `valid` | Error Code | Status |
|----------|---------|-----------|--------|
| Key does not exist | — | `LICENSE_NOT_FOUND` | 404 |
| Key revoked by admin | — | `LICENSE_REVOKED` | 403 |
| Key expired (past `expires_at`) | — | `LICENSE_EXPIRED` | 403 |
| Device not activated | — | `LICENSE_NOT_ACTIVATED` | 403 |
| Max activations reached | — | `LICENSE_MAX_ACTIVATIONS` | 403 |

**cURL:**
```bash
curl -X POST https://auth-engine-efb8.onrender.com/license/verify \
  -H "Content-Type: application/json" \
  -H "User-Agent: MyApp/1.0 (Windows)" \
  -d '{
    "license_key": "MYAPP-A1B2-C3D4-E5F6-G7H8",
    "app_id":      "YOUR_APP_ID",
    "app_secret":  "YOUR_APP_SECRET",
    "device_id":   "stable-device-uuid"
  }'
```

**PowerShell:**
```powershell
$verifyRes = Invoke-RestMethod `
  -Uri "https://auth-engine-efb8.onrender.com/license/verify" `
  -Method POST `
  -ContentType "application/json" `
  -Body (@{
    license_key = "MYAPP-A1B2-C3D4-E5F6-G7H8"
    app_id      = "YOUR_APP_ID"
    app_secret  = "YOUR_APP_SECRET"
    device_id   = "stable-device-uuid"
  } | ConvertTo-Json)

if ($verifyRes.valid) {
  Write-Host "License valid — app can run"
} else {
  Write-Host "License invalid — block app access"
}
```

---

## Admin Actions

### Check License Status

```
POST /license/status
```
```json
{
  "app_id":      "YOUR_APP_ID",
  "app_secret":  "YOUR_APP_SECRET",
  "license_key": "MYAPP-A1B2-C3D4-E5F6-G7H8"
}
```

### Revoke a License (Admin)

Permanently blocks the key. Verify calls will return `LICENSE_REVOKED`.

```
POST /license/revoke
```
```json
{
  "app_id":      "YOUR_APP_ID",
  "app_secret":  "YOUR_APP_SECRET",
  "license_key": "MYAPP-A1B2-C3D4-E5F6-G7H8",
  "reason":      "Customer refunded"
}
```

### Deactivate a Single Device

Frees one activation slot without revoking the whole key.

```
POST /license/deactivate
```
```json
{
  "app_id":        "YOUR_APP_ID",
  "app_secret":    "YOUR_APP_SECRET",
  "activation_id": "activation-uuid"
}
```

---

## Common Errors

| Error Code | Status | Cause |
|-----------|--------|-------|
| `LICENSE_NOT_FOUND` | 404 | Key doesn't exist or belongs to a different app |
| `LICENSE_INVALID` | 400 | Key format is malformed |
| `LICENSE_REVOKED` | 403 | Key was revoked by admin |
| `LICENSE_EXPIRED` | 403 | Key is past its `expires_at` date |
| `LICENSE_NOT_ACTIVATED` | 403 | Device has not been activated |
| `LICENSE_MAX_ACTIVATIONS` | 403 | All activation slots are taken |
| `LICENSE_ALREADY_REVOKED` | 409 | Revoke called on already-revoked key |

---

## Next Steps

- **[05 — Roles and Tenants](./05-roles-and-tenants.md)** — Assign roles to authenticated users
- **[06 — Error Reference](./06-error-reference.md)** — Complete error code list
