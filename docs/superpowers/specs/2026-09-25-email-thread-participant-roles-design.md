# Email Thread Participant Roles — Design

**Date:** 2026-09-25  
**Status:** Approved for planning  
**Goal:** Automatically manage conversation participants and permissions from email activity (From/To/Cc, replies, inbound client mail) and manual invites, including a new **Permanent Member** role and multi-Admin ACL on the initial internal send.

**Source requirements:** Product case doc (Email Thread Participant Management) + clarifications in design review.

**Related existing docs:** `docs/inbox-email-participant-attach-flow.md` (current attach/update/remove API; will need update after implementation).

---

## Problem

Today `email_conversations.participants` only supports `admin` | `full` | `readonly`. Behavior gaps vs product:

| Area | Today | Needed |
|------|--------|--------|
| Initial seed | Owner `admin`; To/Cc Wyton users `full` | All internal From/To/Cc → **Admin** |
| Permanent | Does not exist | New role with irreversible membership |
| Client inbound | No auto Permanent | External From → To/Cc internals → Permanent (not Admins) |
| Full first reply | No promotion | Full → Permanent after first successful outbound reply |
| Remove / change perm | Only `ownerUserId` | **Any Admin** for Full/Read Only only |
| Manual add | Picker Full vs Read-only | Default Full; Read-only still optional |
| Role audit UI | N/A | **v1: server logs only** |

---

## Decision

**Approach: extend the existing conversation ACL** in `emailConversationService` + model enum.

**Not chosen:**

| Alternative | Why not |
|---|---|
| Separate membership / events collections | Heavier for v1; timeline UI deferred |
| `isPermanent` flag beside 3 permissions | Fights a single role ladder and UI labeling |

---

## Roles

Stored on `participants[].permission`:

| Value | Label (UI) |
|-------|------------|
| `admin` | Admin |
| `permanent` | Permanent Member |
| `full` | Full Access |
| `readonly` | Read Only |

### Capability matrix

| Action | Admin | Permanent | Full | Read Only |
|--------|-------|-----------|------|-----------|
| View thread / emails / participants | ✓ | ✓ | ✓ | ✓ |
| Comments / notes | ✓ | ✓ | ✓ | ✓ |
| Send / reply / forward | ✓ | ✓ | ✓ | ✗ |
| Add participants | ✓ | ✓ | ✓ | ✗ |
| Change permission (Full ↔ Read Only only) | ✓ | ✗ | ✗ | ✗ |
| Remove (Full / Read Only only) | ✓ | ✗ | ✗ | ✗ |
| Remove or downgrade Admin | ✗ | ✗ | ✗ | ✗ |
| Remove or downgrade Permanent | ✗ | ✗ | ✗ | ✗ |

**`canWrite`** (send/reply/add/comment write paths that already gate on write): true for `admin` | `permanent` | `full`.

**Admin ACL gate** (remove / change permission): actor’s permission is `admin` (not only `ownerUserId === actor`). Target must be `full` or `readonly`.

`ownerUserId` remains the conversation starter for mailbox/host identity; it is **not** the sole ACL controller anymore.

---

## Auto-assignment rules

### Internal user definition

Unchanged: addresses that resolve to Wyton org users via existing `resolveWytonUsersOnThread` / equivalent. External addresses never become participants.

### Role apply helpers (central)

Service helpers (names illustrative):

- `roleRank(permission)` — for upgrades among non-admin roles only  
- `applyParticipantRole(conversation, userId, nextRole, reason)` — never downgrades Permanent; never changes Admin via auto-rules except Case 1 seed onto new rows  
- `logRoleChange({ threadId, userId, from, to, reason })` — structured server log (v1)

**Auto-upgrade priority (non-Admin only):** Permanent > Full > Read Only.  
**Admins are never auto-promoted to Permanent** (product clarification). Permanent still cannot be downgraded by anyone.

### Case 1 — Initial internal thread (outbound create / first seed)

When an internal user starts a thread and conversation is created/seeded from messages:

- Every internal address on **From, To, CC** → participant with **`admin`**
- Externals skipped

### Case 2 — External client email creates Permanents

When processing inbound mail where **From is not a Wyton user**:

- Every internal on **To, CC** who is not already Admin → ensure participant with **`permanent`**
- Already Admin → leave Admin
- Already Permanent → no-op

### Case 3 — Existing Full / Read Only on client To/Cc

Same inbound path: upgrade Full or Read Only → Permanent. Admins unchanged.

### Case 4 — Read Only restrictions

Enforce server-side (`canWrite` / Admin ACL) and hide/disable send, reply, reply-all, forward, add, remove, permission change in UI when actor is Read Only.

### Case 5 — Manual “+” add

- Default permission: **`full`**
- Optional: inviter may still choose **`readonly`**
- Actor must be Admin, Permanent, or Full (`canWrite`)

### Case 6 — Full’s first outbound reply

Immediately after a successful outbound send by a participant whose current role is **`full`**:

- Upgrade that participant to **`permanent`** (once)
- Admin / Permanent / Read Only unchanged by this rule

### Later replies — newly mentioned internals (existing helper)

`ensureReadonlyParticipantsFromReplyRecipients` stays **Read Only** for Wyton users newly added on a later reply’s To/Cc who were not already participants. That is not Case 1 (initial seed). Case 6 still applies to the **sender** if they were Full.

---

## Conversation create / update triggers

Create conversation if missing and apply relevant rules on:

1. **Outbound send** (compose / reply send path) — Case 1 seed when conversation is first created from the thread’s messages; Case 6 promotion for Full senders  
2. **Inbound sync** when message From is external — Cases 2–3  
3. **Invite / comment / reply collab** lazy paths already used today — run the same seed/promote helpers when those create or touch the conversation  

Do not rely only on opening the participants bar.

---

## API / model changes

### Model (`emailConversationModel.js`)

```js
permission: {
  type: String,
  enum: ["admin", "permanent", "full", "readonly"],
  default: "full",
}
```

Existing rows stay valid; no migration required beyond enum expand. Optional one-time backfill is out of scope unless product requests it.

### Service (`emailConversationService.js`)

| Function | Change |
|----------|--------|
| `seedThreadRecipientsAsParticipants` | Seed internals as **`admin`** (not `full`) |
| `addParticipants` | Default `full`; still accept `readonly`; allow actors with `permanent` |
| `updateParticipantPermission` | Any **Admin** actor; target only `full`/`readonly`; never Admin/Permanent |
| `removeParticipant` | Any **Admin** actor; target only `full`/`readonly`; never Admin/Permanent |
| New: inbound external apply | Promote/add Permanent for To/Cc internals |
| New: after outbound reply | Full → Permanent for sender |
| `canWrite` | Include `permanent` |

### FE types / UI

- `ConversationParticipant.permission` includes `"permanent"`
- Labels + disable remove/permission controls for Admin and Permanent targets
- Admin controls enabled for any Admin actor (not only owner / `isOwner`)
- Manual add dialog: default Full; keep Read-only option
- Read Only: disable send/reply/forward/add as today via `canWrite` / explicit checks

### Audit (v1)

Server-side structured logs only, e.g.:

```
[email-conversation-role] threadId=… userId=… from=full to=permanent reason=client_inbound_to_cc
```

No timeline UI events in v1.

---

## Key files

| Layer | Path |
|-------|------|
| Model | `wyton-api/model/emailConversationModel.js` |
| Service | `wyton-api/services/emailConversationService.js` |
| Controllers / send / sync hooks | `wyton-api/controllers/emailInbox.js`, gmail sync / send paths that already call conversation helpers |
| FE types | `wyton-web/src/utils/emailConversationTimeline.ts` |
| Participants UI | `wyton-web/src/components/email/EmailConversationCollaboration.tsx` |
| Host gates | `wyton-web/src/pages/Inbox/Inbox.tsx` |
| Doc update after ship | `docs/inbox-email-participant-attach-flow.md` |

---

## Out of scope (v1)

- Role-change events in the conversation timeline UI  
- Profile popover multi-mailbox account list changes (separate request)  
- Promoting Admin → Permanent on client mail  
- Changing Permanent’s ACL to match Admin  

---

## Testing focus

1. Internal first send: From/To/Cc Wyton users all `admin`; externals absent  
2. Any Admin can remove a Full and change Full ↔ Read Only; cannot remove Admin or Permanent  
3. External From inbound: To/Cc internals become Permanent; existing Admin stays Admin  
4. Manual add defaults Full; Read-only still works  
5. Full sends first reply → Permanent; second reply no-op  
6. Read Only cannot send/reply/add (API 403 + UI disabled)  
7. Permanent can add participants and send; cannot remove or change permissions  

---

## Resolved clarifications

| Topic | Decision |
|-------|----------|
| Case 1 Admins | Multiple Admins from initial From/To/Cc |
| Who removes | Any Admin; Full/Read Only only; never Admin/Permanent |
| Permanent ACL | Same write/add as Full; no remove/perm change |
| Manual add | Default Full; Read-only optional |
| Create triggers | Outbound send + inbound external + invite/comment/reply |
| Admin vs Permanent auto | Never auto-promote Admin → Permanent |
| Audit | Server logs only for v1 |
