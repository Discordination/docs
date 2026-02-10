# Revolt REST API Reference

> Discordination replicates the Revolt REST API so the existing frontend works without modification.
> Base URL: `https://api.revolt.chat` (Revolt) / `https://api.discordination.com` (ours)

All authenticated endpoints require the `x-session-token` header.

---

## Authentication

| Method | Path | Description |
|--------|------|-------------|
| POST | `/auth/session/login` | Log in with email + password. Returns `{ _id, user_id, token, name }`. May return MFA challenge. |
| DELETE | `/auth/session/logout` | Revoke the current session. |
| GET | `/auth/session/check` | Verify current session is still valid. |
| GET | `/auth/session/all` | List all active sessions for the user. |
| DELETE | `/auth/session/all` | Revoke all sessions except the current one. |
| DELETE | `/auth/session/:id` | Revoke a specific session by ID. |
| POST | `/auth/account/create` | Create account with `{ email, password, invite?, captcha? }`. |
| POST | `/auth/account/reverify` | Re-send verification email. |
| POST | `/auth/account/verify/:code` | Verify email address. |
| POST | `/auth/account/reset_password` | Request password reset email. |
| PATCH | `/auth/account/reset_password` | Complete password reset with token. |
| PATCH | `/auth/account/change/password` | Change password (requires current password). |
| PATCH | `/auth/account/change/email` | Change email (requires current password). |

### Login Request
```json
{
  "email": "user@example.com",
  "password": "...",
  "friendly_name": "My Browser"
}
```

### Login Response
```json
{
  "_id": "01ABCDEF...",
  "user_id": "01ABCDEF...",
  "token": "session-token-here",
  "name": "My Browser"
}
```

---

## Users

| Method | Path | Description |
|--------|------|-------------|
| GET | `/users/@me` | Fetch the authenticated user's full profile. |
| GET | `/users/:id` | Fetch a user by ID. Returns `User` object. |
| PATCH | `/users/@me` | Edit own profile: `{ display_name?, avatar?, status?, profile?, badges?, flags? }`. |
| GET | `/users/:id/profile` | Fetch a user's profile (bio + background). |
| GET | `/users/@me/relationships` | List all relationships (friends, blocked, pending). |
| PUT | `/users/:id/friend` | Send or accept a friend request. |
| DELETE | `/users/:id/friend` | Remove friend or deny request. |
| PUT | `/users/:id/block` | Block a user. |
| DELETE | `/users/:id/block` | Unblock a user. |
| GET | `/users/:id/mutual` | Get mutual friends and servers with a user. |
| GET | `/users/:id/dm` | Open or get existing DM channel with a user. |
| GET | `/users/:id/default_avatar` | Get default avatar for a user. |
| GET | `/users/:id/flags` | Get user flags. |

### User Object
```json
{
  "_id": "01ABCDEF...",
  "username": "alice",
  "display_name": "Alice",
  "avatar": { "_id": "...", "tag": "avatars", "filename": "avatar.png", ... },
  "status": { "text": "Hello!", "presence": "Online" },
  "online": true,
  "flags": 0,
  "badges": 0,
  "relations": [{ "_id": "01...", "status": "Friend" }],
  "bot": null
}
```

---

## Servers

| Method | Path | Description |
|--------|------|-------------|
| POST | `/servers/create` | Create server: `{ name, description?, icon?, banner?, nsfw? }`. |
| GET | `/servers/:id` | Fetch server by ID. |
| PATCH | `/servers/:id` | Edit server properties. |
| DELETE | `/servers/:id` | Delete the server (owner only). |
| PUT | `/servers/:id/ack` | Mark all channels in server as read. |
| GET | `/servers/:id/members` | List all members. Returns `{ members: Member[], users: User[] }`. |
| GET | `/servers/:id/members/:member_id` | Fetch a specific member. |
| PATCH | `/servers/:id/members/:member_id` | Edit member (nickname, roles, timeout). |
| DELETE | `/servers/:id/members/:member_id` | Kick a member. |
| GET | `/servers/:id/bans` | List server bans. |
| PUT | `/servers/:id/bans/:user_id` | Ban a user: `{ reason? }`. |
| DELETE | `/servers/:id/bans/:user_id` | Unban a user. |
| POST | `/servers/:id/channels` | Create channel in server: `{ type, name, description?, nsfw? }`. |
| POST | `/servers/:id/roles` | Create role: `{ name, rank? }`. |
| PATCH | `/servers/:id/roles/:role_id` | Edit role (name, colour, hoist, rank). |
| DELETE | `/servers/:id/roles/:role_id` | Delete a role. |
| PUT | `/servers/:id/permissions/:role_id` | Set role permissions: `{ permissions: { a, d } }`. |
| PUT | `/servers/:id/permissions/default` | Set default permissions: `{ permissions: { a, d } }`. |

### Server Object
```json
{
  "_id": "01ABCDEF...",
  "owner": "01ABCDEF...",
  "name": "My Server",
  "description": "A cool server",
  "categories": [{ "id": "cat1", "title": "General", "channels": ["01..."] }],
  "system_messages": { "user_joined": "01...", "user_left": "01..." },
  "roles": { "role1": { "name": "Mod", "permissions": { "a": 0, "d": 0 }, "colour": "#ff0000" } },
  "default_permissions": 3
}
```

---

## Channels

| Method | Path | Description |
|--------|------|-------------|
| GET | `/channels/:id` | Fetch channel by ID. |
| PATCH | `/channels/:id` | Edit channel (name, description, icon, nsfw). |
| DELETE | `/channels/:id` | Delete/close channel. For DMs, this hides the conversation. |
| POST | `/channels/create` | Create group DM: `{ name, users, nonce? }`. |
| PUT | `/channels/:id/recipients/:user_id` | Add user to group DM. |
| DELETE | `/channels/:id/recipients/:user_id` | Remove user from group DM. |
| PUT | `/channels/:id/permissions/:role_id` | Set role permissions for channel. |
| PUT | `/channels/:id/permissions/default` | Set default permissions for channel. |

### Channel Object
```json
{
  "_id": "01ABCDEF...",
  "channel_type": "TextChannel",
  "server": "01ABCDEF...",
  "name": "general",
  "description": "Main chat channel",
  "last_message_id": "01ABCDEF...",
  "nsfw": false
}
```

**Channel Types:** `TextChannel`, `VoiceChannel`, `Group`, `DirectMessage`, `SavedMessages`

---

## Messages

| Method | Path | Description |
|--------|------|-------------|
| GET | `/channels/:id/messages` | Fetch messages. Query params: `limit`, `before`, `after`, `sort`, `nearby`, `include_users`. |
| POST | `/channels/:id/messages` | Send message: `{ content?, attachments?, replies?, embeds?, masquerade?, interactions? }`. Header: `Idempotency-Key`. |
| GET | `/channels/:id/messages/:msg_id` | Fetch single message. |
| PATCH | `/channels/:id/messages/:msg_id` | Edit message: `{ content?, embeds? }`. |
| DELETE | `/channels/:id/messages/:msg_id` | Delete a message. |
| POST | `/channels/:id/messages/bulk_delete` | Bulk delete: `{ ids: string[] }`. Max 100. |
| POST | `/channels/:id/messages/search` | Search messages: `{ query, limit?, before?, after?, sort?, include_users? }`. |
| PUT | `/channels/:id/messages/:msg_id/reactions/:emoji` | Add reaction. |
| DELETE | `/channels/:id/messages/:msg_id/reactions/:emoji` | Remove own reaction. |
| DELETE | `/channels/:id/messages/:msg_id/reactions` | Remove all reactions (permission required). |
| PUT | `/channels/:id/ack/:msg_id` | Acknowledge (mark as read) up to this message. |
| POST | `/channels/:id/typing` | Send typing indicator (valid for ~3 seconds). |

### Message Object
```json
{
  "_id": "01ABCDEF...",
  "channel": "01ABCDEF...",
  "author": "01ABCDEF...",
  "content": "Hello, world!",
  "attachments": [],
  "embeds": [],
  "replies": ["01ABCDEF..."],
  "edited": "2025-01-01T00:00:00Z",
  "mentions": ["01ABCDEF..."],
  "reactions": { "emoji_id": ["user_id_1", "user_id_2"] }
}
```

---

## Invites

| Method | Path | Description |
|--------|------|-------------|
| POST | `/servers/:id/invites/:channel_id` | Create invite for a channel. Returns `{ _id, type, server, channel, creator }`. |
| GET | `/invites/:code` | Fetch invite info (public, no auth required). |
| POST | `/invites/:code` | Join server via invite code. |
| DELETE | `/invites/:code` | Delete an invite. |

---

## File Uploads

| Method | Path | Description |
|--------|------|-------------|
| POST | `/autumn/attachments` | Upload file. Multipart form: `file` field. Returns `{ id }` to use in message attachments. |
| POST | `/autumn/avatars` | Upload avatar image. |
| POST | `/autumn/backgrounds` | Upload profile background. |
| POST | `/autumn/icons` | Upload server/channel icon. |
| POST | `/autumn/banners` | Upload server/group banner. |
| POST | `/autumn/emojis` | Upload custom emoji. |

**Response:**
```json
{
  "id": "01ABCDEF..."
}
```

Use the returned `id` in the `attachments` array when sending a message.

---

## Push Notifications

| Method | Path | Description |
|--------|------|-------------|
| POST | `/push/subscribe` | Subscribe to push notifications: `{ endpoint, p256dh, auth }` (Web Push). |
| POST | `/push/unsubscribe` | Unsubscribe from push notifications. |

---

## Safety / Moderation

| Method | Path | Description |
|--------|------|-------------|
| POST | `/safety/report` | Report content: `{ content: { type, id, report_reason, additional_context? } }`. |

---

## Custom Emojis

| Method | Path | Description |
|--------|------|-------------|
| GET | `/custom/emoji/:id` | Fetch emoji by ID. |
| PUT | `/custom/emoji/:id` | Create emoji: `{ name, parent: { type: "Server", id }, nsfw? }`. |
| DELETE | `/custom/emoji/:id` | Delete custom emoji. |

---

## Platform / Misc

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | API root. Returns `{ revolt: "version", features: {...} }`. |
| GET | `/onboard/hello` | Check if user needs onboarding. Returns `{ onboarding: bool }`. |
| POST | `/onboard/complete` | Complete onboarding: `{ username }`. |

---

## Error Format

All error responses follow this shape:
```json
{
  "type": "ErrorId",
  "location": "field_name"
}
```

Common error types: `InvalidSession`, `NotFound`, `Unauthorized`, `PermissionDenied`, `TooManyRequests`, `InternalError`.

## Rate Limits

Rate limits are returned via headers:
- `X-RateLimit-Limit` — max requests per window
- `X-RateLimit-Remaining` — requests remaining
- `X-RateLimit-Reset-After` — seconds until window resets

When rate-limited, the API returns HTTP 429 with a `retry_after` field in milliseconds.
