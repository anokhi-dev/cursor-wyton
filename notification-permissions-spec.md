# Feature Spec — Notification Permissions (Create / Update / Delete)

Companion to [notifications-audit.md](notifications-audit.md). Supersedes the earlier [notification-preferences-spec.md](notification-preferences-spec.md) — the earlier version proposed separate collections; this one folds notification control into the existing role/user `permissions` map.

Status: Draft for review · Owner: (assign) · Target release: (tbd)

---

## 1. Goal

Let an admin decide which roles receive **Create / Update / Delete** notifications for a targeted set of modules, using the existing role-and-permission mechanism. No new collections. User-specific overrides continue to work via the existing `users.permissions` map.

Modules in scope:

| # | Module key | Source in audit |
|---|---|---|
| 1 | `CONTRACT_MANAGEMENT` | §7 |
| 2 | `PHASE_INVOICE` | §8 |
| 3 | `CHANGE_EVENT` | §9 |
| 4 | `PERMIT_TRACKING` | §10 |
| 5 | `VIOLATION_TRACKING` | §10 |
| 6 | `RFI_TRACKING` | §10 |
| 7 | `SCHEDULING` | §11 |

---

## 2. Design (locked)

Presence of a permission string in the user's effective `permissions` map = the user receives that notification. Absence = skip.

Three new action strings per module, following the existing `{MODULE}_ALL` / `{MODULE}_VIEW` convention:

```
{MODULE}_NOTIFICATION_CREATE
{MODULE}_NOTIFICATION_UPDATE
{MODULE}_NOTIFICATION_DELETE
```

Example role document ([model/role.js](wyton-api/model/role.js)):

```json
{
  "name": "admin",
  "permissions": {
    "CONTRACT_MANAGEMENT": [
      "CONTRACT_MANAGEMENT_ALL",
      "CONTRACT_MANAGEMENT_VIEW",
      "CONTRACT_MANAGEMENT_NOTIFICATION_CREATE",
      "CONTRACT_MANAGEMENT_NOTIFICATION_UPDATE",
      "CONTRACT_MANAGEMENT_NOTIFICATION_DELETE"
    ],
    "PHASE_INVOICE":      ["PHASE_INVOICE_NOTIFICATION_CREATE", ...],
    "CHANGE_EVENT":       ["CHANGE_EVENT_NOTIFICATION_CREATE", ...],
    "PERMIT_TRACKING":    ["PERMIT_TRACKING_NOTIFICATION_UPDATE"],
    "VIOLATION_TRACKING": ["VIOLATION_TRACKING_NOTIFICATION_UPDATE"],
    "RFI_TRACKING":       ["RFI_TRACKING_NOTIFICATION_UPDATE"],
    "SCHEDULING":         ["SCHEDULING_NOTIFICATION_CREATE", "SCHEDULING_NOTIFICATION_UPDATE"]
  }
}
```

Decisions in effect:

- **Bundled channels** — one string per action gates all three channels (email + push + in-app) for that action. No per-channel strings.
- **Split correspondence** — Permit / Violation / RFI are three separate module keys.
- **Naming** — `DELETE` (not `REMOVE`).
- **Backfill on release** — all 21 strings seeded on `admin` and `employee` roles; existing users' `permissions` maps updated in the same migration so no notification goes silent.
- **Status changes → UPDATE** — Contract Management phase transitions, insurance state changes, change-event approve/reject, etc. all bucket under `_NOTIFICATION_UPDATE`.

Precedence (unchanged from today's model):

1. User's `users.permissions[MODULE]` if present → authoritative.
2. Otherwise, role's `roles.permissions[MODULE]` for the user's role.
3. `ADMIN_ALL` still bypasses everything (super-power).

---

## 3. Full permission catalogue

21 new action strings. Add to [lib/constant.js](wyton-api/lib/constant.js) as an exported enum so validators and the frontend can share it.

```js
// lib/constant.js — add near existing `notification_module` at line 586
const notification_permission_modules = [
  "CONTRACT_MANAGEMENT",
  "PHASE_INVOICE",
  "CHANGE_EVENT",
  "PERMIT_TRACKING",
  "VIOLATION_TRACKING",
  "RFI_TRACKING",
  "SCHEDULING",
];

const notification_permission_actions = ["CREATE", "UPDATE", "DELETE"];

// Flattened list of all 21 strings — convenient for validators & seeds.
const notification_permission_strings =
  notification_permission_modules.flatMap(m =>
    notification_permission_actions.map(a => `${m}_NOTIFICATION_${a}`)
  );
```

Full list (for reference):

```
CONTRACT_MANAGEMENT_NOTIFICATION_CREATE / UPDATE / DELETE
PHASE_INVOICE_NOTIFICATION_CREATE       / UPDATE / DELETE
CHANGE_EVENT_NOTIFICATION_CREATE        / UPDATE / DELETE
PERMIT_TRACKING_NOTIFICATION_CREATE     / UPDATE / DELETE
VIOLATION_TRACKING_NOTIFICATION_CREATE  / UPDATE / DELETE
RFI_TRACKING_NOTIFICATION_CREATE        / UPDATE / DELETE
SCHEDULING_NOTIFICATION_CREATE          / UPDATE / DELETE
```

---

## 4. Trigger → action bucket mapping

Every audit-numbered trigger for the 7 modules routed to CREATE / UPDATE / DELETE. Status changes go to UPDATE (decision locked in §2).

### 4.1 Contract Management → `CONTRACT_MANAGEMENT`

| Audit # | Trigger | Action | Recipient (as-is) |
|---|---|---|---|
| 35 | send_email_to_user — vendor already registered | CREATE | Existing vendor user |
| 36 | send_email_to_user — invite pending | CREATE | Same target |
| 37 | send_email_to_user — brand-new user | CREATE | New invitee |
| 38 | Contract sent for signature | CREATE | Vendor + all admin/employee |
| 39 | Vendor submits insurance module | CREATE | All admin/employee |
| 40 | Vendor requests insurance waiver | CREATE | All admin/employee |
| 41 | Vendor marks contract signed | UPDATE | All admin/employee |
| 42 | Phase status → REQUESTED | UPDATE | Admin + employee (except actor) |
| 43 | Phase status → APPROVED | UPDATE | Vendor |
| 44 | Phase status → EDITED_BY_VENDOR | UPDATE | Admin + employee |
| 45 | Phase status → VENDOR_APPROVED | UPDATE | Admin + employee |
| 46 | Cron — insurance expiring in 15 days | UPDATE | Trade RP(s) + vendor |
| — | Future: contract / phase deleted | DELETE | (tbd) |

### 4.2 Phase Invoices → `PHASE_INVOICE`

| Audit # | Trigger | Action | Recipient |
|---|---|---|---|
| 47 | Vendor raises phase invoice | CREATE | All BIC + RP on trade (except creator) |
| — | Future: invoice edited | UPDATE | tbd |
| — | Future: invoice voided | DELETE | tbd |

### 4.3 Change Events → `CHANGE_EVENT`

| Audit # | Trigger | Action | Recipient |
|---|---|---|---|
| 48 | Vendor creates change event | CREATE | All admin/employee |
| 49 | Admin/employee creates change event | CREATE | All vendors on project |
| 50 | Vendor re-submits rejected change event | UPDATE | All admin/employee |
| 51 | Change event → APPROVED | UPDATE | Creator |
| 52 | Change event → REJECTED | UPDATE | Creator |
| — | Future: change event deleted | DELETE | tbd |

### 4.4 Correspondence → `PERMIT_TRACKING` / `VIOLATION_TRACKING` / `RFI_TRACKING`

Same trigger shape for all three. The dispatcher [emitTrackedCorrespondenceItemUpdateNotification (services/correspondenceTrackingRowActionServices.js:109)](wyton-api/services/correspondenceTrackingRowActionServices.js) already accepts an item-type discriminator — pick the module key from that.

| Audit # | Trigger | Action | Recipient |
|---|---|---|---|
| — | Item created | CREATE | (add if desired — currently silent) |
| 53–55 | Priority toggle / nudge / follow-up / field update | UPDATE | Watchers on `notificationWatchers` (primary skipped) |
| — | Item deleted | DELETE | (add if desired — currently silent) |

### 4.5 Scheduling → `SCHEDULING`

| Audit # | Trigger | Action | Recipient |
|---|---|---|---|
| 56 | Schedule activity created — new assignee | CREATE | Assignee |
| 57 | Schedule activity created — new approver | CREATE | Approver |
| 58 | Schedule updated — newly added assignees/approvers | UPDATE | New assignees/approvers |
| — | Future: schedule item deleted | DELETE | tbd |

---

## 5. Backend integration

### 5.1 New helper — `lib/notificationPermissions.js`

```js
// lib/notificationPermissions.js
const User = require("../model/userModel");
const Role = require("../model/role");

// Returns the user's effective permission array for a given module.
// User map wins over role map, matching the pattern in
// controllers/users.js:19-47 (actorHasAdminAll).
async function effectivePermissions(user, moduleKey) {
  const userMap = user.permissions?.get?.(moduleKey) ?? user.permissions?.[moduleKey];
  if (userMap && userMap.length) return userMap;

  const role = user.roleId?.permissions
    ? user.roleId
    : await Role.findById(user.roleId).lean();
  return role?.permissions?.[moduleKey] ?? [];
}

async function canReceiveNotification(user, moduleKey, action) {
  // ADMIN_ALL super-bypass — keep parity with existing helpers.
  const adminPerms = await effectivePermissions(user, "ADMIN");
  if (adminPerms.includes("ADMIN_ALL")) return true;

  const perms = await effectivePermissions(user, moduleKey);
  return perms.includes(`${moduleKey}_NOTIFICATION_${action}`);
}

async function filterRecipients(users, moduleKey, action) {
  // Batch-load roles once instead of per-user round-trip.
  const roleIds = [...new Set(users.map(u => String(u.roleId?._id ?? u.roleId)))];
  const roleMap = new Map(
    (await Role.find({ _id: { $in: roleIds } }).lean())
      .map(r => [String(r._id), r])
  );
  return users.filter(u => {
    const role = roleMap.get(String(u.roleId?._id ?? u.roleId));
    return checkSync({ ...u, roleId: role }, moduleKey, action);
  });
}

function checkSync(user, moduleKey, action) {
  const userPerm = user.permissions?.get?.(moduleKey)
                 ?? user.permissions?.[moduleKey];
  const perms = (userPerm && userPerm.length)
    ? userPerm
    : (user.roleId?.permissions?.[moduleKey] ?? []);
  const adminPerms = user.permissions?.get?.("ADMIN")
                  ?? user.permissions?.["ADMIN"]
                  ?? user.roleId?.permissions?.["ADMIN"] ?? [];
  if (adminPerms.includes("ADMIN_ALL")) return true;
  return perms.includes(`${moduleKey}_NOTIFICATION_${action}`);
}

module.exports = { canReceiveNotification, filterRecipients, checkSync };
```

### 5.2 Trigger sites to wrap

Wrap every fan-out point identified in the audit. Pattern:

```js
const eligible = await filterRecipients(recipients, MODULE_KEY, ACTION);
if (!eligible.length) return;
await create_in_app_notification({ userId: eligible.map(u => u._id), module, ... });
await fcmPushNotification.sendNotificationToMultipleUsers(eligible.map(u => u._id), notification);
for (const u of eligible) await sendNotificationEmail(u.email, subject, title, message, opts);
```

Concrete edit list:

| Module + Action | Files · lines |
|---|---|
| `CONTRACT_MANAGEMENT_NOTIFICATION_CREATE` | [services/contractManagementService.js](wyton-api/services/contractManagementService.js) lines 524, 541, 574, 1869, 2576, 2683 |
| `CONTRACT_MANAGEMENT_NOTIFICATION_UPDATE` | [services/contractManagementService.js](wyton-api/services/contractManagementService.js) lines 3274, 4082, 4106, 4148, 4191, 4744, 4754 |
| `PHASE_INVOICE_NOTIFICATION_CREATE` | [controllers/phaseInvoice.js:151, 169](wyton-api/controllers/phaseInvoice.js) (`raiseInvoice`) |
| `CHANGE_EVENT_NOTIFICATION_CREATE` | [services/changeEventServices.js](wyton-api/services/changeEventServices.js) lines 402, 418 |
| `CHANGE_EVENT_NOTIFICATION_UPDATE` | [services/changeEventServices.js](wyton-api/services/changeEventServices.js) lines 602, 677 (approve + reject) |
| `PERMIT_TRACKING_NOTIFICATION_UPDATE` / `VIOLATION_..._UPDATE` / `RFI_..._UPDATE` | [services/correspondenceTrackingRowActionServices.js:109, 161, 219](wyton-api/services/correspondenceTrackingRowActionServices.js) — helper picks the module key from the item-type argument it already receives |
| `SCHEDULING_NOTIFICATION_CREATE` | [services/schedulingServices.js:246, 270](wyton-api/services/schedulingServices.js) (`notifySchedulingAssignments`, called from `create_task` line 711) |
| `SCHEDULING_NOTIFICATION_UPDATE` | [services/schedulingServices.js:816](wyton-api/services/schedulingServices.js) (`update_task`) |

Cron jobs — [jobs/recurringTaskCron.js](wyton-api/jobs/recurringTaskCron.js) reaches the same dispatchers, so wrapping the dispatcher covers cron automatically. Verify [run_insurance_policy_exp_reminder_cron (services/contractManagementService.js:4744)](wyton-api/services/contractManagementService.js) picks up the wrap.

### 5.3 Payload validation

Extend the Joi schema on `POST /user/create-role`, `PUT /user/update-role`, `PUT /user/update-permissions`, `POST /user/create-user`, `PUT /user/invite-user` to allow the new strings. Recommended: constrain each module's array to values in `notification_permission_strings ∪ ["{MODULE}_ALL", "{MODULE}_VIEW"]` — catches typos early.

---

## 6. Admin UI

Extend the **existing** Roles & Permissions editor — no new page.

For each of the 7 modules in the permissions matrix, render three additional checkboxes next to `All` / `View`:

```
CONTRACT MANAGEMENT
  [x] All       [x] View
  [x] Notify — Create   [x] Notify — Update   [x] Notify — Delete
```

Redux slice + editor screen live in the existing admin surface (find the current permissions matrix in `src/pages/admin/` — same page used to seed `ADMIN`/`PROJECT`/etc.). The self-service view on [UserProfile.tsx](wyton-web/src/pages/UserProfile.tsx) can show a read-only "Notifications you receive" section grouped by module, with a mute toggle that writes to the user's `permissions` map (removes the string) via `PUT /user/update-permissions`.

Frontend constant registry — add to [src/constants/notificationModules.ts](wyton-web/src/constants/notificationModules.ts):

```ts
export const NOTIFICATION_PERMISSION_MODULES = [
  "CONTRACT_MANAGEMENT",
  "PHASE_INVOICE",
  "CHANGE_EVENT",
  "PERMIT_TRACKING",
  "VIOLATION_TRACKING",
  "RFI_TRACKING",
  "SCHEDULING",
] as const;

export const NOTIFICATION_PERMISSION_ACTIONS = ["CREATE", "UPDATE", "DELETE"] as const;

export const notifyPermString = (mod: string, act: string) =>
  `${mod}_NOTIFICATION_${act}`;
```

---

## 7. Migration

### 7.1 Seed roles

Update [lib/db.js](wyton-api/lib/db.js) admin seed (lines 38–48) and employee seed (lines 77–87) to include the 21 new strings across the 7 module maps. New roles created via `POST /user/create-role` also default to including all 21 (mirroring today's behaviour where new users copy the role's map).

### 7.2 Backfill existing data

One-off script at `scripts/backfill-notification-permissions.js`:

```js
// Pseudo — actual script does two collectionUpdateMany calls.
for (const roleName of ["admin", "employee"]) {
  Role.updateOne({ name: roleName }, {
    $set: buildNotifKeysForRole(roleName)   // merges strings into permissions map
  });
}
User.find({}).cursor().eachAsync(async (u) => {
  // Only backfill users whose current permissions mirror their role
  // (i.e., no custom override). Skip users whose map diverges from role.
  if (userMirrorsRole(u)) {
    User.updateOne({ _id: u._id }, {
      $set: buildNotifKeysForRole(u.roleId.name)
    });
  }
});
```

Run once during deploy, gated by an env flag so it's idempotent-safe.

### 7.3 Result

Zero behaviour change on release. Every notification the audit lists continues to fire. Admin can then start removing strings to mute.

---

## 8. Deliverables checklist

Backend
- [ ] Add `notification_permission_modules`, `notification_permission_actions`, `notification_permission_strings` to [lib/constant.js](wyton-api/lib/constant.js)
- [ ] `lib/notificationPermissions.js` with `filterRecipients` / `canReceiveNotification` / `checkSync`
- [ ] Wrap trigger sites listed in §5.2
- [ ] Extend Joi validation on the 5 permission-mutation routes in [routes/userRoute.js](wyton-api/routes/userRoute.js)
- [ ] Seed both roles in [lib/db.js](wyton-api/lib/db.js) with the 21 new strings
- [ ] `scripts/backfill-notification-permissions.js` + run in deploy

Web
- [ ] Add 3 columns per module in the admin Roles & Permissions editor
- [ ] Constant registry in [src/constants/notificationModules.ts](wyton-web/src/constants/notificationModules.ts)
- [ ] Optional: user self-service mute panel in [UserProfile.tsx](wyton-web/src/pages/UserProfile.tsx)

QA
- [ ] Toggle each of the 21 strings and verify the corresponding trigger from §4 stops firing for a user in that role.
- [ ] User override: remove the string from a single user → they alone stop receiving; others in the role still get it.
- [ ] `ADMIN_ALL` bypass still receives everything even when module strings are missing.
- [ ] Cron paths (task due, insurance expiry, combined due) respect the gates.
- [ ] Regression: no existing audit-listed notification is silenced by the release (backfill validation).
