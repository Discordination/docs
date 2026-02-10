# Database Schema Reference

> Revolt uses MongoDB. Discordination uses PostgreSQL with the same logical schema.
> This document maps the Revolt data model so we maintain API compatibility.

---

## Collections Overview

| Collection | Description | Primary Key |
|-----------|-------------|-------------|
| `users` | User accounts | `_id` (Snowflake) |
| `sessions` | Auth sessions | `_id` (Snowflake) |
| `servers` | Community servers | `_id` (Snowflake) |
| `channels` | Text, voice, DM, group channels | `_id` (Snowflake) |
| `messages` | Chat messages | `_id` (Snowflake) |
| `members` | Server membership | `_id` (compound: server + user) |
| `invites` | Server invite codes | `_id` (string, invite code) |
| `attachments` | Uploaded files | `_id` (Snowflake) |
| `emojis` | Custom emojis | `_id` (Snowflake) |
| `bots` | Bot accounts | `_id` (Snowflake) |
| `user_settings` | Per-user settings | `_id` (user Snowflake) |
| `channel_unreads` | Read position tracking | compound (channel + user) |

---

## users

```
{
  _id:           Snowflake        -- Unique user ID
  username:      string           -- Unique username
  display_name:  string?          -- Optional display name
  avatar:        Attachment?      -- Avatar attachment object
  status: {
    text:        string?          -- Custom status text
    presence:    string?          -- Online | Idle | Focus | Busy | Invisible
  }
  profile: {
    content:     string?          -- Bio / about me (markdown)
    background:  Attachment?      -- Profile background image
  }
  relations: [{                   -- Relationships with other users
    _id:         Snowflake        -- Other user's ID
    status:      string           -- Friend | Outgoing | Incoming | Blocked | BlockedOther | None
  }]
  badges:        int              -- Badge bitfield
  flags:         int              -- User flags bitfield (0=normal, 1=suspended, 2=deleted, 4=banned)
  bot: {                          -- Only present for bot accounts
    owner:       Snowflake        -- Bot owner's user ID
  }
  password:      string           -- Hashed password (never exposed via API)
  email:         string           -- Email address (never exposed via API)
}
```

**Indexes:**
- `username` — unique
- `email` — unique (internal only)

**Flags bitfield:**
- `1` — Suspended
- `2` — Deleted
- `4` — Banned

---

## sessions

```
{
  _id:           Snowflake        -- Session ID
  user_id:       Snowflake        -- Owning user ID
  token:         string           -- Session token (used in x-session-token header)
  name:          string           -- Friendly name (e.g. "Chrome on Windows")
  subscription:  {                -- Web push subscription (optional)
    endpoint:    string
    p256dh:      string
    auth:        string
  }
}
```

**Indexes:**
- `user_id`
- `token` — unique

---

## servers

```
{
  _id:                Snowflake        -- Server ID
  owner:              Snowflake        -- Owner user ID
  name:               string           -- Server name
  description:        string?          -- Server description
  icon:               Attachment?      -- Server icon
  banner:             Attachment?      -- Server banner
  categories: [{                       -- Channel categories
    id:               string           -- Category ID
    title:            string           -- Category name
    channels:         [Snowflake]      -- Ordered list of channel IDs
  }]
  system_messages: {                   -- Channels for system messages
    user_joined:      Snowflake?
    user_left:        Snowflake?
    user_kicked:      Snowflake?
    user_banned:      Snowflake?
  }
  roles: {                             -- Map of role_id → Role
    "<role_id>": {
      name:           string
      permissions: {
        a:            uint64           -- Allow bitfield
        d:            uint64           -- Deny bitfield
      }
      colour:         string?          -- CSS colour string
      hoist:          bool             -- Show separately in member list
      rank:           int              -- Ordering rank (lower = higher priority)
    }
  }
  default_permissions: uint64          -- Default permission bitfield for @everyone
  nsfw:               bool
  flags:              int              -- Server flags
  analytics:          bool             -- Analytics enabled
  discoverable:       bool             -- Listed in server discovery
}
```

**Indexes:**
- `owner`

---

## channels

```
{
  _id:                Snowflake        -- Channel ID
  channel_type:       string           -- TextChannel | VoiceChannel | Group | DirectMessage | SavedMessages
  server:             Snowflake?       -- Server ID (null for DMs/groups)
  name:               string?          -- Channel name (server channels, groups)
  description:        string?          -- Channel topic/description
  icon:               Attachment?      -- Channel icon (groups)
  recipients:         [Snowflake]?     -- User IDs (DMs, groups only)
  default_permissions: Permission?     -- Override default permissions
  role_permissions: {                  -- Map of role_id → Permission
    "<role_id>": { a: uint64, d: uint64 }
  }
  nsfw:               bool
  last_message_id:    Snowflake?       -- Most recent message ID
  active:             bool             -- Whether DM is active/visible
  owner:              Snowflake?       -- Group DM owner
}
```

**Channel Types:**
- `SavedMessages` — Personal notes channel (one per user, `recipients: [user_id]`)
- `DirectMessage` — 1:1 DM (`recipients: [user_a, user_b]`)
- `Group` — Group DM (`recipients: [user_a, user_b, ...]`, has `owner`)
- `TextChannel` — Server text channel (has `server`)
- `VoiceChannel` — Server voice channel (has `server`)

**Indexes:**
- `server`
- `recipients`
- `last_message_id`

---

## messages

```
{
  _id:            Snowflake              -- Message ID
  channel:        Snowflake              -- Channel ID
  author:         Snowflake              -- Author user ID
  content:        string?                -- Message text content
  attachments:    [Attachment]?          -- Uploaded file attachments
  embeds:         [Embed]?               -- Rich embeds (links, custom)
  replies:        [Snowflake]?           -- IDs of messages being replied to
  edited:         datetime?              -- Last edit timestamp
  mentions:       [Snowflake]?           -- Mentioned user IDs
  reactions: {                           -- Map of emoji → [user IDs]
    "<emoji_id>": [Snowflake]
  }
  interactions: {                        -- Bot interaction config
    reactions:    [string]?              -- Emoji reactions to add
    restrict_reactions: bool?            -- Only allow listed reactions
  }
  masquerade: {                          -- Override display name/avatar
    name:         string?
    avatar:       string?                -- URL
    colour:       string?
  }
  nonce:          string?                -- Client-provided dedup nonce
}
```

**Indexes:**
- `channel` + `_id` (compound, primary query pattern)
- `channel` + `author`
- `content` (text index for search)

---

## members

```
{
  _id: {
    server:       Snowflake              -- Server ID
    user:         Snowflake              -- User ID
  }
  nickname:       string?                -- Server-specific nickname
  avatar:         Attachment?            -- Server-specific avatar
  roles:          [string]               -- Assigned role IDs
  timeout:        datetime?              -- Timeout expiry (user is muted until)
  joined_at:      datetime               -- When user joined the server
}
```

**Indexes:**
- `_id.server`
- `_id.user`
- `_id.server` + `_id.user` (compound, unique)

---

## invites

```
{
  _id:            string                 -- Invite code (e.g. "abc123")
  type:           string                 -- "Server"
  server:         Snowflake              -- Server ID
  channel:        Snowflake              -- Target channel ID
  creator:        Snowflake              -- Creator user ID
}
```

**Indexes:**
- `server`
- `creator`

---

## attachments

```
{
  _id:            Snowflake              -- Attachment ID
  tag:            string                 -- Upload tag: avatars | backgrounds | icons | banners | attachments | emojis
  filename:       string                 -- Original filename
  metadata: {
    type:         string                 -- File | Text | Image | Video | Audio
    width:        int?                   -- Image/video width
    height:       int?                   -- Image/video height
  }
  content_type:   string                 -- MIME type
  size:           int64                  -- File size in bytes
  deleted:        bool?                  -- Soft-deleted
  reported:       bool?                  -- Flagged for review
  message_id:     Snowflake?             -- Associated message
  user_id:        Snowflake?             -- Uploader
  server_id:      Snowflake?             -- Associated server
  object_id:      Snowflake?             -- Associated object
}
```

**Indexes:**
- `tag`
- `message_id`
- `user_id`

---

## emojis

```
{
  _id:            Snowflake              -- Emoji ID
  parent: {
    type:         string                 -- "Server" | "Detached"
    id:           Snowflake?             -- Server ID (if type is Server)
  }
  creator_id:     Snowflake              -- Creator user ID
  name:           string                 -- Emoji name (used as :name:)
  animated:       bool                   -- Animated GIF emoji
  nsfw:           bool                   -- NSFW emoji
}
```

**Indexes:**
- `parent.id` (server emojis)
- `creator_id`

---

## bots

```
{
  _id:                  Snowflake        -- Bot user ID (same as users._id)
  owner:                Snowflake        -- Owner user ID
  token:                string           -- Bot token
  public:               bool             -- Can be added by anyone
  analytics:            bool             -- Analytics enabled
  discoverable:         bool             -- Listed in bot directory
  interactions_url:     string?          -- Webhook URL for interactions
  terms_of_service_url: string?
  privacy_policy_url:   string?
  flags:                int
}
```

---

## channel_unreads

```
{
  _id: {
    channel:      Snowflake              -- Channel ID
    user:         Snowflake              -- User ID
  }
  last_id:        Snowflake?             -- Last read message ID
  mentions:       [Snowflake]?           -- Unread mention message IDs
}
```

**Indexes:**
- `_id.user` (fetch all unreads for a user)

---

## Entity Relationships

```
users ─────┬──< sessions          (user has many sessions)
           ├──< members           (user belongs to many servers via members)
           ├──< messages          (user authors many messages)
           ├──< bots              (user owns many bots)
           └──< channel_unreads   (user has read positions)

servers ───┬──< channels          (server has many channels)
           ├──< members           (server has many members)
           ├──< invites           (server has many invites)
           └──< emojis            (server has many custom emojis)

channels ──┬──< messages          (channel contains many messages)
           └──< channel_unreads   (channel has read positions)

messages ──┴──< attachments       (message has many attachments)
```

---

## PostgreSQL Adaptation Notes

When translating this MongoDB schema to PostgreSQL for Discordination:

1. **Snowflake IDs** → `BIGINT` or `TEXT` primary keys
2. **Embedded documents** (e.g., `status`, `profile`) → either JSONB columns or separate tables
3. **Compound `_id`** (members, channel_unreads) → composite primary keys
4. **`roles` map on servers** → separate `server_roles` table with `server_id` FK
5. **`role_permissions` map on channels** → separate `channel_role_permissions` table
6. **`reactions` map on messages** → separate `message_reactions` table
7. **`relations` array on users** → separate `user_relationships` table
8. **Array fields** (recipients, mentions, replies) → junction tables or `BIGINT[]` arrays
9. **Text search** → PostgreSQL full-text search with `tsvector`/`tsquery`
10. **Timestamps** → `TIMESTAMPTZ` columns
