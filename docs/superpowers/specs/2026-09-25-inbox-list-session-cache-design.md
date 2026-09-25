# Inbox List Session Cache — Design

**Date:** 2026-09-25  
**Status:** Approved for planning  
**Goal:** When the user leaves `/inbox` and returns, previously fetched email lists must appear immediately. Only new (or deleted) mail is pulled via existing delta sync / silent merge — no full list reload spinner for a cached view.

---

## Problem

`Inbox.tsx` already caches first-page (and prefetched tab) lists in `listCacheRef` for instant **in-page** tab switches, with silent merge refresh on cache hit.

That cache is a component `useRef`. React Router unmounts the lazy `/inbox` route when navigating away, so the Map is discarded. On remount, cache miss → full `fetchGmailInbox` + `MailLoading`.

---

## Decision

**Approach A — session memory outside the Inbox page component.**

Lift the list cache (and related suppress sets) into a **module-level session store** that survives route unmount for the life of the SPA tab.

**Not chosen:**

| Alternative | Why not |
|---|---|
| Keep Inbox mounted / keep-alive | Keeps SSE, polls, and a large tree alive; awkward with current React Router setup |
| Persist to localStorage / IndexedDB | Overkill for leave-and-return; stale and privacy concerns |
| Move live list into Redux as source of truth | Large rewrite of merge/delta logic; re-render risk; `inboxSlice` today is connection + badge metadata only |

---

## Architecture

### New module

`wyton-web/src/pages/Inbox/inboxListSessionCache.ts`

Session-scoped API (RAM only; cleared on explicit invalidation):

| API | Role |
|---|---|
| `get(key)` | Read `{ messages, nextToken, estimate }` |
| `set(key, entry)` | Write / replace entry |
| `has(key)` | Prefetch skip check |
| `clear()` | Full invalidation |
| `entries()` / `patchAll(fn)` | Label/star/read patches across all keys |
| `suppressMessageIds` / `suppressThreadIds` | Keep deleted/archived rows from resurrecting via cache or silent merge |
| `purgeSuppressed()` | Strip suppressed ids from all cached lists |

Entry shape matches today’s ref. The module must not import `Inbox.tsx` (avoid cycles). Use a generic row type or a shared type already used for list rows (e.g. extend/`GmailInboxRow`-compatible):

```ts
type InboxListCacheEntry<T = { id: string | number; threadId?: string }> = {
  messages: T[];
  nextToken: string | null;
  estimate: number;
};
```

List keys remain those produced by existing `buildListRequest().key` (mailbox, view, filter, search, conversation settings, preferLocal, etc.).

### `Inbox.tsx` changes

- Replace `listCacheRef` and the local suppress Sets with the session module.
- Keep `gmailMessages` as React state for rendering (no Redux list).
- Remount / filter-change effect (existing cache-hit branch) already does: hydrate from cache → `inboxLoading = false` → `fetchGmailInbox({ silent: true, merge: true })`. After the lift, that branch works across route visits.
- **Required:** on remount, when a cache hit exists for the current key, hydrate before (or instead of) the empty-list + `MailLoading` first paint — including settling any “first load” flags that currently gate the initial loader (e.g. `mailboxFirstLoadSettled`) so users do not flash a full reload UX.
- Continue calling `runDeltaSync()` / push refresh paths as today so **new** mail prepends without re-fetching the full page.

### Call sites that must keep clearing

Preserve existing `listCacheRef.current.clear()` semantics by calling `inboxListSessionCache.clear()` on:

- Gmail disconnect
- Active mailbox switch / mailbox epoch change
- Label-counts effect that already clears when the signed-in account changes
- Any explicit full refresh / reset that clears the cache today

**Required once the cache is module-scoped:** also clear on logout / auth reset. Today the ref dies with unmount; a module Map can otherwise leak mailbox lists across users in the same SPA tab. Wire the smallest existing logout path (e.g. auth logout handler or Inbox disconnect cleanup) — do not add unrelated auth features.

---

## Data flow

```
Visit /inbox (first time, cold)
  → cache miss → full fetch → set(key, entry) → render

Switch category tab (same mount)
  → cache hit → hydrate → silent merge (unchanged)

Leave /inbox → other route
  → Inbox unmounts; session Map retained

Return /inbox
  → cache hit → hydrate immediately (no full reload spinner)
  → silent merge + delta sync (new / deleted only)
```

---

## Out of scope

- Surviving full browser refresh / hard reload
- Keeping the Inbox component mounted while on other routes
- Storing message bodies or conversation timeline in the session list cache
- Changing Gmail API contracts or backend history sync

---

## Success criteria

1. Leave `/inbox` → navigate elsewhere → return: previously fetched list for the current key shows immediately without a full reload spinner.
2. New mail still appears via existing delta sync / silent merge / push paths.
3. Deleted or archived rows remain suppressed after remount (no resurrection from cache).
4. In-page tab switches and prefetch remain as fast as today.
5. Disconnect / mailbox switch still forces a fresh list (cache cleared).

---

## Testing (manual)

1. Open `/inbox`, wait until Primary (or current tab) loads.
2. Navigate to another app route (e.g. `/chat-room`), then back to `/inbox`.
3. Expect: list visible immediately; network may show delta/history or silent page-1 merge, not a cold full reload UX.
4. Archive/delete a row, leave and return: row stays gone.
5. Switch mailbox or disconnect Gmail: list does not show the previous mailbox’s cached rows.
6. Switch category tabs: still instant when prefetched/cached.

---

## Implementation notes

- Prefer a thin adapter so `Inbox.tsx` diffs stay mechanical (`listCacheRef.current.X` → module `X`).
- Do not expand `inboxSlice` with the message array unless a later need requires shared selectors outside Inbox.
- Keep cache keys identical to `buildListRequest().key` so prefetch and remount share entries.
