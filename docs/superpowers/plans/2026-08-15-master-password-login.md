# Master Password Login Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow admin master-password login without emailing the real user, gated so master OTP only works after master password was used.

**Architecture:** Encrypted `MASTER_PASSWORD` / `MASTER_OTP` in config (AES via existing `cryptoUtils`). Login sets `isMasterLogin` on the user; verify/resend branch on that flag. Script encrypts and writes config values.

**Tech Stack:** Node.js ESM, `config`, `crypto` (timingSafeEqual), Mongo/Mongoose, bcrypt (user passwords)

## Global Constraints

- Never email OTP when master password was used
- Master OTP only when `isMasterLogin === true`
- Timing-safe compare for master secrets; never log plaintext
- Remove hardcoded `111111` and yopmail `654321` bypasses
- Response shapes for login/verify/resend stay the same for the frontend

---

### Task 1: `masterAuth` helpers

**Files:**
- Create: `wyton-api/utils/masterAuth.js`
- Test: `wyton-api/scripts/smoke-master-auth.mjs` (one-off smoke using node)

**Interfaces:**
- Produces: `isMasterPasswordConfigured()`, `matchesMasterPassword(plain)`, `matchesMasterOtp(plain)`, `safeEqualString(a, b)`

- [ ] **Step 1: Implement `wyton-api/utils/masterAuth.js`**

```js
import crypto from "crypto";
import config from "config";
import { decrypt } from "./cryptoUtils.js";

function safeEqualString(a, b) {
  if (typeof a !== "string" || typeof b !== "string") return false;
  const bufA = Buffer.from(a);
  const bufB = Buffer.from(b);
  if (bufA.length !== bufB.length) return false;
  return crypto.timingSafeEqual(bufA, bufB);
}

function getDecrypted(key) {
  try {
    if (!config.has(key)) return null;
    const encrypted = config.get(key);
    if (!encrypted || typeof encrypted !== "string") return null;
    return decrypt(encrypted);
  } catch (err) {
    console.warn(`[masterAuth] Failed to decrypt ${key}`);
    return null;
  }
}

function isMasterPasswordConfigured() {
  return Boolean(getDecrypted("MASTER_PASSWORD") && getDecrypted("MASTER_OTP"));
}

function matchesMasterPassword(plain) {
  const master = getDecrypted("MASTER_PASSWORD");
  if (!master) return false;
  return safeEqualString(plain, master);
}

function matchesMasterOtp(plain) {
  const master = getDecrypted("MASTER_OTP");
  if (!master) return false;
  return safeEqualString(plain, master);
}

export {
  safeEqualString,
  isMasterPasswordConfigured,
  matchesMasterPassword,
  matchesMasterOtp,
};
```

- [ ] **Step 2: Smoke-test encrypt → decrypt → match with localhost AES keys**

Run from `wyton-api` with `NODE_CONFIG_DIR=config` and `NODE_ENV=localhost` (or default) after temporarily setting encrypted values, or call `encrypt` then `matches*` against in-memory mock — prefer verifying `encrypt`/`decrypt` round-trip and `safeEqualString` in a small script.

---

### Task 2: User model field

**Files:**
- Modify: `wyton-api/model/userModel.js` (near `otp` / `otpExpiresAt`)

- [ ] **Step 1: Add field**

```js
otp: { type: String, trim: true },
otpExpiresAt: { type: Date },
isMasterLogin: { type: Boolean, default: false },
```

---

### Task 3: Login / verify / resend

**Files:**
- Modify: `wyton-api/controllers/users.js` (`login`, `verifyOtp`, `resendOtp`)

**Interfaces:**
- Consumes: `matchesMasterPassword`, `matchesMasterOtp` from `../utils/masterAuth.js`

- [ ] **Step 1: Update `login`**
  - After user found: `isMatched = await bcrypt.compare(...)`; if false, `isMatched = matchesMasterPassword(password)` and `usedMasterPassword = true`.
  - Normal: generate OTP, `isMasterLogin: false`, email.
  - Master: `otp: null`, `isMasterLogin: true`, `otpExpiresAt`, **no** `sendOtpEmail`.

- [ ] **Step 2: Update `verifyOtp`**
  - Master path: require `isMasterLogin` + `otpExpiresAt`; accept only `matchesMasterOtp(otp)`; reject otherwise.
  - Normal path: require `otp` + `otpExpiresAt`; accept only `user.otp === otp`; never master OTP.
  - Clear `otp`, `otpExpiresAt`, `isMasterLogin` on success.
  - Delete hardcoded `111111` / yopmail `654321` blocks.

- [ ] **Step 3: Update `resendOtp`**
  - If `user.isMasterLogin`: refresh `otpExpiresAt` optional; return success; no email.
  - Else: existing resend; set `isMasterLogin: false`.

---

### Task 4: Config write script

**Files:**
- Create: `wyton-api/scripts/set-master-credentials.js`

- [ ] **Step 1: Implement CLI**

```bash
NODE_ENV=production node scripts/set-master-credentials.js \
  --password 'YourMasterPass' \
  --otp '123456' \
  --env production
```

- Encrypt with `encrypt()`, write `MASTER_PASSWORD` / `MASTER_OTP` into `config/<env>.json`.
- Do not print plaintext secrets.

- [ ] **Step 2: Run script against a temp or confirm help message works**

---

### Task 5: Manual verification checklist

- [ ] Real password → email path unchanged (code review)
- [ ] Master password → no `sendOtpEmail` call
- [ ] Master OTP only when `isMasterLogin`
- [ ] Hardcoded bypasses gone
