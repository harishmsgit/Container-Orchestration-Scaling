# StreamingApp — Project Workflow & Execution Guide

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Flow of Execution](#2-flow-of-execution)
3. [Service Details](#3-service-details)
4. [Docker Commands — Full Lifecycle](#4-docker-commands--full-lifecycle)
5. [Environment Configuration](#5-environment-configuration)
6. [API Route Map](#6-api-route-map)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         Browser (:3000)                         │
│                     React SPA via Nginx                         │
└───────┬──────────┬──────────┬──────────┬────────────────────────┘
        │          │          │          │
   HTTP REST   HTTP REST  HTTP REST  WebSocket + REST
        │          │          │          │
   ┌────▼───┐ ┌───▼────┐ ┌──▼───┐ ┌───▼───┐
   │  Auth  │ │Streaming│ │Admin │ │  Chat │
   │ :3001  │ │  :3002  │ │:3003 │ │ :3004 │
   └───┬────┘ └───┬────┘ └──┬───┘ └───┬───┘
       │          │          │         │
       └──────────┴──────┬───┴─────────┘
                         │
                   ┌─────▼─────┐
                   │  MongoDB  │
                   │  :27017   │
                   └───────────┘
```

| Service   | Image Base         | Internal Port | Host Port | Purpose                          |
|-----------|--------------------|---------------|-----------|----------------------------------|
| mongo     | `mongo:6`          | 27017         | 27017     | Database (shared by all backends)|
| auth      | `node:20-alpine`   | 3001          | 3001      | Authentication & user management |
| streaming | `node:20-alpine`   | 3002          | 3002      | Video streaming & catalog        |
| admin     | `node:20-alpine`   | 3003          | 3003      | Admin video management (CRUD)    |
| chat      | `node:20-alpine`   | 3004          | 3004      | Real-time chat via Socket.IO     |
| frontend  | `nginx:1.27-alpine`| 80            | 3000      | React SPA served by Nginx        |

---

## 2. Flow of Execution

### 2.1 Startup Sequence

```
1. docker compose up --build -d
   │
   ├─► [mongo]      Starts first, runs healthcheck:
   │                 mongosh --eval "db.adminCommand('ping')"
   │                 Retries 5 times at 10s intervals (start_period: 20s)
   │
   ├─► [auth]       Waits for mongo (service_healthy) → connects to MongoDB → listens on :3001
   ├─► [streaming]  Waits for mongo (service_healthy) → connects to MongoDB → listens on :3002
   ├─► [admin]      Waits for mongo (service_healthy) → connects to MongoDB → listens on :3003
   ├─► [chat]       Waits for mongo (service_healthy) → connects to MongoDB + Socket.IO → listens on :3004
   │
   └─► [frontend]   Waits for auth, streaming, admin, chat
                     (build-time: npm ci → npm run build → copies to nginx)
                     Listens on :80 (mapped to host :3000)
```

### 2.2 Request Flow — User Registration & Login

```
Browser → POST http://localhost:3001/api/register  → Auth Service → MongoDB (users collection)
Browser → POST http://localhost:3001/api/login      → Auth Service → JWT token returned
Browser → stores JWT in localStorage
Browser → GET  http://localhost:3001/api/verify     → Auth Service validates JWT → user context
```

### 2.3 Request Flow — Video Browsing

```
Browser → GET http://localhost:3002/api/streaming/videos/featured → Streaming Service → MongoDB
Browser → GET http://localhost:3002/api/streaming/videos?genre=X  → Streaming Service → MongoDB
Browser → GET http://localhost:3002/api/streaming/videos/:id      → Streaming Service → video details
Browser → GET http://localhost:3002/api/streaming/stream/:id      → Streaming Service → S3 signed URL / proxy
```

### 2.4 Request Flow — Admin Video Management

```
Browser → GET    http://localhost:3003/api/admin/videos            → Admin Service (adminAuth middleware)
Browser → POST   http://localhost:3003/api/admin/videos            → create video metadata
Browser → POST   http://localhost:3003/api/admin/videos/upload-urls/video     → get S3 pre-signed URL
Browser → POST   http://localhost:3003/api/admin/videos/upload-urls/thumbnail → get S3 pre-signed URL
Browser → PUT    http://localhost:3003/api/admin/videos/:id        → update video
Browser → DELETE http://localhost:3003/api/admin/videos/:id        → delete video
Browser → PATCH  http://localhost:3003/api/admin/videos/:id/featured → toggle featured flag
```

### 2.5 Request Flow — Real-Time Chat

```
Browser → WebSocket http://localhost:3004 (Socket.IO, JWT in handshake)
         ├─► emit 'chat:join' { videoId }     → server joins room, sends last 50 messages
         ├─► emit 'chat:message' { content }  → server broadcasts to room, saves to MongoDB
         ├─► emit 'chat:leave'                → server leaves room
         └─► REST GET http://localhost:3004/api/chat/history/:videoId → paginated history
```

### 2.6 Frontend Build Pipeline

```
Dockerfile (multi-stage):
  Stage 1 — Build:
    node:20-alpine
    ├─ COPY package*.json → npm ci
    ├─ COPY source code
    ├─ Inject REACT_APP_* env vars as build ARGs
    └─ npm run build → /app/build/

  Stage 2 — Production:
    nginx:1.27-alpine
    ├─ COPY --from=build /app/build → /usr/share/nginx/html
    ├─ COPY nginx.conf → /etc/nginx/conf.d/default.conf
    └─ SPA routing: try_files $uri $uri/ /index.html
```

---

## 3. Service Details

### 3.1 Auth Service (port 3001)

| Route                        | Method | Description                        |
|------------------------------|--------|------------------------------------|
| `/health`                    | GET    | Health check                       |
| `/api/register`              | POST   | User registration                  |
| `/api/login`                 | POST   | User login (returns JWT)           |
| `/api/forgetPassword`        | POST   | Password reset email               |
| `/api/verify`                | GET    | Validate JWT token                 |
| `/api/val/:vid/:email`       | GET    | Email verification callback        |

### 3.2 Streaming Service (port 3002)

| Route                                      | Method | Description              |
|--------------------------------------------|--------|--------------------------|
| `/api/health`                              | GET    | Health check             |
| `/api/streaming/videos`                    | GET    | List videos (by genre)   |
| `/api/streaming/videos/featured`           | GET    | Featured videos          |
| `/api/streaming/videos/:videoId`           | GET    | Video details            |
| `/api/streaming/stream/:videoId`           | GET    | Stream video             |
| `/api/streaming/videos/:videoId/stream`    | GET    | Stream video (alt)       |
| `/api/streaming/thumbnails/*`              | GET    | Video thumbnails         |

### 3.3 Admin Service (port 3003)

All routes protected by `adminAuth` middleware.

| Route                                      | Method | Description              |
|--------------------------------------------|--------|--------------------------|
| `/api/health`                              | GET    | Health check             |
| `/api/admin/videos`                        | GET    | List all videos          |
| `/api/admin/videos`                        | POST   | Create video             |
| `/api/admin/videos/upload-urls`            | POST   | Get upload URLs          |
| `/api/admin/videos/upload-urls/video`      | POST   | Get video upload URL     |
| `/api/admin/videos/upload-urls/thumbnail`  | POST   | Get thumbnail upload URL |
| `/api/admin/videos/upload/video`           | POST   | Upload video file        |
| `/api/admin/videos/upload/thumbnail`       | POST   | Upload thumbnail file    |
| `/api/admin/videos/:id`                    | PUT    | Update video             |
| `/api/admin/videos/:id`                    | DELETE | Delete video             |
| `/api/admin/videos/:id/featured`           | PATCH  | Toggle featured          |

### 3.4 Chat Service (port 3004)

| Route / Event                     | Type      | Description                |
|-----------------------------------|-----------|----------------------------|
| `/api/health`                     | GET       | Health check               |
| `/api/chat/history/:videoId`      | GET       | Chat history (REST)        |
| `chat:join`                       | Socket.IO | Join video chat room       |
| `chat:leave`                      | Socket.IO | Leave video chat room      |
| `chat:message`                    | Socket.IO | Send chat message          |
| `chat:history`                    | Socket.IO | Receive historical messages|

### 3.5 Frontend Pages

| Path               | Component        | Access         |
|--------------------|------------------|----------------|
| `/`                | LandingPage      | Public         |
| `/login`           | Login            | Public         |
| `/register`        | Register         | Public         |
| `/forgot-password` | ForgotPassword   | Public         |
| `/browse`          | Browse           | Authenticated  |
| `/admin/*`         | AdminDashboard   | Admin role     |
| `/settings`        | Settings         | Authenticated  |
| `/profile`         | Profile          | Authenticated  |
| `/collection`      | Collection       | Authenticated  |

---

## 4. Docker Commands — Full Lifecycle

### 4.1 Build & Start

```bash
# Build all images and start containers in detached mode
docker compose up --build -d

# Build without starting
docker compose build

# Build a specific service
docker compose build frontend
docker compose build auth streaming admin chat

# Build with no cache (clean rebuild)
docker compose build --no-cache

# Start without rebuilding (use existing images)
docker compose up -d
```

### 4.2 Stop & Remove

```bash
# Stop all running containers (keep images & volumes)
docker compose stop

# Stop a specific service
docker compose stop frontend

# Stop and remove containers + default network
docker compose down

# Stop, remove containers, AND delete volumes (⚠️ deletes DB data)
docker compose down -v

# Stop, remove containers, AND delete images
docker compose down --rmi all

# Nuclear option: remove everything (containers + images + volumes + orphans)
docker compose down --rmi all -v --remove-orphans
```

### 4.3 Restart & Recreate

```bash
# Restart all services
docker compose restart

# Restart a specific service
docker compose restart auth

# Force recreate containers (without rebuild)
docker compose up -d --force-recreate

# Recreate only one service
docker compose up -d --force-recreate --no-deps frontend
```

### 4.4 Logs

```bash
# View logs for all services
docker compose logs

# Follow logs in real-time
docker compose logs -f

# Follow logs for a specific service
docker compose logs -f auth

# Tail last 100 lines
docker compose logs --tail 100

# Logs for multiple services
docker compose logs -f auth streaming chat
```

### 4.5 Status & Inspection

```bash
# List running containers
docker compose ps

# List all containers (including stopped)
docker compose ps -a

# Show resource usage (CPU, memory, network)
docker stats

# Inspect a specific container
docker inspect streamingapp-auth-1

# Check health status
docker inspect --format='{{.State.Health.Status}}' streamingapp-mongo-1

# Validate compose file
docker compose config
```

### 4.6 Execute Commands Inside Containers

```bash
# Open a shell in a running container
docker exec -it streamingapp-auth-1 sh
docker exec -it streamingapp-mongo-1 mongosh

# Run a one-off command
docker exec streamingapp-auth-1 node -e "console.log(process.env.PORT)"

# Seed videos into the database
docker exec streamingapp-streaming-1 node scripts/seedVideos.js
```

### 4.7 Image Management

```bash
# List project images
docker image ls | findstr streamingapp

# Remove a specific image
docker image rm streamingapp-frontend

# Remove all unused images
docker image prune

# Remove ALL unused images (not just dangling)
docker image prune -a

# Show image build history / layers
docker history streamingapp-frontend
```

### 4.8 Volume Management

```bash
# List volumes
docker volume ls

# Inspect the MongoDB data volume
docker volume inspect streamingapp_mongo-data

# Remove the data volume (⚠️ deletes all DB data)
docker volume rm streamingapp_mongo-data

# Remove all unused volumes
docker volume prune
```

### 4.9 Network Management

```bash
# List networks
docker network ls

# Inspect the compose network
docker network inspect streamingapp_default

# See which containers are on the network
docker network inspect streamingapp_default --format='{{range .Containers}}{{.Name}} {{end}}'
```

### 4.10 Scaling (Horizontal)

```bash
# Scale a service to multiple replicas
docker compose up -d --scale streaming=3

# Scale back down
docker compose up -d --scale streaming=1
```

### 4.11 Health Checks (Quick Verification)

```bash
# Check all service health endpoints
curl http://localhost:3001/health
curl http://localhost:3002/api/health
curl http://localhost:3003/api/health
curl http://localhost:3004/api/health
curl -o /dev/null -s -w "%{http_code}" http://localhost:3000

# PowerShell equivalent
Invoke-WebRequest -Uri http://localhost:3001/health -UseBasicParsing
Invoke-WebRequest -Uri http://localhost:3002/api/health -UseBasicParsing
Invoke-WebRequest -Uri http://localhost:3003/api/health -UseBasicParsing
Invoke-WebRequest -Uri http://localhost:3004/api/health -UseBasicParsing
(Invoke-WebRequest -Uri http://localhost:3000 -UseBasicParsing).StatusCode
```

### 4.12 Common Workflows

```bash
# Full clean restart (rebuild everything from scratch)
docker compose down -v
docker compose build --no-cache
docker compose up -d

# Update only the frontend after code changes
docker compose up -d --build --no-deps frontend

# Update only a backend service after code changes
docker compose up -d --build --no-deps auth

# View container resource consumption
docker compose top

# Export container filesystem for debugging
docker export streamingapp-auth-1 > auth-debug.tar
```

---

## 5. Environment Configuration

### Required `.env` file (project root)

```ini
# Shared
CLIENT_URLS=http://localhost:3000
JWT_SECRET=changeme
MONGO_DB=streamingapp

# AWS (required for S3 video storage)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-south-1
AWS_S3_BUCKET=
AWS_CDN_URL=

# Service ports
AUTH_PORT=3001
STREAMING_PORT=3002
ADMIN_PORT=3003
CHAT_PORT=3004
STREAMING_PUBLIC_URL=http://localhost:3002

# Frontend build-time API URLs
REACT_APP_AUTH_API_URL=http://localhost:3001/api
REACT_APP_STREAMING_API_URL=http://localhost:3002/api
REACT_APP_STREAMING_PUBLIC_URL=http://localhost:3002
REACT_APP_ADMIN_API_URL=http://localhost:3003/api/admin
REACT_APP_CHAT_API_URL=http://localhost:3004/api/chat
REACT_APP_CHAT_SOCKET_URL=http://localhost:3004
```

> **Note:** Frontend `REACT_APP_*` variables are baked at build time. Changing them requires a rebuild of the frontend image.

---

## 6. API Route Map

```
http://localhost:3000                → Frontend (Nginx / React SPA)

http://localhost:3001                → Auth Service
  ├── GET  /health
  ├── POST /api/register
  ├── POST /api/login
  ├── POST /api/forgetPassword
  ├── GET  /api/verify
  └── GET  /api/val/:vid/:email

http://localhost:3002                → Streaming Service
  ├── GET  /api/health
  ├── GET  /api/streaming/videos
  ├── GET  /api/streaming/videos/featured
  ├── GET  /api/streaming/videos/:videoId
  ├── GET  /api/streaming/stream/:videoId
  └── GET  /api/streaming/thumbnails/*

http://localhost:3003                → Admin Service
  ├── GET    /api/health
  ├── GET    /api/admin/videos
  ├── POST   /api/admin/videos
  ├── POST   /api/admin/videos/upload-urls
  ├── POST   /api/admin/videos/upload-urls/video
  ├── POST   /api/admin/videos/upload-urls/thumbnail
  ├── POST   /api/admin/videos/upload/video
  ├── POST   /api/admin/videos/upload/thumbnail
  ├── PUT    /api/admin/videos/:id
  ├── DELETE /api/admin/videos/:id
  └── PATCH  /api/admin/videos/:id/featured

http://localhost:3004                → Chat Service
  ├── GET  /api/health
  ├── GET  /api/chat/history/:videoId
  └── WebSocket (Socket.IO)
       ├── chat:join
       ├── chat:leave
       ├── chat:message
       └── chat:history
```

---

## 7. Troubleshooting

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| Backend exits immediately | MongoDB not ready | Check `docker compose logs mongo`, ensure healthcheck passes |
| Frontend returns 404 on refresh | Missing nginx SPA config | Verify `nginx.conf` has `try_files $uri $uri/ /index.html` |
| CORS errors in browser | `CLIENT_URLS` mismatch | Ensure `.env` `CLIENT_URLS` matches browser origin |
| API calls fail from frontend | Wrong `REACT_APP_*` URLs | Rebuild frontend after changing: `docker compose up -d --build frontend` |
| MongoDB data lost after restart | Volume not persisted | Check `docker volume ls` for `streamingapp_mongo-data` |
| Container keeps restarting | App crash loop | Check `docker compose logs <service>` for error details |
| Port already in use | Another process on port | `netstat -ano \| findstr :<port>` then stop conflicting process |
| S3 upload fails | Missing AWS credentials | Set `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in `.env` |
| Chat not connecting | Socket.IO auth failure | Verify JWT token in localStorage and `JWT_SECRET` matches across services |
