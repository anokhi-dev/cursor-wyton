# Email Thread Participant Roles Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans or superpowers:subagent-driven-development. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement multi-Admin seed, Permanent Member role, Admin ACL (remove/change Full & Read Only), and auto-promotion rules from the approved design.

**Architecture:** Extend `participants[].permission` enum with `permanent`; centralize role helpers in `emailConversationService.js`; hook outbound send + Gmail metadata upsert for Cases 1/2/3/6; update FE labels and ACL gates.

**Tech Stack:** Node/Express, Mongo/Mongoose, React/TypeScript (existing Inbox conversation overlay).

## Global Constraints

- Never auto-promote Admin → Permanent.
- Never remove or downgrade Admin or Permanent.
- Any Admin (not only `ownerUserId`) may remove/change Full and Read Only.
- Permanent has same write/add as Full; no remove/perm change.
- Manual add defaults to Full; Read-only remains optional.
- Audit v1 = structured `console` logs only.
- Later-reply new internals stay Read Only via existing helper.

## File map

| File | Responsibility |
|---|---|
| `wyton-api/model/emailConversationModel.js` | Add `permanent` to enum |
| `wyton-api/services/emailConversationService.js` | Role helpers, seed admin, ACL, inbound/outbound promote |
| `wyton-api/services/gmailSyncService.js` | Call inbound rules after metadata upsert |
| `wyton-api/controllers/users.js` | After send: seed + Full→Permanent |
| `wyton-web/src/utils/emailConversationTimeline.ts` | Type includes permanent |
| `wyton-web/src/components/email/EmailConversationCollaboration.tsx` | Labels + ACL UI |
| `docs/inbox-email-participant-attach-flow.md` | Sync with new rules |

---

### Task 1: Model + service core (roles, seed, ACL)

**Files:**
- Modify: `wyton-api/model/emailConversationModel.js`
- Modify: `wyton-api/services/emailConversationService.js`

- [x] Add `permanent` to permission enum
- [x] `canWrite` includes `permanent`
- [x] `seedThreadRecipientsAsParticipants` → `admin`
- [x] `updateParticipantPermission` / `removeParticipant`: actor must be Admin participant; target only full/readonly
- [x] Helpers: `logRoleChange`, `applyExternalInboundParticipantRules`, `promoteFullSenderToPermanentAfterSend`, `ensureThreadConversationSeeded`
- [x] Export new helpers

### Task 2: Wire outbound + inbound triggers

**Files:**
- Modify: `wyton-api/controllers/users.js` (after successful send)
- Modify: `wyton-api/services/gmailSyncService.js` (after `upsertMessageMetadata` when From external)

- [x] After send: ensure seed + promote Full sender → Permanent
- [x] After inbound metadata upsert: best-effort inbound Permanent rules

### Task 3: Frontend ACL + labels

**Files:**
- Modify: `wyton-web/src/utils/emailConversationTimeline.ts`
- Modify: `wyton-web/src/components/email/EmailConversationCollaboration.tsx`

- [x] Permanent label; protect Admin/Permanent from remove/perm UI
- [x] `isAdmin` from actor permission `admin` (not only owner)
- [x] Default Full on add; keep Read-only option
- [x] Update popup note copy

### Task 4: Doc sync

**Files:**
- Modify: `docs/inbox-email-participant-attach-flow.md`

- [x] Reflect permanent role + Admin ACL + seed defaults
