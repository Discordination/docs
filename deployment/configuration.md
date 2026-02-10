# Configuration Reference

All Discordination services are configured via environment variables.

## API Server

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | No | `8080` | HTTP listen port |
| `DATABASE_URL` | Yes | — | PostgreSQL connection string |
| `REDIS_URL` | Yes | — | Redis connection string |
| `JWT_SECRET` | Yes | — | Secret for signing session tokens |
| `JWT_EXPIRY` | No | `7d` | Session token expiry duration |
| `CORS_ORIGINS` | No | `*` | Comma-separated allowed origins |
| `MAX_MESSAGE_LENGTH` | No | `4000` | Maximum message content length |
| `LOG_LEVEL` | No | `info` | Log level: debug, info, warn, error |

## Gateway

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | No | `4000` | HTTP/WebSocket listen port |
| `SECRET_KEY_BASE` | Yes | — | Phoenix secret key base |
| `DATABASE_URL` | No | — | PostgreSQL URL (for Ecto queries) |
| `REDIS_URL` | Yes | — | Redis URL for pub/sub and events |
| `HEARTBEAT_INTERVAL` | No | `45000` | Client heartbeat interval (ms) |
| `HEARTBEAT_TIMEOUT` | No | `60000` | Heartbeat timeout (ms) |

### LiveKit Voice (Gateway)

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `LIVEKIT_API_KEY` | For voice | — | LiveKit API key |
| `LIVEKIT_API_SECRET` | For voice | — | LiveKit API secret |
| `LIVEKIT_WS_URL` | For voice | — | LiveKit WebSocket URL |
| `LIVEKIT_TOKEN_TTL` | No | `3600` | Token validity (seconds) |

### Push Notifications (Gateway)

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `FCM_PROJECT_ID` | For Android | — | Firebase project ID |
| `FCM_CREDENTIALS_JSON` | For Android | — | Service account JSON |
| `FCM_CREDENTIALS_FILE` | For Android | — | Path to credentials file |
| `APNS_KEY_ID` | For iOS | — | APNs key ID |
| `APNS_TEAM_ID` | For iOS | — | Apple team ID |
| `APNS_PRIVATE_KEY` | For iOS | — | P8 private key contents |
| `APNS_BUNDLE_ID` | For iOS | — | App bundle ID |
| `APNS_SANDBOX` | No | `false` | Use sandbox endpoint |

## File Server

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | No | `8090` | HTTP listen port |
| `S3_BUCKET` | Yes | — | S3 bucket name |
| `S3_REGION` | Yes | — | AWS region |
| `S3_ENDPOINT` | No | — | Custom endpoint (for MinIO) |
| `S3_ACCESS_KEY` | No | — | Access key (uses IAM role if empty) |
| `S3_SECRET_KEY` | No | — | Secret key |
| `MAX_FILE_SIZE` | No | `20971520` | Max upload size (bytes, default 20MB) |
| `THUMB_SIZES` | No | `128,256,512` | Thumbnail sizes to generate |

## Link Proxy

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | No | `8091` | HTTP listen port |
| `USER_AGENT` | No | `Discordination/1.0` | User-Agent for outgoing requests |
| `REQUEST_TIMEOUT` | No | `10s` | Timeout for URL fetches |
| `MAX_BODY_SIZE` | No | `5242880` | Max response body (5MB) |

## GIF Proxy

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | No | `8092` | HTTP listen port |
| `TENOR_API_KEY` | Yes | — | Tenor API v2 key |
| `REDIS_URL` | Yes | — | Redis URL for caching |
| `SEARCH_CACHE_TTL` | No | `5m` | Search result cache TTL |
| `TRENDING_CACHE_TTL` | No | `10m` | Trending results cache TTL |

## Cleanup Daemon

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `S3_BUCKET` | Yes | — | S3 bucket to clean |
| `S3_REGION` | Yes | — | AWS region |
| `DATABASE_URL` | Yes | — | PostgreSQL URL (for reference checking) |
| `RETENTION_DAYS` | No | `30` | Days before orphan deletion |
| `RUN_INTERVAL` | No | `24h` | Interval between cleanup runs |
| `DRY_RUN` | No | `false` | Log but don't delete |

## Web Client

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `VITE_API_URL` | No | `/api` | API base URL |
| `VITE_WS_URL` | No | `/ws` | WebSocket gateway URL |
| `VITE_APP_NAME` | No | `Discordination` | Application display name |
