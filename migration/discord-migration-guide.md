# Discord Server Migration Guide

This guide walks server administrators through migrating a Discord server to Discordination using the migration bot.

## Prerequisites

- A running Discordination instance (see [Quickstart](../deployment/quickstart.md))
- A Discord bot token with the following permissions:
  - Read Messages / View Channels
  - Read Message History
  - Send Messages (for DM invites)
  - Manage Guild (for member list access)
  - Use Slash Commands
- The Discord bot must be added to your server with the **Server Members Intent** and **Message Content Intent** enabled
- (Optional) An S3-compatible bucket for media/attachment storage

## Overview

The migration process has three stages:

1. **Scrape** — Export your Discord server's structure, roles, members, messages, emoji, and attachments
2. **Import** — Create the equivalent server on Discordination with all data preserved
3. **Bridge** — Run a bidirectional message bridge during the transition period

## Step 1: Configure Environment

Create a `.env` file (or export the variables) with your credentials:

```bash
# Required
export DISCORD_BOT_TOKEN="your-discord-bot-token"
export DISCORD_GUILD_ID="your-discord-server-id"
export DISCORDINATION_API_URL="https://api.your-discordination.com"
export DISCORDINATION_API_TOKEN="your-admin-api-token"

# Optional — S3 for attachments and media
export S3_BUCKET="discordination-uploads"
export S3_REGION="us-east-1"
export S3_ENDPOINT=""  # Leave empty for AWS, set for MinIO

# Optional — WebSocket URL for bridge mode
export DISCORDINATION_WS_URL="wss://ws.your-discordination.com"

# Optional
export DATA_DIR="./data"
export DRY_RUN="false"
```

### Getting Your Discord Guild ID

1. Open Discord Settings → Advanced → Enable **Developer Mode**
2. Right-click your server name → **Copy Server ID**

### Creating a Discord Bot

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications)
2. Click **New Application**, name it (e.g., "Migration Bot")
3. Go to **Bot** → Click **Add Bot**
4. Under **Privileged Gateway Intents**, enable:
   - Server Members Intent
   - Message Content Intent
5. Copy the bot token
6. Go to **OAuth2** → **URL Generator**:
   - Scopes: `bot`, `applications.commands`
   - Permissions: Read Messages, Read Message History, Send Messages, Manage Guild
7. Use the generated URL to invite the bot to your server

## Step 2: Scrape Your Discord Server

Run the scraper to export all server data:

```bash
migrate --scrape --guild YOUR_GUILD_ID
```

Or if using Docker:

```bash
docker run --env-file .env ghcr.io/discordination/migration-bot:latest --scrape
```

This will:
- Export server name, icon, description
- Export all categories and channels (text + voice)
- Export roles with colors, permissions, and hierarchy
- Export the full member list with avatars
- Export complete message history (paginated, rate-limited)
- Export pinned messages per channel
- Export custom emoji
- Upload attachments and media to S3 (if configured)

Output is saved to `./data/server_<guild_id>_<timestamp>.json`.

### Scrape Duration

Scraping respects Discord's rate limits. Approximate times:

| Server Size | Channels | Messages | Estimated Time |
|-------------|----------|----------|----------------|
| Small       | < 20     | < 10K    | 5–15 minutes   |
| Medium      | 20–50    | 10K–100K | 30–90 minutes  |
| Large       | 50+      | 100K+    | 2–8 hours      |

The scraper supports **resumable migrations** — if interrupted, re-run the same command and it will continue where it left off.

## Step 3: Import Into Discordination

Once the scrape is complete, import the data:

```bash
migrate --import --data ./data/server_<guild_id>_<timestamp>.json
```

This will:
1. Create the Discordination server with matching name and description
2. Create all roles (preserving colors and hierarchy)
3. Create all channels (text and voice, with topics and categories)
4. Import all messages with author attribution (`**[Username]** message content`)
5. Generate cryptographic claim tokens for each member
6. Send Discord DMs inviting members to claim their accounts

### Claim Flow

Each non-bot member receives a DM like:

> Hey **Username**! Your community **Server Name** has migrated to **Discordination**. Claim your account: https://api.your-discordination.com/claim/abc123...
>
> This link expires in 30 days.

When a user clicks their claim link, they create a Discordination account and all their migrated messages are re-attributed to their new account.

Claim tokens are saved to `./data/tokens_<guild_id>_<timestamp>.json` for reference.

## Step 4: Run the Bridge (Optional)

During the transition period, you can run a bidirectional message bridge so conversations continue across both platforms:

```bash
migrate --bridge
```

This starts the bot in bridge mode with admin slash commands. Then in Discord:

1. **Link channels**: Use `/migrate-bridge discordination-channel-id:CHANNEL_ID` in each Discord channel you want to bridge
2. **Check status**: `/migrate-status`
3. **Pause/resume**: `/migrate-pause` and `/migrate-resume`
4. **Remove bridge**: `/migrate-unbridge`

Messages sent in a bridged Discord channel appear in the linked Discordination channel (and vice versa), prefixed with the author's name.

### Bridge Behavior

- Messages from Discord users appear in Discordination as `**[DiscordUsername]** message`
- Messages from Discordination users appear in Discord as `**[Username]** message`
- Bridged messages are detected by the `**[name]**` prefix and are not re-bridged (prevents loops)
- The bridge polls the Discordination API every 3 seconds for new messages

## Step 5: Complete the Migration

Once most members have claimed their accounts and activity has shifted to Discordination:

1. Run `/migrate-pause` to stop the bridge
2. Post a final announcement in Discord directing remaining users to Discordination
3. Shut down the migration bot
4. (Optional) Archive or delete the Discord server

## Troubleshooting

### "DISCORD_BOT_TOKEN is required"
Ensure the `DISCORD_BOT_TOKEN` environment variable is set and the token is valid.

### "API returned 403"
Check that your `DISCORDINATION_API_TOKEN` has admin permissions on the target Discordination instance.

### Scrape seems stuck
The scraper respects Discord rate limits (typically 50 requests per second with bucket-based throttling). Large channels with many messages will take time. Check the JSON log output for progress updates.

### DMs not being sent
Discord limits bot DMs. If the bot hasn't interacted with a user before, Discord may block the DM. The migration bot logs failed DMs and still generates claim tokens — you can share claim URLs manually.

### Messages appear out of order
The importer processes messages in chronological order. If you see ordering issues, it may be due to API rate limiting during import. The message timestamps from Discord are preserved in the content.

### S3 upload failures
Ensure your S3 credentials and bucket configuration are correct. The scraper will continue without S3 — attachments won't be uploaded but message text is still preserved.

## Data Privacy

- The scraper only accesses data the bot has permission to read
- Scraped data is stored locally in JSON format — review it before importing
- Claim tokens are cryptographically random (32 bytes, hex-encoded) and expire after 30 days
- No data is sent to third parties — everything goes directly from Discord to your Discordination instance
- Members who don't claim their accounts within the expiry period will have their migrated messages attributed to a placeholder account
