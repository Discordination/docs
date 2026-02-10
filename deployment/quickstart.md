# Quickstart: Self-Hosted Deployment

Deploy Discordination on a single server using Docker Compose.

## Prerequisites

- Docker Engine 24+ and Docker Compose v2
- A domain name with DNS pointed to your server
- 2 GB RAM minimum (4 GB recommended)
- 20 GB disk space

## 1. Clone and Configure

```bash
git clone https://github.com/Discordination/infrastructure.git
cd infrastructure/compose
cp .env.example .env
```

Edit `.env` with your configuration:

```env
# Required
DOMAIN=chat.example.com
POSTGRES_PASSWORD=<generate-a-strong-password>
JWT_SECRET=<generate-a-256-bit-secret>

# Optional (for voice)
LIVEKIT_API_KEY=<your-livekit-key>
LIVEKIT_API_SECRET=<your-livekit-secret>
LIVEKIT_WS_URL=wss://livekit.example.com

# Optional (for push notifications)
FCM_CREDENTIALS_JSON=<firebase-service-account-json>
APNS_KEY_ID=<apple-key-id>
APNS_TEAM_ID=<apple-team-id>
APNS_PRIVATE_KEY=<apple-p8-key-contents>
```

## 2. Generate Secrets

```bash
# Generate JWT secret
openssl rand -hex 32

# Generate Postgres password
openssl rand -base64 24
```

## 3. Start Services

```bash
docker compose up -d
```

This starts: PostgreSQL, Redis, API, Gateway, Web, File Server, Link Proxy, GIF Proxy, and Cleanup daemon.

## 4. Run Database Migrations

```bash
docker compose exec api /app/api --migrate
```

## 5. Create First Admin User

```bash
curl -X POST https://chat.example.com/api/auth/account/create \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "email": "admin@example.com", "password": "your-password"}'
```

## 6. Access the Web Client

Open `https://chat.example.com` in your browser and log in.

## 7. Configure TLS (if not using a reverse proxy)

If you're not behind a TLS-terminating proxy (like Cloudflare or nginx), enable Let's Encrypt in the compose file:

```yaml
# In docker-compose.yml, uncomment the certbot service
certbot:
  image: certbot/certbot
  volumes:
    - ./certbot/conf:/etc/letsencrypt
    - ./certbot/www:/var/www/certbot
```

## Updating

```bash
docker compose pull
docker compose up -d
docker compose exec api /app/api --migrate
```

## Stopping

```bash
docker compose down        # Stop services (keep data)
docker compose down -v     # Stop services and delete volumes (DESTRUCTIVE)
```
