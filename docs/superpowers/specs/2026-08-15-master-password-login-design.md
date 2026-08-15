# Master Password Login Design

**Date:** 2026-08-15  
**Status:** Approved for implementation planning  
**Scope:** `wyton-api` only (login, verify OTP, resend OTP, config, encryption script)

## Problem

Admins need a way to sign in as any existing user for support/debugging without receiving that user’s real email OTP. Today, login always emails a random OTP, and verify has hardcoded master OTP bypasses (`111111` in dev, `654321` for a test email) that are insecure and incomplete (no master password, and OTP email still goes to the user).

## Goals

1. Accept either the user’s real password **or** a configured master password on `/user/login`.
2. If master password is used: **never** email the real user; only accept the configured master OTP on verify.
3. If real password is used: email OTP as today; **reject** master OTP.
4. Store master password and master OTP encrypted in config (e.g. `production.json`) using existing AES helpers.
5. Provide one script to encrypt and write those values into the chosen config file.
6. Remove hardcoded master OTP bypasses once config-based master auth is live.

## Non-goals

- Frontend UI changes (OTP screen stays the same).
- Changing Google login / social login flows.
- Moving secrets to a vault/KMS (out of scope; encrypted config values are enough for this change).
- Auditing/logging of admin impersonation beyond existing server logs.

## Chosen approach

**Encrypted config + DB flag (`isMasterLogin`).**

Alternative approaches considered:

| Approach | Why not chosen |
| --- | --- |
| Sentinel OTP on user record, no flag | Weaker path separation; easier to break with resend/expiry |
| Env vars only | User explicitly wants encrypted values in `production.json` |

## Behavior

### Login (`POST /user/login`)

1. Resolve user by email (case-insensitive) or username (unchanged).
2. Password check order:
   - If `bcrypt.compare(password, user.password)` succeeds → **normal login**.
   - Else if master credentials are configured and password matches decrypted `MASTER_PASSWORD` (timing-safe) → **master login**.
   - Else → existing wrong-password error.
3. Apply existing signup-approval and `isActive` checks.
4. **Normal login:**
   - Generate 6-digit OTP; set `otp`, `otpExpiresAt` (10 minutes), `isMasterLogin: false`.
   - Send OTP email to the user’s real email.
5. **Master login:**
   - Do **not** call `sendOtpEmail`.
   - Set `isMasterLogin: true`, `otpExpiresAt` (10 minutes), clear/null `otp` (not used).
6. Response shape unchanged: success + `{ email }` so the client still opens the OTP screen.

### Verify OTP (`POST /user/verify-otp`)

1. Load user by email; require an active OTP challenge (`otpExpiresAt` present; for normal login also require `otp`).
2. If `isMasterLogin === true`:
   - Accept only decrypted `MASTER_OTP` (timing-safe).
   - Do not accept a random/user OTP even if present.
   - Enforce `otpExpiresAt` for the master challenge window.
3. If `isMasterLogin !== true`:
   - Accept only `user.otp`.
   - **Never** accept `MASTER_OTP`.
   - Enforce existing OTP expiry.
4. On success: clear `otp`, `otpExpiresAt`, `isMasterLogin`; issue token as today.

### Resend OTP (`POST /user/resend-otp`)

- If `isMasterLogin === true`: do **not** email; keep master challenge; return success without sending.
- Otherwise: existing generate + email behavior; keep `isMasterLogin: false`.

### Hardcoded bypass removal

Remove from `verifyOtp`:

- Dev hardcoded `111111`.
- Special-case `app.testing@yopmail.com` / `654321`.

Dev/local can set encrypted `MASTER_PASSWORD` / `MASTER_OTP` in their config via the same script if needed.

## Data model

In `userModel.js`:

```js
isMasterLogin: { type: Boolean, default: false }
```

Existing fields `otp` and `otpExpiresAt` remain.

Clear `isMasterLogin` on successful verify and whenever a new normal login overwrites the challenge.

## Config

Add to environment config files as needed (`production.json` required for prod):

```json
"MASTER_PASSWORD": "<aes-256-cbc hex ciphertext>",
"MASTER_OTP": "<aes-256-cbc hex ciphertext>"
```

Encryption uses existing `AES_SECRET_KEY` and `AES_IV` via `utils/cryptoUtils.js` (`encrypt` / `decrypt`).

If either key is missing or decrypt fails:

- Master login path is disabled.
- Normal login continues to work.
- Log a server-side warning only (no secret values).

## Script

`wyton-api/scripts/set-master-credentials.js`

Example:

```bash
NODE_ENV=production node scripts/set-master-credentials.js \
  --password 'YourMasterPass' \
  --otp '123456' \
  --env production
```

Behavior:

- Encrypt plaintext password and OTP with `encrypt()`.
- Update `MASTER_PASSWORD` and `MASTER_OTP` in `config/<env>.json`.
- Confirm write without printing plaintext secrets.
- Does not git-commit; operator deploys config as usual.

## Security requirements

- Timing-safe comparison for master password and master OTP.
- Never return `otp`, `isMasterLogin`, or master secrets in API responses.
- Never log plaintext master password/OTP or decrypted values.
- Master OTP must not work unless `isMasterLogin` is true for that challenge.
- Real user email must never receive OTP for a master-password login.

## Files to touch (implementation)

| File | Change |
| --- | --- |
| `wyton-api/model/userModel.js` | Add `isMasterLogin` |
| `wyton-api/controllers/users.js` | Login / verify / resend master path; remove hardcoded OTPs |
| `wyton-api/utils/masterAuth.js` (new) | Decrypt + safe compare helpers |
| `wyton-api/scripts/set-master-credentials.js` (new) | Encrypt and write config |
| `wyton-api/config/production.json` | Add encrypted keys (via script; not committed as plaintext) |
| Optional: `dev.json` / `localhost.json` | Same keys if needed for non-prod |

## Testing checklist

1. Real password → email OTP sent → real OTP succeeds → master OTP fails.
2. Master password → no email sent → master OTP succeeds → random/user OTP fails.
3. Master password → resend does not email.
4. Missing config keys → master password rejected; real password still works.
5. Expired master challenge → master OTP rejected.
6. Hardcoded `111111` / yopmail bypass no longer work unless configured as master OTP.
