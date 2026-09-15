# FCM Push Notifications for Wyton Chat

Wyton uses a **custom chat platform** for real-time messaging and the **Wyton API** for FCM delivery after a message is sent (client path).

## Flow

```
Wyton Web ──HTTP/WS──► Chat platform (CHAT_BASE_URL)
     │
     │ POST /user/chat/message-push
     ▼
Wyton API ──Firebase Admin──► FCM → recipients
```

## API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/user/chat/session` | JWT | Issue chat JWT + baseUrl/appId |
| POST | `/user/chat/ensure-users` | JWT | Ensure users exist on chat platform |
| POST | `/user/chat/message-push` | JWT | Dispatch FCM + in-app chat notification |
| GET/PUT | `/user/chat/favourites` | JWT | Legacy Mongo favourites (optional) |

## Config (`wyton-api/config/*.json`)

```json
"CHAT_BASE_URL": "http://localhost:8080",
"CHAT_APP_ID": "wyton_dev",
"CHAT_APP_SECRET": "..."
```

Do **not** put `CHAT_APP_SECRET` in the browser. Web obtains `baseUrl`/`appId`/`token` from `/user/chat/session`.

## Web files

- `wyton-web/src/services/chatConnection.ts` — connect/disconnect via session
- `wyton-web/src/services/wytonChatClient.ts` — HTTP + WebSocket client
- `wyton-web/src/services/chat.ts` — channel/message helpers + push notify
- `wyton-web/src/pages/ChatRoom/ChatRoom.tsx` — UI

## FCM

Same as before: `POST /user/setup-fcm-token`, Firebase Admin SDK on API, `CHAT_MESSAGE` notification type with `channelUrl` deep link.
