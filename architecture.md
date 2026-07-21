# Vachit – Architecture & Implementation Docs

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Connectivity Modes](#2-connectivity-modes)
3. [Data Models](#3-data-models)
4. [API Design](#4-api-design)
5. [Frontend Architecture](#5-frontend-architecture)
6. [Offline & P2P](#6-offline--p2p)
7. [Anonymous System](#7-anonymous-system)
8. [Games System](#8-games-system)
9. [UPI Payments](#9-upi-payments)
10. [Security & Privacy](#10-security--privacy)
11. [Age Restrictions](#11-age-restrictions)
12. [Deployment](#12-deployment)

---

## 1. System Overview

```
┌─────────────────────────────────────────────────────┐
│                    CLIENT (SPA)                       │
│  Preact + Zustand + Service Worker + IndexedDB       │
├──────────┬──────────┬──────────┬────────────────────┤
│ Internet │ BT (BLE) │ WiFi P2P │ SMS (Encrypted)    │
│   API    │  Mesh    │  Direct  │  Carrier Bridge     │
└────┬─────┴────┬─────┴────┬─────┴──────┬─────────────┘
     │          │          │            │
     ▼          ▼          ▼            ▼
┌─────────────────────────────────────────────────────┐
│                  BACKEND SERVICES                     │
│  FastAPI + PostgreSQL + Redis + S3 + WebSocket       │
│  ┌─────────┐ ┌──────────┐ ┌──────────────────────┐  │
│  │ REST    │ │ Realtime │ │ SMS Bridge Service    │  │
│  │ API     │ │ WS/Centrifugo│ (Twilio/Sinch API) │  │
│  └─────────┘ └──────────┘ └──────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### Tech Stack
| Layer | Technology |
|-------|-----------|
| Frontend | Preact 10 + Zustand + Tailwind CSS |
| Service Worker | Workbox (offline cache + sync) |
| Backend API | FastAPI (Python 3.12) |
| Realtime | WebSocket via Centrifugo |
| Database | PostgreSQL 16 + Redis 7 |
| File Storage | S3-compatible (Cloudflare R2) + CDN |
| Push Notifications | Firebase Cloud Messaging + APNs |
| SMS Bridge | Twilio / Sinch API |
| CI/CD | GitHub Actions → Docker → Fly.io |

---

## 2. Connectivity Modes

### Mode State Machine
```
     ┌──────────┐
     │ Internet  │ ◄──── default
     └────┬─────┘
          │ no internet detected
    ┌─────┴──────┐
    │  Offline   │
    │ BT / WiFi  │
    └─────┬──────┘
          │ no BT/WiFi peer
    ┌─────┴──────┐
    │    SMS     │
    │  Fallback  │
    └────────────┘
```

### Mode Capabilities
| Feature | Internet | BT / WiFi P2P | SMS |
|---------|:--------:|:-------------:|:---:|
| Text Chat | ✅ | ✅ | ✅ |
| Media (img/video/audio) | ✅ | ❌ | ❌ |
| Document Sharing (≤25GB) | ✅ | ✅ (nearby) | ❌ |
| Voice/Video Calls | ✅ | ❌ | ❌ |
| Games (Solo) | ✅ | ✅ | ✅ |
| Games (Multiplayer) | ✅ | ✅ (P2P) | ✅ (Chess/Ludo via SMS) |
| UPI Payments | ✅ | ❌ | ❌ |
| Anonymous Discovery | ✅ | ❌ | ❌ |
| Bots | ✅ | ❌ | ❌ |
| Music Streaming | ✅ | ❌ | ❌ |
| Status Updates | ✅ | ✅ | ❌ |

### Emergency SMS Bridge
- When User A (online) sends to User B (offline): server routes via Twilio encrypted SMS
- Limit: 5 SMS/day for free tier, unlimited for premium
- User B (offline) → User A (online): unlimited (carrier SMS → server → WebSocket to A)
- Encryption: AES-256-GCM with per-session keys exchanged during initial pairing

---

## 3. Data Models

### Users
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone VARCHAR(15) UNIQUE NOT NULL,
  username VARCHAR(30) UNIQUE,
  display_name VARCHAR(50),
  avatar_url TEXT,
  about TEXT,
  verified BOOLEAN DEFAULT false,
  birth_date DATE,
  created_at TIMESTAMPTZ DEFAULT now(),
  last_seen TIMESTAMPTZ
);
```

### Chats & Messages
```sql
CREATE TABLE chats (
  id UUID PRIMARY KEY,
  type VARCHAR(10) CHECK (type IN ('direct','group','anonymous')),
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE chat_members (
  chat_id UUID REFERENCES chats(id),
  user_id UUID REFERENCES users(id),
  role VARCHAR(10) DEFAULT 'member',
  PRIMARY KEY (chat_id, user_id)
);

CREATE TABLE messages (
  id UUID PRIMARY KEY,
  chat_id UUID REFERENCES chats(id),
  sender_id UUID REFERENCES users(id),
  content TEXT,
  type VARCHAR(20) DEFAULT 'text',  -- text, image, file, game_move, poll, payment, date_request, system
  metadata JSONB DEFAULT '{}',
  connectivity_mode VARCHAR(10) DEFAULT 'internet',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### Anonymous Sessions
```sql
CREATE TABLE anonymous_sessions (
  id UUID PRIMARY KEY,
  alias VARCHAR(20) UNIQUE NOT NULL,  -- e.g. "Coder#7812"
  chat_id UUID REFERENCES chats(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  expires_at TIMESTAMPTZ DEFAULT (now() + interval '24 hours'),
  is_permanent BOOLEAN DEFAULT false,
  deleted_by_user BOOLEAN DEFAULT false
);
```

### Games
```sql
CREATE TABLE games (
  id UUID PRIMARY KEY,
  chat_id UUID REFERENCES chats(id),
  game_type VARCHAR(20),  -- chess, ludo, minimilitia, cards, candycrush, etc.
  player1_id UUID REFERENCES users(id),
  player2_id UUID REFERENCES users(id),
  state JSONB NOT NULL DEFAULT '{}',
  turn UUID REFERENCES users(id),
  winner_id UUID REFERENCES users(id),
  mode VARCHAR(10) DEFAULT 'internet',  -- internet, p2p, sms
  created_at TIMESTAMPTZ DEFAULT now(),
  ended_at TIMESTAMPTZ
);
```

### Payments
```sql
CREATE TABLE transactions (
  id UUID PRIMARY KEY,
  from_user UUID REFERENCES users(id),
  to_user UUID REFERENCES users(id),
  amount DECIMAL(10,2),
  currency VARCHAR(3) DEFAULT 'INR',
  status VARCHAR(15) DEFAULT 'pending',
  upi_ref VARCHAR(50),
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

## 4. API Design

### Base URL: `https://api.vachit.com/v1`

### Auth
```
POST   /auth/register          # Phone + OTP
POST   /auth/verify            # Verify OTP
POST   /auth/login             # JWT token pair
POST   /auth/refresh           # Refresh access token
```

### Users
```
GET    /users/me               # Current user profile
PUT    /users/me               # Update profile
GET    /users/:id              # Public profile
PUT    /users/me/settings      # Privacy settings
```

### Chats
```
GET    /chats                  # List user's chats
POST   /chats                  # Create chat (direct/group)
GET    /chats/:id              # Chat details
GET    /chats/:id/messages     # Paginated messages
POST   /chats/:id/messages     # Send message
DELETE /chats/:id/messages/:mid # Delete for everyone
```

### Anonymous
```
POST   /anon/discover          # Register for discovery (location + interests)
GET    /anon/nearby            # Get nearby users (lat/lng + filters)
POST   /anon/request           # Send chat request to nearby user
PUT    /anon/:id/permanent     # Convert to permanent anonymous friend
DELETE /anon/:id               # Delete chat from both devices
```

### Games
```
POST   /games/start            # Start game in chat
POST   /games/:id/move         # Submit move
GET    /games/:id              # Game state
POST   /games/:id/sms-move     # Submit move via SMS gateway
```

### Payments (UPI)
```
POST   /payments/send          # Initiate UPI transfer
GET    /payments/transactions  # Transaction history
POST   /payments/verify        # Verify UPI mandate
```

### Files
```
POST   /files/upload           # Multipart upload (chunked for >100MB)
GET    /files/:id              # Download file
```

---

## 5. Frontend Architecture

### Component Tree
```
App
├── Header (connectivity selector, theme, search, menu)
├── TabBar (Chats | Anon | UPI | Status | Calls)
├── Panels
│   ├── ChatsPanel
│   │   ├── SearchBar
│   │   ├── ChatList
│   │   └── ChatItem (avatar, name, preview, unread count)
│   ├── AnonymousPanel
│   │   ├── DiscoveryToggle
│   │   ├── SnapMap (location-based user markers)
│   │   ├── NearbyList (list view)
│   │   └── FilterSheet (interests, appearance, height, hair, gender)
│   ├── UpiPanel (balance, send/receive, transactions)
│   ├── StatusPanel (status ring, updates)
│   └── CallsPanel (call history)
├── ChatScreen (overlay)
│   ├── ChatHeader (back, avatar, name, P2P toggle, call buttons)
│   ├── AnonBanner (24h timer, make permanent, delete)
│   ├── MessageList
│   │   ├── TextMessage
│   │   ├── FileMessage
│   │   ├── NearbyCard
│   │   ├── BotCard
│   │   └── DateRequestCard
│   ├── SlashCommandPopup
│   ├── AttachSheet (Doc, Camera, Gallery, Location, Contact, Offline, Pay, Date, Poll, Split, Bot, Music)
│   └── MessageInput
├── SettingsPage (full screen)
│   ├── Profile, Privacy, Messages, Connectivity, Music, Age, Account
│   └── Toggle switches, modals for each section
├── GamesModal (from header menu)
│   └── GameCards (8 games, gated by connectivity)
├── HeightModal, HairModal (anonymous filters)
├── FilterSheet (bottom slide-up)
└── Toast notifications
```

### State Management (Zustand)
```js
const useStore = create((set, get) => ({
  // Connectivity
  connectivity: 'internet',
  setConnectivity: (mode) => set({ connectivity: mode }),

  // Auth
  user: null,
  isUnder18: false,

  // Chats
  activeChat: null,
  chats: [],
  messages: {},

  // Anonymous
  anonDiscoveryOn: false,
  nearbyUsers: [],
  permanentFriends: {},
  deletedChats: {},

  // Filters
  filterInterest: 'All',
  filterGender: 'All',
  filterFindSpecific: false,
  filterHeightMin: null,
  filterHeightMax: null,
  filterHairType: null,
  filterHairColor: null,

  // Settings
  theme: 'light',
  msgTimer: 'off',
  connPriority: 'none',
  musicApps: [],
}));
```

---

## 6. Offline & P2P

### Service Worker Strategy
```
1. Precache: index.html, CSS, JS bundles, game assets
2. Runtime:
   - Static assets: Cache-First (1 year expiry)
   - API calls: Network-First (3s timeout, fallback to cache)
   - Messages: Network-First, queue in IndexedDB on failure
3. Background Sync: 'sync' event → flush IndexedDB message queue
```

### P2P WebRTC (Direct Chat)
```
1. Signaling via WebSocket (exchange SDP offers/answers)
2. STUN/TURN servers for NAT traversal
3. DataChannel for text messages
4. Encryption: DTLS-SRTP (built into WebRTC)
5. Fallback: if P2P fails, route via WebSocket (server relay)
```

### Bluetooth Mesh (Offline BT)
```
1. BLE advertising with service UUID
2. GATT characteristics for message exchange
3. Mesh flooding for multi-hop
4. Payload encrypted with pre-shared session key
5. Max payload: 512 bytes per BLE packet
```

### WiFi Direct (Offline WiFi)
```
1. WiFi P2P (Android) / Multipeer Connectivity (iOS)
2. TCP socket between peers on random port
3. JSON-encoded messages over TLS
4. File transfer: chunked binary streaming
```

### SMS Bridge
```
Online User → Server (WebSocket) → Twilio API → SMS → Offline User
Offline User → SMS → Twilio webhook → Server (WebSocket) → Online User
```

---

## 7. Anonymous System

### Discovery Flow
```
1. User enables discovery → sends location + interests to server
2. Server stores hashed location (geohash, precision ~100m)
3. Nearby query: SELECT * FROM anon_sessions WHERE geohash LIKE 'prefix%'
4. Results filtered client-side by interests, appearance, gender, height, hair
5. Chat request → server creates anonymous chat (24h TTL)
6. Both parties can "Make Permanent" → TTL removed, friendship persists
7. Either party can "Delete for All" → chat wiped from both devices
```

### Privacy Guarantees
- Identity never revealed unless voluntarily shared
- Location hashed with 100m precision
- All anonymous chats E2E encrypted
- Auto-deleted after 24h unless converted to permanent
- No logs retained after deletion

---

## 8. Games System

### Architecture
```
┌─────────┐    ┌──────────┐    ┌─────────┐
│ Player A │◄──►│ Game     │◄──►│ Player B│
│ (Client) │    │ Engine   │    │ (Client)│
└─────────┘    │ (Server) │    └─────────┘
               └──────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    Internet    BT/WiFi P2P    SMS
    (WebSocket) (TCP/UDP)   (SMS Gateway)
```

### Game Types
| Game | Players | Mode | Offline Support |
|------|---------|------|-----------------|
| Chess | 2 | Turn-based | Internet, P2P, SMS |
| Ludo | 2-4 | Turn-based | Internet, P2P, SMS |
| Mini Militia | 2-6 | Real-time | Internet, P2P |
| Cards | 2-8 | Turn-based | Internet, P2P |
| Candy Crush | 1 | Solo | All modes |
| Subway Surfer | 1 | Solo | All modes |
| Carrom | 2-4 | Turn-based | Internet, P2P |
| Cricket | 2 | Real-time | Internet, P2P |

### SMS Game Protocol (Chess/Ludo)
```
Move encoded as: "VACHIT:GAME:chess:<game_id>:<from_sq><to_sq>:<seq>"
Example: "VACHIT:GAME:chess:abc123:e2e4:42"
Encrypted with session key → sent via SMS
Receiving app intercepts SMS (filter by "VACHIT:GAME:" prefix)
Decrypts, validates, applies move locally
```

---

## 9. UPI Payments

### Integration
```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Vachit  │────►│  UPI     │────►│  NPCI    │
│  Server  │     │  Gateway │     │  Network │
└──────────┘     │  (Razorpay│     └──────────┘
                  │  /PhonePe)│
                  └──────────┘
```

### Flow
1. User enters amount + recipient VPA (e.g., user@vachitupi)
2. Server creates payment intent via UPI gateway
3. Gateway returns deep link / collect request
4. User authorizes via UPI PIN in their UPI app
5. Webhook confirms payment → funds transferred
6. Transaction stored in `transactions` table

### Offline UPI
- Uses NPCI's *99# USSD service
- Works without internet via SMS-based UPI
- Limited to basic send/receive

---

## 10. Security & Privacy

### Encryption
| Layer | Algorithm |
|-------|----------|
| Transport | TLS 1.3 |
| Messages (E2E) | Signal Protocol (X3DH + Double Ratchet) |
| SMS Bridge | AES-256-GCM with pre-shared session key |
| P2P | DTLS-SRTP (WebRTC built-in) |
| Files at rest | AES-256 (S3 server-side encryption) |

### Authentication
- OAuth2 JWT (access + refresh tokens)
- Access token: 15 min expiry
- Refresh token: 30 days, one-time use, rotated on each refresh
- Phone OTP for initial registration

### Privacy Settings (per user)
```json
{
  "last_seen_visibility": "everyone|contacts|nobody",
  "read_receipts": true,
  "profile_photo_visibility": "everyone|contacts|nobody",
  "live_location_sharing": false,
  "message_timer": "off|24h|7d|90d"
}
```

---

## 11. Age Restrictions

### Under-18 Policy
- Birth date collected during registration (mandatory)
- If age < 18 at time of check:
  - ❌ Dating features hidden
  - ❌ UPI/Payments hidden
  - ❌ Bot marketplace hidden
  - ❌ `/date`, `/pay`, `/bot` commands blocked
- Age re-checked daily server-side
- Attempts to bypass (changing birth date) flagged for review

### Implementation
```python
def is_under_18(user: User) -> bool:
    if not user.birth_date:
        return True  # Safety: assume under 18 if no birth date
    today = date.today()
    age = today.year - user.birth_date.year
    if today.month < user.birth_date.month or (
        today.month == user.birth_date.month and today.day < user.birth_date.day
    ):
        age -= 1
    return age < 18

def filter_features_for_user(user: User, features: list) -> list:
    if is_under_18(user):
        return [f for f in features if f not in ('dating', 'payments', 'bots')]
    return features
```

---

## 12. Deployment

### Infrastructure
```
┌────────────────────────────────────────────┐
│                 Fly.io                      │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐ │
│  │ API     │  │ Centrifugo│  │ Worker    │ │
│  │ (2x CPU)│  │ (1x CPU) │  │ (1x CPU)  │ │
│  └─────────┘  └──────────┘  └───────────┘ │
└────────────────────────────────────────────┘
         │              │
    ┌────┴────┐    ┌────┴────┐
    │ PostgreSQL│   │  Redis  │
    │ (Supabase)│   │ (Upstash)│
    └──────────┘    └─────────┘

┌─────────────────────────────────┐
│        Cloudflare               │
│  ┌──────────┐  ┌─────────────┐ │
│  │ R2 (S3)  │  │ Pages (CDN) │ │
│  └──────────┘  └─────────────┘ │
└─────────────────────────────────┘
```

### CI/CD Pipeline
```yaml
# .github/workflows/deploy.yml
name: Deploy
on: push branches [main]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test
      - run: pip install -r requirements.txt && pytest

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
      - uses: cloudflare/pages-action@v1
        with:
          directory: dist/
      - uses: superfly/flyctl-actions@v1
        with:
          args: deploy
```

### Environment Variables
```
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
JWT_SECRET=...
TWILIO_SID=...
TWILIO_AUTH_TOKEN=...
UPI_GATEWAY_KEY=...
S3_ENDPOINT=...
S3_ACCESS_KEY=...
S3_SECRET_KEY=...
CENTRIFUGO_API_KEY=...
```

---

## Appendix: File Structure
```
vachit/
├── client/                  # Frontend SPA
│   ├── src/
│   │   ├── components/      # UI components
│   │   ├── hooks/           # Custom hooks (useWebSocket, useConnectivity)
│   │   ├── stores/          # Zustand stores
│   │   ├── workers/         # Service Worker + Web Workers
│   │   ├── lib/             # Encryption, SMS parsing, game engines
│   │   └── styles/          # Tailwind config
│   ├── public/              # Static assets, manifest.json
│   └── package.json
├── server/                  # FastAPI Backend
│   ├── app/
│   │   ├── api/             # Route handlers
│   │   ├── models/          # SQLAlchemy models
│   │   ├── services/        # Business logic (sms_bridge, game_engine, upi)
│   │   ├── core/            # Config, security, deps
│   │   └── main.py          # Entry point
│   ├── tests/
│   ├── requirements.txt
│   └── Dockerfile
├── docs/                    # Documentation
│   ├── architecture.md      # This file
│   ├── api.md               # API reference
│   └── design.md            # UI/UX design notes
├── .github/workflows/       # CI/CD
├── prototype.html           # Interactive prototype
└── README.md
```
