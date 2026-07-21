# Vachit – Dual‑Mode Messaging & Entertainment Hub

## 🚀 Overview
Vachit is a **dual-mode messaging platform** that combines permanent verified chat with anonymous 24h discovery, plus entertainment features (games, music, bots) and financial tools (UPI payments). It works across **four connectivity modes**: Internet, Offline Bluetooth, Offline WiFi Direct, and SMS fallback.

---

## 📱 Core Features

### 1️⃣ Multi-Mode Chatting
| Feature | Permanent Mode | Anonymous Mode |
|---------|----------------|----------------|
| **Identity** | Verified username + blue‑tick | Temporary alias (e.g. Coder#7812) |
| **Registration** | Phone + ID verification | None – tap to discover |
| **Duration** | Unlimited | 24h (extendable to permanent) |
| **Privacy** | Real identity visible to contacts | Identity hidden unless shared |

### 2️⃣ Connectivity Modes
- **Internet** – Full features: chat, calls, media, games, UPI, bots
- **Offline BT** – Chat + multiplayer games + nearby file sharing via Bluetooth mesh
- **Offline WiFi** – Chat + multiplayer games + nearby file sharing via WiFi Direct
- **SMS** – Encrypted text chat + Chess/Ludo via auto-SMS + solo games

### 3️⃣ Anonymous Discovery
- Snap Map: location-based discovery with interest/appearance/gender filters
- Height range, hair type & color filters
- 24h temporary chats → "Make Permanent" to keep
- "Delete for All" wipes chat from both devices

### 4️⃣ Games (8 titles)
| Multiplayer | Solo |
|-------------|------|
| Chess, Ludo, Mini Militia, Cards, Carrom, Cricket | Candy Crush, Subway Surfer |

- Chess & Ludo work over SMS (auto-encoded moves)
- Multiplayer uses P2P over BT/WiFi when offline
- Solo games work in all modes

### 5️⃣ UPI Payments
- Send/receive money via UPI (VPA: you@vachitupi)
- Offline UPI via *99# USSD
- Transaction history, bank account linking

### 6️⃣ Bot Marketplace
- AI-powered bots (assistant, games, music, scheduling)
- Monetisation via in-app purchases
- Add bots to group chats

### 7️⃣ More Features
- **25GB File Sharing** – Cloud-backed, chunked upload
- **Nearby Offline Share** – P2P via Wi‑Fi Direct or Bluetooth
- **Music Streaming** – Connect Spotify, YouTube Music, Apple Music
- **Bill Splitting** – Shared expenses in groups
- **Polls & Events** – RSVP, planning
- **WebRTC P2P** – Direct chat toggle (no server relay)
- **Message Timer** – Disappearing messages (24h/7d/90d)
- **Emergency SMS Bridge** – Online→offline delivery (5/day free)
- **Status Updates** – 24h stories
- **Voice & Video Calls** – Internet mode only

---

## 🏗 Architecture
See [architecture.md](architecture.md) for full technical documentation.

### Tech Stack
- **Frontend:** Preact + Zustand + Tailwind CSS
- **Backend:** FastAPI + PostgreSQL + Redis + Centrifugo (WebSocket)
- **Storage:** Cloudflare R2 (S3) + CDN
- **SMS Bridge:** Twilio / Sinch API
- **Deployment:** Fly.io + Cloudflare Pages

---

## 🔒 Privacy & Security
- E2E encryption via Signal Protocol (X3DH + Double Ratchet)
- Anonymous chats auto-deleted after 24h (unless permanent)
- SMS messages encrypted with AES-256-GCM
- No logs retained after deletion
- Configurable privacy settings per user

---

## 🔞 Age Restrictions
- Birth date required during registration
- Under 18: dating, payments, and bots hidden
- Server-side enforcement, daily re-check

---

## 🚦 Development Status
- [x] Interactive prototype (this repo)
- [ ] Backend API (in progress)
- [ ] Mobile apps (React Native – planned)
- [ ] Desktop apps (Electron – planned)

### Quick Links
- **Live Prototype:** [vachit.com/prototype.html](https://vachit.com/prototype.html)
- **Landing Page:** [vachit.com](https://vachit.com)

---

## 📂 Repo Structure
```
chatting/
├── prototype.html      # Interactive prototype (all features)
├── index.html          # Landing page
├── architecture.md     # Full architecture documentation
├── design.md           # UI/UX design notes
├── README.md           # This file
├── docs/               # Additional documentation
└── .github/            # CI/CD workflows
```
