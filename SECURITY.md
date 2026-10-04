# SECURITY POLICY — Cardiology Clinic Management Ecosystem

> Backend security model overview. This system handles **Protected Health Information (PHI)** for cardiology patients.

---

## 1. 🛡️ Overview

The system is built on Firebase and follows a strict **least-privilege, audit-first, no-PHI-in-logs** policy.

- **Authentication:** Firebase Auth
- **Database:** Firestore (protected by security rules)
- **Storage:** Firebase Cloud Storage (protected by storage rules)
- **Compute:** Cloud Functions Gen 2 (TypeScript)
- **Integration:** Meta WhatsApp Cloud API + Google Gemini AI

---

## 2. 🔑 Authentication

- All access to Firestore and Storage requires `request.auth != null`.
- Callable functions verify `request.auth` and return `HttpsError("unauthenticated")` when absent.
- The staff role is passed as a **Custom Claim** (`request.auth.token.role`) and set server-side only — it cannot be spoofed from the client.

---

## 3. 🚦 Authorization (RBAC)

The Firestore security rules enforce role-based access control:

```js
function isAuthenticated() { return request.auth != null; }

function hasRole(role) {
  return isAuthenticated() && request.auth.token.role == role;
}

function isStaff() {
  return hasRole('Physician') ||
         hasRole('Lead Assistant') ||
         hasRole('Staff Assistant');
}
```

**Staff Roles:**
- `Physician`
- `Lead Assistant`
- `Staff Assistant`

### Collection Permission Table

| Collection | Read | Write | Note |
|-----------|------|-------|------|
| `Patients` | `isStaff()` | `isStaff()` | Core patient records |
| `Bookings` | `isStaff()` | `false` | Writes via Cloud Functions only (admin context) |
| `WaitlistEntries` | `isStaff()` | `isStaff()` | Waitlist |
| `Conversations` | `isStaff()` | `isStaff()` | Unified inbox root |
| `Conversations/.../messages` | `isStaff()` | `isStaff()` | Message threads per WhatsApp number |
| `MedicalRecords` | `isStaff()` | `isStaff()` | Patient medical history |
| `ai_logs` | `isStaff()` | `false` | Audit only — written by functions |
| `EmergencyAlerts` | `isStaff()` | `false` | Written by functions only |
| Any other collection | `false` | `false` | Default block |

---

## 4. 🏥 PHI Protection

- All PHI collections are restricted by `request.auth` + `isStaff()`.
- **Booking status changes** are blocked from the client (`allow write: if false`) — performed exclusively via Cloud Functions in the admin context.
- **Server-only write collections:** `ai_logs`, `EmergencyAlerts`, `Services` — not writable from the client.
- **Cache purge:** The frontend PHI cache (Hive, encrypted) is purged on logout.
- **No PHI in logs:** The webhook avoids printing raw message bodies; the orchestrator logs only response type and length.

---

## 5. 🔐 Secrets Management

- **No committed keys:** All service-account JSON files are excluded in `.gitignore`.
- **Secret Manager / `defineSecret`:** All secrets are defined via `defineSecret` in code and set using:
  ```bash
  firebase functions:secrets:set <SECRET_NAME>
  ```
- **ADC:** Use Application Default Credentials for local development. Do not commit key files to the repository.
- **Rotation:** Any exposed key must be rotated immediately and the old key revoked.

---

## 6. ✅ App Check

- **Recommendation:** Enable `enforceAppCheck: true` on all callables once a reCAPTCHA Enterprise key is provisioned.
- **Web API Key:** Must be restricted to the clinic's hosting domains only.

---

## 7. 🪝 Webhook Security

The WhatsApp webhook:

- **Verification (GET):** Compares `hub.verify_token` against the secret using `crypto.timingSafeEqual` (constant-time comparison, prevents timing attacks).
- **Message signature (POST):** Validates the `x-hub-signature-256` header using HMAC-SHA256. Any invalid signature returns `403`.
- **Idempotency layer:** Prevents duplicate processing of Meta retries.
- **Rate-limit layer:** Blocks abuse early before any AI processing.

---

## 8. 📝 Compliance Notes

Because the system is medical and contains PHI:

- **Least privilege:** Writes to sensitive collections (`Bookings`, `ai_logs`, `EmergencyAlerts`) are restricted to the backend only.
- **Audit trail:** Every AI interaction is logged in `ai_logs` (readable by staff for auditing).
- **No PHI in logs:** Raw message content is never logged.
- **Emergencies:** Emergency keywords create `EmergencyAlerts` documents → dashboard shows a red banner for the medical team.
- **Key rotation:** Periodically review Secret Manager permissions and IAM access.
