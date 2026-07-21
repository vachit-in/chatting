# Chatting App Architecture

## High‑Level Design (HLDS)
1. **Client** – Single‑Page Application (SPA) built with Preact + Zustand for state.
2. **Realtime Layer** – WebSocket endpoint (`wss://api.chatting.in/ws`) powered by FastAPI + `uvicorn` with `websockets` library.
3. **REST API** – CRUD for users, rooms, messages, files; served by FastAPI, auto‑generated OpenAPI docs.
4. **Auth Service** – OAuth2 JWT (refreshable), built into the API; optional SSO via Google/Apple.
5. **Storage** – 
   * Metadata & indexes: PostgreSQL + Redis for pub/sub and caching.
   * Media files: S3 compatible object store with CloudFront CDN.
6. **Service Worker** – Caches static assets, intercepts fetches for offline mode; sync queue for unsent messages.
7. **CI/CD** – GitHub Actions: lint (flake8, eslint), tests, Docker build & push to GHCR, automated deploy to Fly.io or Railway.

## Low‑Level Design (LDDS)
- **Frontend**
  * `src/` contains component tree: `App`, `ChatList`, `ChatView`, `MessageInput`, `FileUpload`.
  * State split: global store for auth, rooms; local component state for input.
  * WebSocket connection handled via a custom hook (`useWebSocket`).
- **Backend**
  * `app/` with FastAPI routers: `/auth`, `/rooms`, `/messages`, `/files`.
  * Dependency injection for DB sessions, Redis client.
  * Background tasks (via `asyncio.create_task`) to push messages to WebSocket clients.
- **Service Worker**
  * Precache `index.html`, CSS/JS bundles and images.
  * Runtime caching strategy: Cache‑First for static, Network‑First for API with stale‑while‑revalidate.

## Performance & Offline Strategy
1. Bundle size < 200 KB (esbuild + Preact tree‑shaking).
2. Images served as WebP; optional lazy loading via `loading="lazy"`.
3. Use HTTP/2, keep‑alive sockets for websockets.
4. IndexedDB queue to store outgoing messages while offline; auto‑sync when back online.
5. Service Worker pushes notifications on new messages if the tab is not active.

## Security Considerations
- HTTPS enforced via HSTS.
- SameSite = Strict cookies, CSRF tokens for state‑changing endpoints.
- JWT signed with HS256; secrets stored in env variables.
- Rate limiting per IP using Redis token bucket.

---
> **Note**: This repo is a skeleton. Full implementation will be added incrementally.