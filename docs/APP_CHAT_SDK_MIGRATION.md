# Wyton App — Chat SDK Migration (Sendbird → Wyton Chat)

**Audience:** `wyton-app` (React Native)  
**Status:** Web + API already migrated. App still uses Sendbird and **must** be updated or chat will break against the new API.

Related: [FCM_CHAT_PUSH.md](./FCM_CHAT_PUSH.md)

---

## Why this is required

| Layer | Status |
|-------|--------|
| `wyton-api` | Sendbird removed. Chat uses `CHAT_*` + `/user/chat/*` |
| `wyton-web` | Uses custom client (`wytonChatClient` + `/user/chat/session`) |
| `wyton-app` | **Still on `@sendbird/chat` + old `/user/sendbird/*` paths** |

Old app API calls that **no longer exist** on API:

| Old (broken) | New |
|--------------|-----|
| `POST /user/sendbird/ensure-users` | `POST /user/chat/ensure-users` |
| `POST /user/sendbird/message-push` | `POST /user/chat/message-push` |
| `GET/PUT /user/sendbird/favourites` | `GET/PUT /user/chat/favourites` |

There is **no** Sendbird webhook anymore. Push is FCM via Wyton API only.

---

## Target architecture (match web)

```
┌─────────────┐   GET /user/chat/session    ┌─────────────┐
│  wyton-app  │ ──────────────────────────► │  wyton-api  │
└──────┬──────┘                             └──────┬──────┘
       │                                           │
       │  token + baseUrl + appId                  │ CHAT_APP_SECRET
       │                                           ▼
       │ HTTP /v1/* + WS /ws              ┌─────────────────┐
       └─────────────────────────────────►│ Wyton Chat SDK  │
                                          │ (CHAT_BASE_URL) │
                                          └─────────────────┘
       │
       │ after send (optional, keep parity with web)
       ▼
  POST /user/chat/message-push  →  FCM + in-app bell
```

**Do not** put `CHAT_APP_SECRET` in the app. Session is issued by Wyton API.

---

## 1. Config changes

### Remove

From `src/config/env.dev.ts`, `env.prod.ts`, `env.ts`:

```ts
SENDBIRD_APP_ID: '...'
export const SENDBIRD_APP_ID = config.SENDBIRD_APP_ID;
```

### Optional (docs / local only)

App does **not** need hard-coded chat base URL if it always uses session from API. If you want debug fallbacks:

```ts
// Optional — prefer values from GET /user/chat/session
CHAT_APP_ID?: string;      // e.g. wyton_dev
CHAT_BASE_URL?: string;    // e.g. https://chat.liftrex.com
```

---

## 2. Dependencies

### Remove

```bash
npm uninstall @sendbird/chat
# also remove any @sendbird/* UI kits if present
```

### Add

No npm Sendbird package. Port (or share) a RN-compatible client modeled on:

- `wyton-web/src/services/wytonChatClient.ts`
- `wyton-web/src/services/chatConnection.ts`

Use React Native `WebSocket` (global) + your existing `api` axios instance.

---

## 3. Connection layer (replace Sendbird connect)

### Replace these files / APIs

| Current | Replace with |
|---------|----------------|
| `src/lib/sendbirdClient.ts` | `src/lib/chatConnection.ts` + `src/services/wytonChatClient.ts` (RN) |
| `ensureSendbirdConnected(userId)` | `connectChat(userId)` → `GET /user/chat/session` then WS/HTTP connect |
| `disconnectSendbird()` | `disconnectChat()` |
| `isSendbirdConnectionOpen()` | `isChatConnected()` |

### Session contract (same as web)

```http
GET /user/chat/session
Authorization: Bearer <wyton-jwt>
```

Response shape (approx):

```json
{
  "token": "<chat-jwt>",
  "baseUrl": "https://chat.liftrex.com",
  "appId": "wyton_dev",
  "user": {
    "id": "...",
    "userId": "<same id used today>",
    "displayName": "...",
    "avatar": "..."
  }
}
```

Then:

1. HTTP: `Authorization: Bearer <token>` against `baseUrl`
2. WS: `ws(s)://.../ws?token=...&appId=...`

**User id scheme:** keep the same ids web uses (typically Mongo `_id` string). Must match members used for FCM `recipientUserIds`.

### Call sites to update

- `src/store/slices/authSlice.ts` — logout → `disconnectChat`
- `src/notifications/PushNotificationHandler.tsx`
- `src/hooks/useSendbirdPresenceLifecycle.ts` → rename / rewire to chat presence
- Any screen that calls `ensureSendbirdConnected`

---

## 4. Messaging / channels / UI

### Core service

| Current | Action |
|---------|--------|
| `src/services/chat.ts` | Keep UI helpers; swap Sendbird channel ops for `wytonChat.*` |
| `src/screens/chat/sendbird/*` | Rename folder to `chat/` (or keep UI, drop SDK types) |
| `src/screens/chat/sendbird/SnapbirdChatScreen.tsx` | Drive off Wyton chat client |
| `src/screens/Chat-room.tsx` | Same |

### API path fixes in `src/services/chat.ts` (minimum / immediate)

Even before full SDK swap, **update these or pushes/ensure-users fail**:

```ts
// OLD
await api.post('/user/sendbird/ensure-users', { userIds });
await api.post('/user/sendbird/message-push', { ... });

// NEW
await api.post('/user/chat/ensure-users', { userIds });
await api.post('/user/chat/message-push', { ... });
```

`message-push` body (unchanged shape):

```ts
{
  recipientUserIds: string[];
  channelUrl?: string;
  channelName?: string;
  messagePreview?: string;
  senderDisplayName?: string;
}
```

### Suggested `wytonChat` methods to mirror (from web)

- `listChannels` / `getChannel` / `createDirect` / `createGroup`
- `listMessages` / `sendMessage` / `editMessage` / `deleteMessages`
- `toggleReaction` / `markRead` / `markDelivered`
- handlers: `message_received`, `message_sent`, `typing_*`, `read_receipt_updated`, `message_delivered`, `user_online` / `user_offline`

Avoid refetching full history after every successful send (web already fixed that).

---

## 5. Favourites

| Current | New |
|---------|-----|
| `GET/PUT /user/sendbird/favourites` | `GET/PUT /user/chat/favourites` |
| Sendbird user metadata chunks in `chatFavourites.ts` | Prefer chat platform `/v1/me/favourites` (web) **or** Wyton API `/user/chat/favourites` |

**Recommended:** match web — use chat SDK `GET/PUT /v1/me/favourites` after session connect.  
Keep `/user/chat/favourites` only as fallback/migration.

Update `src/lib/chatFavourites.ts` accordingly; remove `loadFavouritesFromSendbird` / metaData chunking.

---

## 6. Presence

Sendbird presence stack to replace:

- `src/services/sendbirdPresence.ts`
- `src/services/sendbirdPresenceStore.ts`
- `src/types/sendbirdPresence.ts`
- `src/hooks/useSendbirdPresenceLifecycle.ts`
- `src/hooks/useUserPresence.ts` / `useLastSeen.ts`

Wyton chat exposes:

- WS: `user_online` / `user_offline`
- HTTP: `GET /v1/users/:userId/presence`

Map app “last seen” UI to that (or drop Sendbird meta key `lastSeen`).

---

## 7. Push / notifications

| Current | Action |
|---------|--------|
| `src/notifications/sendbirdPushBridge.ts` | Replace with Wyton WS `message_received` local notify **or** rely on FCM only |
| `attachSendbirdIncomingPushBridge` | Rename → chat incoming bridge using `wytonChat` handlers |
| Deep links reading `sendbird_channel_url` / `sendbirdChannelUrl` | Prefer `channelUrl` (FCM `data.channelUrl` / `redirectUrl`) — keep old keys briefly for back-compat |
| FCM token setup | Keep `POST /user/setup-fcm-token` with `deviceType: "ios" \| "android"` |

After send on app, call `POST /user/chat/message-push` (same as web) so recipients get FCM when offline.

---

## 8. Polls / events (Sendbird-specific)

These use `@sendbird/chat/poll` and Sendbird message APIs:

- `src/lib/chatPoll.ts`
- `src/lib/pollVoters.ts`
- `src/screens/chat/CreatePollScreen.tsx`
- `src/screens/chat/PollVotesScreen.tsx`
- `src/screens/chat/CreateEventScreen.tsx`
- `src/components/chatRoom/PollMessageBubble.tsx`

**Decision needed:**

1. **Port polls** onto Wyton chat metadata / custom message types, or  
2. **Hide/disable** poll & calendar-event chat features until chat platform supports them.

Web custom chat does not currently use Sendbird PollModule — don’t assume polls work after cutover.

---

## 9. Types / imports cleanup

Replace all:

```ts
import ... from '@sendbird/chat'
import ... from '@sendbird/chat/groupChannel'
import ... from '@sendbird/chat/message'
import ... from '@sendbird/chat/poll'
```

With app-local types (mirror `wyton-web/src/types/chatSdk.ts`):

- `ChatChannel`, `ChatMessage`, `GroupChannelHandler`, reactions, read receipts

Rename folders/files for clarity (optional but recommended):

- `screens/chat/sendbird/` → `screens/chat/ui/` or `screens/chat/thread/`
- `sendbirdChatHelpers.ts` → `chatHelpers.ts`
- `sendbirdPushBridge.ts` → `chatPushBridge.ts`

---

## 10. Suggested implementation order

1. **Hotfix (today):** update API paths in `services/chat.ts` + `chatFavourites.ts` to `/user/chat/*` so ensure-users / push / favourites work against new API (still on Sendbird realtime until step 2).
2. **Port client:** RN `wytonChatClient` + `connectChat` from `/user/chat/session`.
3. **Swap Chat-room / Snapbird screen** off `@sendbird/chat` onto new client.
4. **Presence + push bridge** off Sendbird handlers.
5. **Favourites** → `/v1/me/favourites` or `/user/chat/favourites`.
6. **Polls/events:** port or disable.
7. **Remove** `@sendbird/chat`, `SENDBIRD_APP_ID`, dead Sendbird files.
8. **QA:** DM + group send/receive, FCM cold start deep link (`channelUrl`), favourites, typing, read/delivery ticks, logout disconnect.

---

## 11. File checklist (app)

### Must change / replace

- [ ] `src/config/env.ts` / `env.dev.ts` / `env.prod.ts` — remove `SENDBIRD_APP_ID`
- [ ] `src/lib/sendbirdClient.ts` — replace
- [ ] `src/services/chat.ts` — paths + SDK calls
- [ ] `src/lib/chatFavourites.ts` — paths + storage
- [ ] `src/screens/Chat-room.tsx`
- [ ] `src/screens/chat/sendbird/**`
- [ ] `src/notifications/sendbirdPushBridge.ts` + `PushNotificationHandler.tsx`
- [ ] `src/notifications/pushNavigationState.ts` — channel url keys
- [ ] `src/services/sendbirdPresence*.ts` + hooks
- [ ] `src/store/slices/authSlice.ts`
- [ ] `package.json` — remove `@sendbird/chat`

### Review / decide

- [ ] Poll + event chat features
- [ ] Optimistic media upload paths that assume Sendbird file messages
- [ ] Any hardcoded `sendbird_group_channel_` URL assumptions

---

## 12. Quick reference — Wyton API chat endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/user/chat/session` | Issue chat token + baseUrl + appId |
| `POST` | `/user/chat/ensure-users` | Ensure users on chat platform `{ userIds: string[] }` |
| `POST` | `/user/chat/message-push` | FCM + in-app for recipients |
| `GET`/`PUT` | `/user/chat/favourites` | Legacy Mongo favourites |
| `POST` | `/user/setup-fcm-token` | Register device FCM token |

Chat platform (after session), base = session `baseUrl`:

| Method | Path |
|--------|------|
| `GET` | `/v1/me` |
| `GET`/`POST` | `/v1/channels`, `/v1/channels/direct`, `/v1/channels/groups` |
| `GET`/`POST` | `/v1/channels/:id/messages` |
| `POST` | `/v1/channels/:id/read`, `/delivered`, `.../reactions` |
| `GET`/`PUT` | `/v1/me/favourites` |
| WS | `/ws?token=...&appId=...` |

---

## 13. Web reference implementations

Copy patterns from:

| Concern | Web file |
|---------|----------|
| Session connect | `wyton-web/src/services/chatConnection.ts` |
| HTTP + WS client | `wyton-web/src/services/wytonChatClient.ts` |
| Channel/message helpers + push | `wyton-web/src/services/chat.ts` |
| Types | `wyton-web/src/types/chatSdk.ts` |
| UI orchestration | `wyton-web/src/pages/ChatRoom/ChatRoom.tsx` |

---

*Generated for app migration after Sendbird removal from `wyton-api` and `wyton-web`.*
