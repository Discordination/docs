# Revolt WebSocket Events Reference

> Discordination replicates the Revolt WebSocket protocol so the existing frontend connects without modification.
> WebSocket URL: `wss://ws.revolt.chat` (Revolt) / `wss://ws.discordination.com` (ours)

The WebSocket uses JSON-encoded messages. Each message has a `type` field that identifies the event.

---

## Connection Flow

1. Client connects to the WebSocket endpoint
2. Client sends `Authenticate` with session token
3. Server responds with `Authenticated`
4. Server sends `Ready` with initial state
5. Client maintains connection with `Ping`/`Pong` heartbeats

---

## Client → Server Events

### Authenticate
First message after connecting. Provides the session token.
```json
{
  "type": "Authenticate",
  "token": "session-token-here"
}
```

### BeginTyping
Start showing typing indicator in a channel.
```json
{
  "type": "BeginTyping",
  "channel": "01ABCDEF..."
}
```

### EndTyping
Stop showing typing indicator.
```json
{
  "type": "EndTyping",
  "channel": "01ABCDEF..."
}
```

### Ping
Heartbeat keepalive. Server responds with `Pong`. Send every 10-30 seconds.
```json
{
  "type": "Ping",
  "data": 12345,
  "responded": null
}
```

---

## Server → Client Events

### Authenticated
Confirms successful authentication.
```json
{
  "type": "Authenticated"
}
```

### Ready
Sent after authentication. Contains the full initial state.
```json
{
  "type": "Ready",
  "users": [User, ...],
  "servers": [Server, ...],
  "channels": [Channel, ...],
  "members": [Member, ...],
  "emojis": [Emoji, ...]
}
```

### Pong
Response to client `Ping`.
```json
{
  "type": "Pong",
  "data": 12345
}
```

---

## Message Events

### Message
A new message was sent.
```json
{
  "type": "Message",
  "_id": "01ABCDEF...",
  "channel": "01ABCDEF...",
  "author": "01ABCDEF...",
  "content": "Hello!",
  "attachments": [],
  "embeds": [],
  "replies": [],
  "mentions": [],
  "reactions": {},
  "masquerade": null
}
```

### MessageUpdate
A message was edited.
```json
{
  "type": "MessageUpdate",
  "id": "01ABCDEF...",
  "channel": "01ABCDEF...",
  "data": {
    "content": "Updated content",
    "edited": "2025-01-01T00:00:00Z"
  }
}
```

### MessageDelete
A message was deleted.
```json
{
  "type": "MessageDelete",
  "id": "01ABCDEF...",
  "channel": "01ABCDEF..."
}
```

### MessageAppend
An embed was appended to a message (e.g., link preview resolved).
```json
{
  "type": "MessageAppend",
  "id": "01ABCDEF...",
  "channel": "01ABCDEF...",
  "append": {
    "embeds": [{ "type": "Website", "url": "...", "title": "..." }]
  }
}
```

### MessageReact
A reaction was added to a message.
```json
{
  "type": "MessageReact",
  "id": "01ABCDEF...",
  "channel_id": "01ABCDEF...",
  "user_id": "01ABCDEF...",
  "emoji_id": "01ABCDEF..."
}
```

### MessageUnreact
A reaction was removed from a message.
```json
{
  "type": "MessageUnreact",
  "id": "01ABCDEF...",
  "channel_id": "01ABCDEF...",
  "user_id": "01ABCDEF...",
  "emoji_id": "01ABCDEF..."
}
```

---

## Channel Events

### ChannelCreate
A new channel was created.
```json
{
  "type": "ChannelCreate",
  "_id": "01ABCDEF...",
  "channel_type": "TextChannel",
  "server": "01ABCDEF...",
  "name": "new-channel"
}
```

### ChannelUpdate
A channel was updated.
```json
{
  "type": "ChannelUpdate",
  "id": "01ABCDEF...",
  "data": {
    "name": "renamed-channel",
    "description": "New description"
  },
  "clear": ["Icon"]
}
```
The `clear` array lists fields that were removed (set to null).

### ChannelDelete
A channel was deleted.
```json
{
  "type": "ChannelDelete",
  "id": "01ABCDEF..."
}
```

### ChannelGroupJoin
A user joined a group DM.
```json
{
  "type": "ChannelGroupJoin",
  "id": "01ABCDEF...",
  "user": "01ABCDEF..."
}
```

### ChannelGroupLeave
A user left a group DM.
```json
{
  "type": "ChannelGroupLeave",
  "id": "01ABCDEF...",
  "user": "01ABCDEF..."
}
```

### ChannelStartTyping
A user started typing in a channel.
```json
{
  "type": "ChannelStartTyping",
  "id": "01ABCDEF...",
  "user": "01ABCDEF..."
}
```

### ChannelStopTyping
A user stopped typing in a channel.
```json
{
  "type": "ChannelStopTyping",
  "id": "01ABCDEF...",
  "user": "01ABCDEF..."
}
```

### ChannelAck
A channel was acknowledged (marked as read).
```json
{
  "type": "ChannelAck",
  "id": "01ABCDEF...",
  "user": "01ABCDEF...",
  "message_id": "01ABCDEF..."
}
```

---

## Server Events

### ServerCreate
Sent when the user joins a server or a new server is created.
```json
{
  "type": "ServerCreate",
  "_id": "01ABCDEF...",
  "server": { ...Server },
  "channels": [Channel, ...]
}
```

### ServerUpdate
A server was updated.
```json
{
  "type": "ServerUpdate",
  "id": "01ABCDEF...",
  "data": {
    "name": "New Name",
    "description": "Updated description"
  },
  "clear": []
}
```

### ServerDelete
A server was deleted, or the user left/was removed.
```json
{
  "type": "ServerDelete",
  "id": "01ABCDEF..."
}
```

### ServerMemberJoin
A new member joined the server.
```json
{
  "type": "ServerMemberJoin",
  "id": "01ABCDEF...",
  "user": "01ABCDEF..."
}
```

### ServerMemberUpdate
A member's properties were updated (nickname, roles, avatar).
```json
{
  "type": "ServerMemberUpdate",
  "id": {
    "server": "01ABCDEF...",
    "user": "01ABCDEF..."
  },
  "data": {
    "nickname": "New Nick",
    "roles": ["role1", "role2"]
  },
  "clear": []
}
```

### ServerMemberLeave
A member left or was removed from the server.
```json
{
  "type": "ServerMemberLeave",
  "id": "01ABCDEF...",
  "user": "01ABCDEF..."
}
```

### ServerRoleUpdate
A role was created or updated.
```json
{
  "type": "ServerRoleUpdate",
  "id": "01ABCDEF...",
  "role_id": "role1",
  "data": {
    "name": "Moderator",
    "colour": "#00ff00",
    "permissions": { "a": 123, "d": 0 }
  }
}
```

### ServerRoleDelete
A role was deleted.
```json
{
  "type": "ServerRoleDelete",
  "id": "01ABCDEF...",
  "role_id": "role1"
}
```

---

## User Events

### UserUpdate
A user's profile was updated.
```json
{
  "type": "UserUpdate",
  "id": "01ABCDEF...",
  "data": {
    "status": { "text": "Away", "presence": "Idle" },
    "online": true
  },
  "clear": []
}
```

### UserRelationship
A relationship status changed.
```json
{
  "type": "UserRelationship",
  "id": "01ABCDEF...",
  "user": { ...User },
  "status": "Friend"
}
```

**Relationship statuses:** `Friend`, `Outgoing`, `Incoming`, `Blocked`, `BlockedOther`, `None`

### UserPlatformWipe
A user's account was platform-wiped (banned globally).
```json
{
  "type": "UserPlatformWipe",
  "user_id": "01ABCDEF...",
  "flags": 4
}
```

---

## Emoji Events

### EmojiCreate
A custom emoji was created.
```json
{
  "type": "EmojiCreate",
  "_id": "01ABCDEF...",
  "parent": { "type": "Server", "id": "01ABCDEF..." },
  "creator_id": "01ABCDEF...",
  "name": "cool_emoji",
  "animated": false,
  "nsfw": false
}
```

### EmojiDelete
A custom emoji was deleted.
```json
{
  "type": "EmojiDelete",
  "id": "01ABCDEF..."
}
```

---

## Session Events

### Auth
Session invalidation — the user should log out.
```json
{
  "type": "Auth",
  "event_type": "DeleteSession",
  "user_id": "01ABCDEF...",
  "session_id": "01ABCDEF..."
}
```

---

## Bulk Event

### Bulk
A batch of events sent together (optimization for initial load).
```json
{
  "type": "Bulk",
  "v": [
    { "type": "Message", ... },
    { "type": "ChannelUpdate", ... }
  ]
}
```

---

## Error Events

### Error
Sent when an error occurs (e.g., invalid authentication).
```json
{
  "type": "Error",
  "error": "InvalidSession"
}
```

Common errors: `LabelMe`, `InternalError`, `InvalidSession`, `OnboardingNotFinished`, `AlreadyAuthenticated`

---

## Implementation Notes

- Messages are delivered only for channels the user has access to
- `Ready` payload can be large — consider lazy-loading for large servers
- Typing indicators expire after ~3 seconds if not refreshed
- The `clear` array in update events lists fields that should be set to null/removed
- `Ping`/`Pong` should be sent every 10-30 seconds to prevent disconnection
- If the connection drops, reconnect and re-authenticate — the server will send a fresh `Ready`
