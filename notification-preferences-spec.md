# Feature Spec — Role-Based Notification Preferences (Create / Update / Delete)

Companion doc to [notifications-audit.md](notifications-audit.md). Introduces admin-configurable notification switches per **role × module × action** for a targeted set of modules.

Status: Draft for review · Owner: (assign) · Target release: (tbd)

---

## 1. Goal

Today every notification trigger fires unconditionally. Admin cannot turn notifications on/off for a role. This feature lets an admin decide, per role, which of the following actions should produce notifications, for each of the modules in scope:

- **Create** (a new item is added)
- **Update** (an existing item changes — includes status transitions)
- **Delete** (an item is removed)

Applies to these modules only (from the audit):

1. Contract Management (Vendor / Wyton) — audit §7
2. Phase Invoices — audit §8
3. Change Events — audit §9
4. Correspondence Tracking — Permit / Violation / RFI — audit §10
5. Scheduling — audit §11

Notifications for auth, chat, tasks, action plan, bidding, documents, meetings — **out of scope** for this iteration (add later if needed; the design supports it additively).

---

## 2. Non-goals

- Doesn't change who a notification is addressed to (recipient logic — RP / BIC / vendor / all admins — stays as it is today).
- Doesn't change the payload / template of any notification.
- Doesn't gate the underlying **action** (a role can still create a change event even if it doesn't get a notification for it). This is a notification-preference feature, not an access-control feature.
- Doesn't touch existing `{MODULE}_ALL` / `{MODULE}_VIEW` permission maps.

---

## 3. Current baseline (from the audit)

- **Roles seeded**: `admin`, `employee` ([lib/db.js](wyton-api/lib/db.js)). Vendor / consultant are user categories, not seeded roles today.
- **Permissions shape**: Mongo `Map<String, [String]>` on both `roles.permissions` and `users.permissions` (user override), e.g. `{ TASK_ROOM: ["TASK_ROOM_ALL", "TASK_ROOM_VIEW"] }`.
- **No permission middleware** exists — only [assertAdminAllAccess (controllers/users.js:19-47)](wyton-api/controllers/users.js). Routes gate with Hapi strategies `userAuth` / `adminAuth` only.
- **None** of the 5 target modules have permission keys today (only `PROJECT_CONTRACT` exists, for legacy bidding).

Implication: we're adding a **new** notification-preference layer next to permissions — not extending the existing `permissions` map. Two concerns, two collections.

---

## 4. Trigger → CRUD action mapping

Existing triggers do not all fit "create / update / delete" cleanly. Below is the proposed mapping. Every row here corresponds to an entry in [notifications-audit.md](notifications-audit.md) with its file:line intact.

### 4.1 Contract Management (module key: `CONTRACT_MANAGEMENT`)

| Audit # | Trigger | Action bucket | Recipient today |
|---|---|---|---|
| 35 | send_email_to_user — vendor already registered | Create | Existing vendor user |
| 36 | send_email_to_user — invite already pending | Create | Same target |
| 37 | send_email_to_user — brand-new user | Create | New invitee |
| 38 | Contract sent for signature | Create | Vendor + all admin/employee |
| 39 | Vendor submits insurance module | Create | All admin/employee |
| 40 | Vendor requests insurance waiver | Create | All admin/employee |
| 41 | Vendor marks contract signed | Update (status → signed) | All admin/employee |
| 42 | Phase status → REQUESTED | Update | Admin + employee (except actor) |
| 43 | Phase status → APPROVED | Update | Vendor |
| 44 | Phase status → EDITED_BY_VENDOR | Update | Admin + employee |
| 45 | Phase status → VENDOR_APPROVED | Update | Admin + employee |
| 46 | Cron — insurance expiring in 15 days | Update (system reminder) | Trade RP(s) + vendor user |
| — | (Future) Contract / phase deleted | Delete | (tbd) |

Note: contract-management flow is state-based; treating every phase / insurance status change as **Update** keeps the switch semantically clean.

### 4.2 Phase Invoices (module key: `PHASE_INVOICE`)

| Audit # | Trigger | Action bucket | Recipient today |
|---|---|---|---|
| 47 | Vendor raises phase invoice | Create | All BIC + RP on trade (except creator) |
| — | (Future) Invoice edited (amount/date) | Update | (tbd) |
| — | (Future) Invoice deleted / voided | Delete | (tbd) |

### 4.3 Change Events (module key: `CHANGE_EVENT`)

| Audit # | Trigger | Action bucket | Recipient today |
|---|---|---|---|
| 48 | Vendor creates change event | Create | All admin/employee |
| 49 | Admin/employee creates change event | Create | All vendors on the project |
| 50 | Vendor re-submits rejected change event | Update | All admin/employee |
| 51 | Change event → APPROVED | Update | Creator |
| 52 | Change event → REJECTED | Update | Creator |
| — | (Future) Change event deleted | Delete | (tbd) |

### 4.4 Correspondence Tracking (module keys: `PERMIT_TRACKING`, `VIOLATION_TRACKING`, `RFI_TRACKING`)

Same trigger shape for all three sub-modules — set preferences per sub-module so an admin can enable RFI notifications while muting Violations, etc.

| Audit # | Trigger | Action bucket | Recipient today |
|---|---|---|---|
| — | Item created | Create | (currently no notification — add if desired) |
| 53–55 | Priority toggle / nudge / follow-up / field update | Update | Watchers on `notificationWatchers` (primary skipped) |
| — | Item deleted | Delete | (currently no notification — add if desired) |

### 4.5 Scheduling (module key: `SCHEDULING`)

| Audit # | Trigger | Action bucket | Recipient today |
|---|---|---|---|
| 56 | Schedule activity created — new assignee | Create | Assignee |
| 57 | Schedule activity created — new approver | Create | Approver |
| 58 | Schedule updated — newly added assignees/approvers | Update | New assignees/approvers |
| — | (Future) Schedule item deleted | Delete | (tbd) |

---

## 5. Data model

### 5.1 New collection — `role_notification_settings`

One document per (role, module).

```json
{
  "_id": "ObjectId",
  "roleId": "ObjectId → roles",
  "module": "CONTRACT_MANAGEMENT",          // one of the module keys in §4
  "actions": {
    "create": { "email": true,  "push": true,  "inApp": true  },
    "update": { "email": false, "push": true,  "inApp": true  },
    "delete": { "email": false, "push": false, "inApp": true  }
  },
  "updatedBy": "ObjectId → users",
  "createdAt": "...",
  "updatedAt": "..."
}
```

Rationale for the shape:
- **Per-channel booleans** inside each action so an admin can allow "in-app only" for updates without also sending emails. If admins should only see one on/off per action, collapse to a single boolean — see §11 Q1.
- **Missing document = system default** (see §6). No need to seed a row for every combination on day one.

Indexes: `{ roleId: 1, module: 1 }` unique.

### 5.2 New collection — `user_notification_overrides` (optional user opt-out)

Mirrors the role/user permission pattern: a user can override their role's setting for themselves (mainly used to *silence*, not *enable* beyond role — see precedence in §6).

```json
{
  "_id": "ObjectId",
  "userId": "ObjectId → users",
  "module": "CONTRACT_MANAGEMENT",
  "actions": {
    "create": { "email": false, "push": false, "inApp": false },
    "update": { "email": false, "push": true,  "inApp": true  }
  },
  "createdAt": "...",
  "updatedAt": "..."
}
```

Indexes: `{ userId: 1, module: 1 }` unique.

### 5.3 System-default table

Ship a JS constant `DEFAULT_NOTIFICATION_MATRIX` in `lib/notificationDefaults.js` used when no `role_notification_settings` doc exists. Preserves today's behaviour: everything on.

```js
const ALL_ON = { create: {email:true,push:true,inApp:true},
                 update: {email:true,push:true,inApp:true},
                 delete: {email:true,push:true,inApp:true} };
module.exports.DEFAULT_NOTIFICATION_MATRIX = {
  CONTRACT_MANAGEMENT: ALL_ON,
  PHASE_INVOICE:       ALL_ON,
  CHANGE_EVENT:        ALL_ON,
  PERMIT_TRACKING:     ALL_ON,
  VIOLATION_TRACKING:  ALL_ON,
  RFI_TRACKING:        ALL_ON,
  SCHEDULING:          ALL_ON,
};
```

---

## 6. Precedence & resolution

To decide whether a given recipient user should receive a given notification:

```
resolveNotifyFlag(user, module, action, channel):
  1. If user_notification_overrides has (user._id, module) →
       return that doc.actions[action][channel]      // user opt-out wins
  2. Else if role_notification_settings has (user.roleId, module) →
       return that doc.actions[action][channel]      // role setting
  3. Else →
       return DEFAULT_NOTIFICATION_MATRIX[module].actions[action][channel]
```

Rules:
- User override is **filter only** — a user can turn *off* a notification their role receives. It cannot turn *on* a notification their role has disabled. (Prevents a user from bypassing an admin-configured mute.) — see §11 Q2 if admins want two-way user override.
- Actor is still excluded from their own event (existing behaviour, untouched).
- `admin` role is treated like any other role — an admin can mute themselves out of everything if they choose.

---

## 7. Backend integration

### 7.1 New helper module — `lib/notificationPreferences.js`

```js
// lib/notificationPreferences.js
const RoleNotifSetting = require("../model/roleNotificationSetting");
const UserNotifOverride = require("../model/userNotificationOverride");
const { DEFAULT_NOTIFICATION_MATRIX } = require("./notificationDefaults");

async function resolveNotifyFlag(user, module, action, channel) { ... }

async function filterRecipients(users, module, action, channel) {
  // returns the subset of users whose resolved flag is true
}

async function shouldNotify(user, module, action, channels /* array */) {
  // returns { email: bool, push: bool, inApp: bool } already resolved
}

module.exports = { resolveNotifyFlag, filterRecipients, shouldNotify };
```

Batch-load role settings and user overrides in one query per call, keyed by `roleId` and `userId` sets, to avoid N+1.

### 7.2 Trigger-site changes

The audit tells us the exact dispatchers to wrap. Each site converts

```js
await create_in_app_notification({ userId: recipients, module, ... });
await fcmPushNotification.sendNotificationToMultipleUsers(recipients, notification);
await sendNotificationEmail(email, subject, title, message, opts);
```

into a resolved-flag guarded fan-out. Suggested pattern (pseudo):

```js
const perUser = await Promise.all(recipients.map(async (u) => ({
  user: u,
  flags: await shouldNotify(u, MODULE, ACTION, ["email", "push", "inApp"]),
})));

const inAppTargets = perUser.filter(x => x.flags.inApp).map(x => x.user);
const pushTargets  = perUser.filter(x => x.flags.push ).map(x => x.user);
const emailTargets = perUser.filter(x => x.flags.email).map(x => x.user);

if (inAppTargets.length) await create_in_app_notification({ userId: inAppTargets, module, ... });
if (pushTargets.length ) await fcmPushNotification.sendNotificationToMultipleUsers(pushTargets, notification);
for (const t of emailTargets)  await sendNotificationEmail(t.email, subject, title, message, opts);
```

### 7.3 Exact files/functions to modify

| Module | Files · lines to wrap | Constant needed |
|---|---|---|
| Contract Management | [services/contractManagementService.js](wyton-api/services/contractManagementService.js) lines 524, 541, 574, 1869, 2576, 2683, 3274, 4082, 4106, 4148, 4191, 4744, 4754 | `MODULE="CONTRACT_MANAGEMENT"`, `ACTION` per §4.1 |
| Phase Invoices | [controllers/phaseInvoice.js:151,169](wyton-api/controllers/phaseInvoice.js) `raiseInvoice` | `PHASE_INVOICE` / `create` |
| Change Events | [services/changeEventServices.js](wyton-api/services/changeEventServices.js) lines 402, 418, 602, 677 | `CHANGE_EVENT` per §4.3 |
| Correspondence Tracking | [services/correspondenceTrackingRowActionServices.js:109](wyton-api/services/correspondenceTrackingRowActionServices.js) shared helper `emitTrackedCorrespondenceItemUpdateNotification` — accepts `itemType` today; use it to pick `PERMIT_TRACKING` / `VIOLATION_TRACKING` / `RFI_TRACKING` | per §4.4 |
| Scheduling | [services/schedulingServices.js:246,270,816](wyton-api/services/schedulingServices.js) `notifySchedulingAssignments`, `update_task` | `SCHEDULING` per §4.5 |

Also update the shared task dispatcher pattern in [services/taskServices.js:2861 `sendTaskNotifications`](wyton-api/services/taskServices.js) if it ever handles the 5 modules — currently it does not, so leave alone.

Cron jobs in [jobs/recurringTaskCron.js](wyton-api/jobs/recurringTaskCron.js) go through the same dispatchers, so wrapping the dispatchers covers cron implicitly.

### 7.4 Module catalogue

Add the seven new module keys to [lib/constant.js](wyton-api/lib/constant.js) alongside existing `notification_module` at line 586. Keys are shared between preferences and existing in-app notification module labels, so the frontend `src/constants/notificationModules.ts` stays in sync.

---

## 8. Admin API surface

All under `adminAuth`.

| Verb | Path | Purpose |
|---|---|---|
| `GET` | `/notification-settings/roles/:roleId` | Get role settings for all in-scope modules (returns merged with defaults so UI always has a full matrix) |
| `PUT` | `/notification-settings/roles/:roleId` | Upsert one or more `(module, actions)` entries |
| `GET` | `/notification-settings/roles/:roleId/modules/:module` | Single module |
| `DELETE` | `/notification-settings/roles/:roleId/modules/:module` | Reset to system default |
| `GET` | `/notification-settings/users/:userId` | User-level overrides (self or admin) |
| `PUT` | `/notification-settings/users/:userId` | Upsert user overrides |
| `DELETE` | `/notification-settings/users/:userId/modules/:module` | Reset override |

Self-endpoint variant for a signed-in user editing their own overrides:

| `GET` | `/user/me/notification-settings` | Under `userAuth` |
| `PUT` | `/user/me/notification-settings` | Under `userAuth` |

Request/response payload matches §5.1.

Validation: `Joi` schema enforcing `module ∈ {CONTRACT_MANAGEMENT, PHASE_INVOICE, CHANGE_EVENT, PERMIT_TRACKING, VIOLATION_TRACKING, RFI_TRACKING, SCHEDULING}` and `action ∈ {create, update, delete}`.

---

## 9. Web admin UI

New route `/admin/notification-settings` (admin-only, gated by existing `PermissionGuard` used across [App.tsx](wyton-web/src/App.tsx)).

Layout:
- **Role tabs** across the top (Admin, Employee, and any future roles).
- **Module table** below — one row per module, three column groups (Create / Update / Delete), each with three channel checkboxes (Email / Push / In-app).
- **Reset** button per row → `DELETE /notification-settings/roles/:roleId/modules/:module`.
- **Save** button batches diff into a single `PUT`.

Corresponding self-serve preferences on the User Profile page (existing [UserProfile.tsx](wyton-web/src/pages/UserProfile.tsx)) — same matrix, filled from role setting, editable to opt out. Cell showing "muted by admin" (disabled) when the role setting is `false` — user cannot re-enable.

Redux slice: add `notificationSettingsSlice` under `src/store/slices/`. Reuse `handleApiSuccess`/`handleApiError` from [services/api.ts](wyton-web/src/services/api.ts) for toast feedback.

---

## 10. Migration & rollout

1. Ship models + helper + admin API + admin UI behind no flag — defaults keep all notifications on, so **no behaviour change** on release.
2. Data migration: none. Absence of a doc = default (all on).
3. QA matrix: for each of the 7 modules × 3 actions × 3 channels, verify (a) admin can toggle from UI, (b) toggling off drops the corresponding channel from the send fan-out, (c) toggling back on restores it, (d) user override further restricts but cannot expand.
4. Instrument the helper: log each `resolveNotifyFlag` result with `{userId, module, action, channel, source: user|role|default, allowed}` at debug level so we can observe drift.
5. Backfill (optional): once admins have configured their preferences, add an `assertConfigured` env flag that logs a warn when a trigger falls back to defaults.

---

## 11. Open questions (need product decisions)

1. **Per-channel or per-action?** Spec above is per-channel per action (9 checkboxes per module). Simpler alternative: one boolean per action, applied to all channels equally (3 checkboxes per module). Recommend per-channel — matches the mixed-channel behaviour that already exists in the audit (some events are in-app only, some multi-channel).
2. **User override direction?** Filter-only (user can mute, not un-mute) matches how permissions work today. If a user should be able to opt-in beyond role, flip the resolution in §6.
3. **`delete` semantics for modules that don't emit delete notifications today?** Options: (a) leave the toggle inert until we add delete triggers; (b) drop the `delete` column for Phase Invoices / Scheduling until they support it. Recommend (a) — future-proof, and the UI can grey out inert cells.
4. **Scope of "recipient"** — the 5 modules today notify a mix of individual users (RP/BIC/vendor/creator) and everyone in a class ("all admin/employee"). For the class-based sends (e.g. change event #48 "all admin/employee"), the helper still runs per-user, so the resolution is naturally correct — every recipient is filtered by their own role setting. Confirm this is desired vs. a single role-level pre-check.
5. **Vendor role.** Vendors are not a seeded role today. Should we add a `vendor` seed role now so their notification prefs can be configured, or key vendor prefs off "any user without an admin/employee role"? Recommend seeding `vendor` and `consultant` roles as part of this feature — cleaner mental model.
6. **Audit log**: capture every change to `role_notification_settings` and `user_notification_overrides` in an audit collection? Recommended for admin surface.

---

## 12. Deliverables checklist

Backend
- [ ] Models: `model/roleNotificationSetting.js`, `model/userNotificationOverride.js`
- [ ] Constants: `lib/notificationDefaults.js`; new module keys in [lib/constant.js](wyton-api/lib/constant.js)
- [ ] Helper: `lib/notificationPreferences.js` with `filterRecipients`, `shouldNotify`
- [ ] Trigger-site wrapping (files listed in §7.3)
- [ ] Routes + controllers for §8 API
- [ ] Seed roles for `vendor`, `consultant` (if Q5 approved)

Web
- [ ] Slice: `src/store/slices/notificationSettingsSlice.ts`
- [ ] Page: `src/pages/admin/NotificationSettings.tsx` + route in [App.tsx](wyton-web/src/App.tsx)
- [ ] User self-service section in [UserProfile.tsx](wyton-web/src/pages/UserProfile.tsx)
- [ ] Navigation entry in [NewNavbar.tsx](wyton-web/src/components/layout/NewNavbar.tsx) admin submenu

QA
- [ ] Matrix test per §10.3
- [ ] Regression: existing notification triggers still fire when defaults hold
