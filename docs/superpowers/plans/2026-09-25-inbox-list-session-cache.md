# Inbox List Session Cache Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Survive `/inbox` route unmount by lifting the in-memory list cache (and suppress sets) out of `Inbox.tsx` so remount paints cached mail immediately and only delta/silent-merge pulls new mail.

**Architecture:** Module-level `Map` + suppress `Set`s in `inboxListSessionCache.ts`. `Inbox.tsx` swaps `listCacheRef.current` / suppress refs for those exports. Logout clears via `performForcedLogout`. Cache-hit hydrate also settles first-load flags to avoid `MailLoading` flash.

**Tech Stack:** React, TypeScript, existing Inbox fetch/delta paths (no Redux list).

## Global Constraints

- Session RAM only — no localStorage/IndexedDB for list rows.
- Do not move live `gmailMessages` into Redux.
- Keep `buildListRequest().key` as the cache key.
- Clear on disconnect, mailbox switch, account change, and logout.
- Avoid importing `Inbox.tsx` from the cache module (no cycles).

## File map

| File | Responsibility |
|---|---|
| Create: `wyton-web/src/pages/Inbox/inboxListSessionCache.ts` | Session Map + suppress Sets + `clearInboxListSessionCache()` + `rebindInboxListSessionAccount()` |
| Modify: `wyton-web/src/pages/Inbox/Inbox.tsx` | Use module cache; settle first-load on hydrate; remount-safe account rebind |
| Modify: `wyton-web/src/utils/performForcedLogout.ts` | Clear session cache on logout |

---

### Task 1: Session cache module

**Files:**
- Create: `wyton-web/src/pages/Inbox/inboxListSessionCache.ts`

**Interfaces:**
- Produces: `inboxListCache`, `suppressedInboxMessageIds`, `suppressedInboxThreadIds`, `clearInboxListSessionCache()`, `InboxListCacheEntry<T>`

- [ ] **Step 1: Create module**

```ts
export type InboxListCacheRow = {
  id: string | number;
  threadId?: string;
};

export type InboxListCacheEntry<T extends InboxListCacheRow = InboxListCacheRow> = {
  messages: T[];
  nextToken: string | null;
  estimate: number;
};

/** Survives Inbox route unmount for the SPA tab lifetime. */
export const inboxListCache = new Map<string, InboxListCacheEntry<any>>();
export const suppressedInboxMessageIds = new Set<string>();
export const suppressedInboxThreadIds = new Set<string>();

export function clearInboxListSessionCache(): void {
  inboxListCache.clear();
  suppressedInboxMessageIds.clear();
  suppressedInboxThreadIds.clear();
}
```

- [ ] **Step 2: Commit**

```bash
git add wyton-web/src/pages/Inbox/inboxListSessionCache.ts
git commit -m "Add module-level inbox list session cache."
```

---

### Task 2: Wire Inbox.tsx to session cache + first-paint hydrate

**Files:**
- Modify: `wyton-web/src/pages/Inbox/Inbox.tsx`

**Interfaces:**
- Consumes: exports from Task 1

- [ ] **Step 1: Import and remove local refs**

Add import:

```ts
import {
  inboxListCache,
  suppressedInboxMessageIds,
  suppressedInboxThreadIds,
  clearInboxListSessionCache,
} from "./inboxListSessionCache";
```

Remove:

```ts
const listCacheRef = useRef<Map<...>>(new Map());
const suppressedMessageIdsRef = useRef(new Set<string>());
const suppressedThreadIdsRef = useRef(new Set<string>());
```

- [ ] **Step 2: Mechanical renames**

| From | To |
|---|---|
| `listCacheRef.current` | `inboxListCache` |
| `suppressedMessageIdsRef.current` | `suppressedInboxMessageIds` |
| `suppressedThreadIdsRef.current` | `suppressedInboxThreadIds` |

Where all three are cleared together (label-counts account-change effect), prefer:

```ts
clearInboxListSessionCache();
```

Other `inboxListCache.clear()`-only sites stay clear-lists-only unless they already cleared suppress sets (keep prior semantics).

- [ ] **Step 3: First-paint hydrate**

In both cache-hit paths in the main list `useEffect` (~cachedEarly and ~cached branches), after setting messages / loading false, also:

```ts
setMailboxFirstLoadSettled(true);
```

so `(inboxLoading \|\| !mailboxFirstLoadSettled) && conversationDisplayList.length === 0` does not show `MailLoading initial` when cache already has rows.

- [ ] **Step 4: Smoke-check TypeScript**

Run: `cd wyton-web && npx tsc --noEmit -p tsconfig.json 2>&1 | head -40`  
Expected: no errors in `Inbox.tsx` / `inboxListSessionCache.ts` (or only pre-existing unrelated errors).

- [ ] **Step 5: Commit**

```bash
git add wyton-web/src/pages/Inbox/Inbox.tsx
git commit -m "Persist inbox list cache across route remounts."
```

---

### Task 3: Clear cache on logout

**Files:**
- Modify: `wyton-web/src/utils/performForcedLogout.ts`

**Interfaces:**
- Consumes: `clearInboxListSessionCache` from Task 1

- [ ] **Step 1: Clear on forced logout**

```ts
import { clearInboxListSessionCache } from "@/pages/Inbox/inboxListSessionCache";
// inside performForcedLogout, before or after dispatch(logout()):
clearInboxListSessionCache();
```

- [ ] **Step 2: Commit**

```bash
git add wyton-web/src/utils/performForcedLogout.ts
git commit -m "Clear inbox list session cache on logout."
```

---

### Task 4: Manual verification

- [ ] Open `/inbox`, wait for list
- [ ] Go to another route, return to `/inbox` — list appears without full reload spinner; network may show silent/delta only
- [ ] Archive/delete a row, leave and return — row stays gone
- [ ] Switch mailbox or disconnect — previous mailbox cache not shown
- [ ] Logout and login as another user — no leaked list

---

## Spec coverage

| Spec requirement | Task |
|---|---|
| Module-level session Map | 1 |
| Suppress sets survive remount | 1–2 |
| Inbox uses module; no Redux list | 2 |
| First-paint / MailLoading settle | 2 |
| Clear on disconnect / mailbox / account | 2 (existing sites) |
| Clear on logout | 3 |
| Manual success criteria | 4 |
