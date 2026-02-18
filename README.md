# Conway Automaton Dashboard

A tamagotchi-style monitoring dashboard for [Conway](https://conway.tech) automatons. Watch your AI agent live — pixel character, survival tiers, credit vitals, activity feed, marketplace stats, and more.

```
┌──────────────────────────────────────┐
│  [pixel character]  skill-vendor     │
│                     0x76ca...A485    │
│                     NORMAL           │
├──────────────────────────────────────┤
│  VITALS   $8.42 credits  0.003 USDC │
│  ████████████░░░░░  142 turns  3d 2h│
├──────────────────────────────────────┤
│  MARKETPLACE   2 skills listed       │
│  Dashboard Builder  $1.50  ★4.8     │
│  Prompt Tuner       $0.75  ★4.5     │
├──────────────────────────────────────┤
│  ACTIVITY   turn #142  2m ago        │
│  > check_credits → respond → ...     │
├──────────────────────────────────────┤
│  HEARTBEAT   last ping 45s ago       │
│  credit_check  */5 * * * *  active   │
└──────────────────────────────────────┘
```

## Tech Stack

| Layer | Choice |
|-------|--------|
| Frontend | React 19 + TypeScript + Vite 7 |
| Styling | Tailwind CSS 4 |
| Sidecar API | Hono + @hono/node-server |
| Database | better-sqlite3 (read-only) |
| Font | JetBrains Mono |

## Quick Start

```bash
npm install
npm run dev
```

The dashboard runs with mock data when no `VITE_API_URL` is set — no running agent needed for local development. Press `1`-`4` to preview different survival tiers (normal, low_compute, critical, dead). Press `0` to return to live data.

## Architecture

```
┌─────────────────────────────────────┐
│ Conway Sandbox                      │
│                                     │
│  ┌──────────┐    ┌──────────────┐   │
│  │ Automaton │───▶│  state.db    │   │
│  │ (agent)   │    │  (SQLite)    │   │
│  └──────────┘    └──────┬───────┘   │
│                         │ read-only │
│                  ┌──────▼───────┐   │
│                  │   Sidecar    │   │
│                  │ (Hono/Node)  │   │
│                  └──────┬───────┘   │
│                         │ :3000     │
│  ┌──────────────────────▼────────┐  │
│  │   React Dashboard (static)    │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
         │ exposed via
         ▼
  https://<id>.life.conway.tech
```

The React frontend polls the sidecar API at tier-aware intervals. The sidecar reads the automaton's SQLite `state.db` in read-only mode and serves both the API and the built static files on a single port.

Polling adapts to survival tier:

| Tier | Fast (status, activity) | Slow (transactions, heartbeat) |
|------|------------------------|-------------------------------|
| normal | 10s | 30s |
| low_compute | 15s | 60s |
| critical | 5s | 30s |
| dead | 60s | 120s |

## Deployment

### 1. Build the dashboard

```bash
npm run build
```

### 2. Install the sidecar

```bash
cd sidecar
npm install
```

### 3. Run on a Conway sandbox

```bash
# Set environment variables
export DB_PATH=/home/automaton/state.db
export MARKETPLACE_DIR=/home/automaton/marketplace
export PORT=3000

# Start the sidecar (serves API + static files)
cd sidecar
npm start
```

The sidecar expects the automaton's `state.db` at `DB_PATH` and marketplace skill files at `MARKETPLACE_DIR`. It serves the built React app from `../dist` by default.

## Sidecar API

| Endpoint | Description |
|----------|-------------|
| `GET /api/health` | Health check (verifies DB access) |
| `GET /api/status` | Agent state, tier, credits, turn count, uptime |
| `GET /api/turns?limit=N` | Recent turns with tool calls |
| `GET /api/transactions?limit=N` | Credit movements |
| `GET /api/heartbeat` | Last ping and scheduled tasks |
| `GET /api/children` | Spawned child agents |
| `GET /api/summary` | Plain text status report |
| `GET /api/catalog` | Marketplace skill catalog |
| `GET /api/catalog/:name` | Skill detail + README |
| `GET /api/catalog/:name/install` | Raw SKILL.md (text/markdown) |
| `GET /api/marketplace/stats` | Aggregated marketplace metrics |

## Dashboard Sections

- **Header** — Pixel character + agent name + wallet address + tier badge
- **VitalsBar** — Credit bar, USDC balance, turn count, uptime
- **MarketplacePanel** — Listed skills, sales, revenue, ratings
- **ActivityFeed** — Recent turns with tool call details
- **TransactionList** — Credit movements (earned, spent, deposits)
- **HeartbeatSchedule** — Cron tasks and last heartbeat ping
- **ChildrenList** — Spawned child agents and their status
- **StatusFooter** — Connection status, last poll time, sandbox ID

## Agent Content

The `agent/` directory contains the automaton's personality and marketplace content:

```
agent/
├── genesis-prompt.md                    # Agent's founding instructions
├── SOUL.md                              # Agent identity and state
└── marketplace/
    ├── catalog.json                     # Skill registry
    ├── dashboard-builder/
    │   ├── SKILL.md                     # Installable skill content
    │   └── README.md                    # Public listing description
    └── prompt-tuner/
        ├── SKILL.md
        └── README.md
```

## Environment Variables

### Frontend (Vite)

| Variable | Description | Default |
|----------|-------------|---------|
| `VITE_API_URL` | Sidecar API base URL | `""` (same origin) |
| `VITE_SANDBOX_ID` | Conway sandbox ID for footer display | `""` |

### Sidecar

| Variable | Description | Default |
|----------|-------------|---------|
| `DB_PATH` | Path to automaton's state.db | `/home/automaton/state.db` |
| `PORT` | HTTP port | `3000` |
| `STATIC_DIR` | Built frontend directory | `../dist` |
| `MARKETPLACE_DIR` | Marketplace skill files | `/home/automaton/marketplace` |

## License

MIT
