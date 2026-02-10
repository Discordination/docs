# Architecture Overview

Discordination is a self-hosted Discord alternative built as a set of microservices.

## Services

| Service | Language | Description | Default Port |
|---------|----------|-------------|-------------|
| **API** | Go | REST API server (Revolt-compatible) | 8080 |
| **Gateway** | Elixir | WebSocket gateway for real-time events | 4000 |
| **Web** | React/TS | Web client (served via nginx) | 3000 |
| **Desktop** | Tauri | Native desktop wrapper | — |
| **File Server** | Go | S3-backed file uploads and thumbnailing | 8090 |
| **Link Proxy** | Go | OG metadata + image proxy (SSRF-safe) | 8091 |
| **GIF Proxy** | Go | Tenor API proxy with Redis cache | 8092 |
| **Cleanup** | Go | Scheduled S3 orphan removal daemon | — |

## External Dependencies

| Dependency | Purpose | Required |
|------------|---------|----------|
| **PostgreSQL** | Primary database (RDS Aurora recommended) | Yes |
| **Redis** | Caching, rate limiting, pub/sub, sessions | Yes |
| **S3** (or MinIO) | File storage | Yes |
| **LiveKit** | Voice/video WebRTC SFU | For voice |
| **CloudFront** | CDN for web client and files | Recommended |

## Network Topology

```
                    ┌────────────┐
                    │ CloudFront │
                    └──────┬─────┘
                           │
                    ┌──────▼─────┐
                    │    ALB     │
                    └──────┬─────┘
                           │
          ┌────────┬───────┼───────┬────────┐
          │        │       │       │        │
     ┌────▼──┐ ┌──▼───┐ ┌─▼──┐ ┌─▼────┐ ┌─▼──────┐
     │  API  │ │ Gate- │ │ Web│ │ File │ │ Link/  │
     │       │ │ way   │ │    │ │ Srv  │ │ GIF    │
     └───┬───┘ └──┬───┘ └────┘ └──┬───┘ │ Proxy  │
         │        │                │      └────────┘
    ┌────▼────────▼────┐     ┌────▼────┐
    │    PostgreSQL     │     │   S3    │
    └──────────────────┘     └─────────┘
              │
         ┌────▼────┐
         │  Redis  │
         └─────────┘
```

## Port Allocation

In development, all services run on localhost with distinct ports. In production, an Application Load Balancer routes traffic by path prefix:

| Path | Target |
|------|--------|
| `/api/*` | API service |
| `/ws` | Gateway (WebSocket upgrade) |
| `/autumn/*` | File server |
| `/january/*` | Link proxy |
| `/*` | Web client (catch-all) |
