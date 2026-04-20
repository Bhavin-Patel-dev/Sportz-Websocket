## 📖 Introduction

**Sportz** is a **real-time sports data backend** built with Node.js, Express, and WebSockets. It allows you to:

- Create and manage live sports matches across any sport

- Post live commentary events (goals, cards, substitutions, etc.) to a match as it happens

- **Monitor match lists and receive instantaneous score and play-by-play commentary updates** through a robust streaming architecture

- Subscribe to specific matches — only receive updates for the matches you care about

- Stay protected with enterprise-grade security via Arcjet (bot detection, rate limiting, shield protection) on both HTTP and WebSocket layers

Think of it as the **engine behind a live sports ticker or scoreboard** — like what powers the little score popup on Google when you search a football match. The platform's streaming architecture features heartbeats, per-match subscriptions, and rate limiting — built for real-world reliability.

---

## ✨ Key Features

| Feature                                 | Description                                                                            |
| --------------------------------------- | -------------------------------------------------------------------------------------- |
| 🔴 **Real-Time WebSocket Broadcasting** | Instantly pushes match creation and play-by-play commentary to connected clients       |
| 🎯 **Per-Match Subscriptions**          | Clients subscribe to specific match IDs — only receive updates for matches they follow |
| 💓 **Heartbeat Keep-Alive**             | Ping/pong every 30s with `isAlive` tracking — dead connections are auto-terminated     |
| ⚽ **Multi-Sport Support**              | Flexible schema supports football, cricket, basketball, and any sport                  |
| 📝 **Rich Commentary System**           | Commentary events with minute, period, actor, team, event type, metadata, and tags     |
| 🛡️ **Dual-Layer Arcjet Security**       | Bot detection, sliding window rate limiting, and shield on both HTTP and WebSocket     |
| ✅ **Zod v4 Validation**                | Schema-first request validation with detailed error messages                           |
| 🗄️ **Drizzle ORM + PostgreSQL**         | Type-safe queries with a code-first schema and migration support                       |
| 🔒 **Environment-Aware Security**       | `DRY_RUN` mode for development, `LIVE` mode for production — zero code changes needed  |
| 📦 **ES Modules**                       | Fully modern ESM codebase — no CommonJS                                                |

---

## 🏗️ How It Works

```
Client (REST/WS)        Express + WS Server           PostgreSQL
    │                         │                            │
    │── POST /matches ────────▶│                            │
    │                         │── Insert match ───────────▶│
    │                         │◀─ Return match ────────────│
    │◀── 201 Created ─────────│                            │
    │                         │                            │
    │         ┌───────────────▼──────────────────┐         │
    │         │  Broadcast to ALL ws:// clients  │         │
    │         │  { type: "match_created", ... }  │         │
    │         └──────────────────────────────────┘         │
    │                         │                            │
    │── WS: subscribe ────────▶│                            │
    │         { type: "subscribe", matchId: 4 }            │
    │◀── { type: "subscribed", matchId: 4 } ──────         │
    │                         │                            │
    │── POST /matches/4/commentary ──────────────▶│        │
    │                         │── Insert commentary ──────▶│
    │◀── 201 Created ─────────│◀─ Return commentary ───────│
    │                         │                            │
    │         ┌───────────────▼──────────────────┐         │
    │         │  Broadcast to match 4 subscribers│         │
    │         │  { type: "commentary", ... }     │         │
    │         └──────────────────────────────────┘         │
    │                         │                            │
    │         ┌───────────────▼──────────────────┐         │
    │         │  Heartbeat: ping every 30s       │         │
    │         │  No pong? → terminate connection │         │
    │         └──────────────────────────────────┘         │
```

### Request Lifecycle

1. **Request arrives** → `express.json()` parses the body

2. **Arcjet middleware** runs: checks for bots, rate limits, and suspicious patterns

3. **Route handler** validates params/body using Zod schemas

4. **Drizzle ORM** executes a type-safe query against PostgreSQL

5. **WebSocket broadcast** fires — match creation goes to all clients; commentary goes only to subscribers of that match

6. **Response** is returned to the caller

### WebSocket Connection Lifecycle

1. Client connects to `ws://host/ws`

2. Arcjet runs independently on the WS connection — rate limited to 5 req/2s

3. Server sends `{ type: "welcome" }` immediately

4. Client sends `{ type: "subscribe", matchId: 4 }` to follow a match

5. Server sends targeted `commentary` events only to that match's subscribers

6. Every 30s — server pings all clients; unresponsive ones are terminated

7. On disconnect — server automatically cleans up all subscriptions

---

## 📁 Project Structure

```
Sportz/
├── src/
│   ├── index.js                  # App entry point — Express + HTTP + WS server setup
│   ├── arcjet.js                 # Arcjet security middleware (bot detection, rate limiting)
│   │
│   ├── routes/
│   │   ├── matches.js            # GET /matches, POST /matches
│   │   └── commentary.js         # GET /matches/:id/commentary, POST /matches/:id/commentary
│   │
│   ├── ws/
│   │   └── server.js             # WebSocket server — heartbeats, subscriptions, broadcast logic
│   │
│   ├── db/
│   │   ├── db.js                 # Drizzle DB client instance
│   │   └── schema.js             # Table definitions (matches, commentary)
│   │
│   ├── validation/
│   │   ├── matches.js            # Zod schemas for match routes
│   │   └── commentary.js         # Zod schemas for commentary routes
│   │
│   └── utils/
│       └── match-status.js       # Helper: derives match status from start/end times
│
├── drizzle.config.js             # Drizzle Kit config
├── .env                          # Environment variables (not committed)
├── package.json
└── README.md
```

---

## 🚀 Getting Started

### Prerequisites

- Node.js v22+
- PostgreSQL database
- An [Arcjet](https://arcjet.com) account and API key

### 1. Clone the repo

```bash
git clone https://github.com/Bhavin-Patel-dev/Sportz-Websocket.git
cd Sportz-Websocket
```

### 2. Install dependencies

```bash
npm install
```

### 3. Set up environment variables

Create a `.env` file in the root:

```env
# Server
PORT=8000
HOST=0.0.0.0
NODE_ENV=development

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/sportz

# Arcjet Security
ARCJET_KEY=your_arcjet_key_here
ARCJET_MODE=DRY_RUN        # Use LIVE in production
```

### 4. Generate and run migrations

```bash
npm run db:generate
npm run db:migrate
```

### 5. Start the server

```bash
npm run dev
```

Server runs at `http://localhost:8000`
WebSocket available at `ws://localhost:8000/ws`

---

## 📡 API Reference

### Matches

| Method | Endpoint   | Description                           |
| ------ | ---------- | ------------------------------------- |
| `GET`  | `/matches` | List all matches (supports `?limit=`) |
| `POST` | `/matches` | Create a new match                    |

#### POST `/matches` — Request Body

```json
{
  "sport": "football",
  "homeTeam": "Manchester City",
  "awayTeam": "Manchester United",
  "startTime": "2026-04-15T14:00:00.000Z",
  "endTime": "2026-04-15T16:00:00.000Z",
  "homeScore": 0,
  "awayScore": 0
}
```

---

### Commentary

| Method | Endpoint                  | Description                 |
| ------ | ------------------------- | --------------------------- |
| `GET`  | `/matches/:id/commentary` | List commentary for a match |
| `POST` | `/matches/:id/commentary` | Add a commentary event      |

#### POST `/matches/:id/commentary` — Request Body

```json
{
  "minutes": 65,
  "sequence": 120,
  "period": "2nd half",
  "eventType": "goal",
  "actor": "Phil Foden",
  "team": "Manchester City",
  "message": "Sensational finish from a freekick!",
  "metadata": { "assistedBy": "Kevin De Bruyne" },
  "tags": ["goal", "freekick"]
}
```

---

### WebSocket

Connect to the WebSocket server:

```bash
npx wscat -c ws://127.0.0.1:8000/ws
```

#### Message Protocol

**On connect** — server sends a welcome:

```json
{ "type": "welcome" }
```

**Subscribe to a match** — receive only that match's commentary:

```json
{ "type": "subscribe", "matchId": 4 }
```

```json
{ "type": "subscribed", "matchId": 4 }
```

**On match creation** — broadcast to ALL connected clients:

```json
{
  "type": "match_created",
  "data": {
    "id": 4,
    "sport": "football",
    "homeTeam": "Manchester City",
    "awayTeam": "Manchester United",
    "status": "scheduled"
  }
}
```

**On commentary added** — sent only to subscribers of that match:

```json
{
  "type": "commentary",
  "data": {
    "matchId": 4,
    "minutes": 65,
    "eventType": "goal",
    "actor": "Phil Foden",
    "message": "Sensational finish from a freekick!"
  }
}
```

**Unsubscribe from a match:**

```json
{ "type": "unsubscribe", "matchId": 4 }
```

> **Heartbeat:** The server pings every client every **30 seconds**. Clients that don't respond with a pong are automatically terminated and their subscriptions cleaned up.

---

## 🔐 Security — Arcjet

This project uses [Arcjet](https://arcjet.com) for production-grade API protection on **both HTTP and WebSocket layers independently**:

| Rule              | HTTP                                       | WebSocket                                            |
| ----------------- | ------------------------------------------ | ---------------------------------------------------- |
| `shield`          | Blocks SQLi, XSS, attack patterns          | Same                                                 |
| `detectBot`       | Blocks malicious bots                      | Same                                                 |
| `slidingWindow`   | 50 req / 10s                               | 5 req / 2s                                           |
| **Deny response** | `403 Forbidden` or `429 Too Many Requests` | WS close code `1008` (policy) or `1013` (rate limit) |

**Development tip:** Set `ARCJET_MODE=DRY_RUN` in `.env` to log decisions without blocking — essential when testing with Postman, Thunder Client, or curl.

---

## 🧠 Learnings & Takeaways

Building this project was a deep dive into several real-world backend concepts. Here's what stands out:

**WebSockets & HTTP coexistence**
Running both an HTTP REST API and a WebSocket server on the same port using Node's `http.createServer()` and upgrading connections — understanding how the same TCP port handles both protocols.

**WebSocket heartbeats & connection health**
Implementing a ping/pong heartbeat loop with `isAlive` flags to detect and terminate zombie connections. Without this, dead clients silently accumulate and waste server memory.

**Per-match pub/sub with a Map of Sets**
Using `Map<matchId, Set<socket>>` to manage targeted subscriptions — broadcasting commentary only to clients subscribed to a specific match, rather than flooding all connected clients.

**Arcjet on WebSocket connections**
Protecting WebSocket upgrades with independent rate limiting — using WebSocket close codes (`1008` for policy violation, `1013` for rate limit) instead of HTTP status codes.

**ESM in Node.js**
ES Modules require explicit `.js` extensions on all imports — unlike CommonJS which resolves them automatically. This is a common gotcha when migrating or starting modern Node.js projects.

**Drizzle ORM type safety**
Code-first schema definitions with `pg-core` — understanding how `.array()`, `text()`, `jsonb()`, and `integer()` map to actual PostgreSQL column types, and why a wrong type (like `integer` for a `period` string) causes hard runtime crashes.

**Zod v4 breaking changes**
`z.record()` now requires two arguments in Zod v4 (`z.record(z.string(), z.unknown())`). Also `z.iso.datetime()` replaces `z.string().datetime()`. Staying current with library major versions matters.

**Express router `mergeParams`**
When mounting child routers, parent route params (like `:id`) are stripped by default. `Router({ mergeParams: true })` is essential for nested routes to access parent params.

**Arcjet for real-world API security**
Integrating bot detection and rate limiting as middleware — and understanding why HTTP clients like Postman appear as "bots" to a `detectBot` rule during development.

**IPv6 vs IPv4 on localhost**
Node.js 18+ resolves `localhost` to `::1` (IPv6) by default. If your server binds to `0.0.0.0` (IPv4), you must use `127.0.0.1` explicitly in clients.

---

## 🛠️ Tech Stack

| Technology      | Role                                    |
| --------------- | --------------------------------------- |
| **Node.js v22** | Runtime                                 |
| **Express 5**   | HTTP framework                          |
| **ws**          | WebSocket server                        |
| **Drizzle ORM** | Type-safe ORM + migrations              |
| **PostgreSQL**  | Primary database                        |
| **Zod v4**      | Schema validation                       |
| **Arcjet**      | Security (rate limiting, bot detection) |
| **drizzle-kit** | Schema generation & migrations          |

---

## 🤝 Contributing

This is a **backend-only project** — and that's intentional.

If you'd like to contribute, here are some great ways:

- 🐛 **Found a bug?** [Open an issue](https://github.com/Bhavin-Patel-dev/Sportz-Websocket/issues)
- 💡 **Have an idea?** Raise a feature request via issues

- 🎨 **Frontend developer?** This backend is wide open — **build a UI around it!** A live scoreboard, a match dashboard, a real-time commentary feed — the WebSocket is ready and waiting for you

- ⭐ **If you found this useful or learned something from it, drop a star on the repo** — it genuinely helps and means a lot!

### How to contribute

```bash
# Fork the repo, then:
git clone https://github.com/YOUR_USERNAME/Sportz-Websocket.git
cd Sportz-Websocket
git checkout -b feature/your-feature-name

# Make your changes, then:
git commit -m "feat: your feature description"
git push origin feature/your-feature-name
# Open a Pull Request 🚀
```

---

<div align="center">

**Made with ☕ + 💻 by [Bhavin Patel](https://github.com/Bhavin-Patel-dev)**

_Built to learn. Shared to help others learn too._

⭐ **Star this repo if it helped you** ⭐

</div>
