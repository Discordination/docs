# Discordination Documentation

Project documentation — deployed to GitHub Pages

## API Reference

Discordination replicates the [Revolt](https://revolt.chat) API protocol so the existing frontend works without modification.

- [REST API Reference](api-reference/rest-api.md) — HTTP endpoints for auth, users, servers, channels, messages, files
- [WebSocket Events](api-reference/websocket-events.md) — Real-time event protocol (client ↔ server)
- [Database Schema](api-reference/database-schema.md) — Data model and entity relationships

## Architecture

- **Gateway** (Elixir/Phoenix) — WebSocket server handling real-time events
- **API** (Go) — REST API server
- **Services** (Go) — File server, link proxy, GIF proxy
- **Web** (React/TypeScript) — Frontend client (forked from Revolt frontend)

## Infrastructure

- AWS EC2 (ca-central-1) with Docker Compose
- Caddy for TLS termination and reverse proxy
- PostgreSQL + Redis for data storage
- LiveKit for voice/video
- S3 for media storage
