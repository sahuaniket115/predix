# Predix — Decentralized Prediction Market on Solana

> **Live URL:** [https://predix.sxrk.tech](https://predix.sxrk.tech)  
> **Program ID:** `2FoSgViaZXUXL8txXYxc893cUSpPCuvdVZBJ9YDzUKzE`  
> **Network:** Solana Devnet  
> **Collateral Token:** USDC (Devnet)

---

## Table of Contents

1. [Overview](#overview)  
2. [Architecture](#architecture)  
3. [Repositories](#repositories)  
4. [Tech Stack](#tech-stack)  
5. [Frontend — predix-frontend](#frontend--predix-frontend)  
6. [Backend — predix-backend](#backend--predix-backend)  
7. [On-Chain Program — solana-prediction-market-program](#on-chain-program--solana-prediction-market-program)  
8. [Database Schema](#database-schema)  
9. [API Reference](#api-reference)  
10. [Order & Trading Flow](#order--trading-flow)  
11. [Environment Variables](#environment-variables)  
12. [Getting Started](#getting-started)  
13. [Deployment](#deployment)  
14. [Project Structure](#project-structure)  

---

## Overview

Predix is a full-stack, decentralized prediction market platform built on Solana. Users buy and sell **YES/NO outcome shares** on real-world events (politics, crypto, sports, economics, finance). Shares are priced between 0¢ and 100¢ — the market price reflects collective probability.

**Key Design Decisions:**

| Concern | Approach |
|---|---|
| Order Matching | Off-chain (Rust matching engine) for instant execution |
| Settlement | On-chain via Anchor program for trustless finality |
| State Sync | Event Listener reads Solana logs → persists to Postgres |
| Authentication | Privy (Google OAuth + embedded Solana wallets) |
| Token Model | 1 USDC → 1 YES + 1 NO (split/merge mechanism) |

---

## Architecture

```
┌──────────────┐      ┌──────────────────────────────────────┐
│   Frontend   │      │           Backend (Rust)              │
│  (Next.js)   │─────▶│                                      │
│              │ REST │  ┌──────────┐    ┌────────────────┐  │
│  • Markets   │◀─────│  │ Axum API │───▶│ Matching Engine│  │
│  • Trading   │      │  └────┬─────┘    │ (per-market    │  │
│  • Admin     │      │       │          │  Tokio task)   │  │
│  • Profile   │      │       │          └───────┬────────┘  │
└──────────────┘      │       │                  │ trades    │
                      │       ▼                  ▼           │
                      │  ┌─────────┐    ┌──────────────┐     │
                      │  │Postgres │◀───│ Anchor Client │    │
                      │  │   DB    │    │     SDK       │────────▶ Solana
                      │  └────▲────┘    └──────────────┘     │    (Devnet)
                      │       │                              │
                      │  ┌────┴──────────┐                   │
                      │  │Event Listener │◀──── WebSocket ───────── Solana Logs
                      │  └───────────────┘                   │
                      └──────────────────────────────────────┘
```

### Data Flow (High Level)

1. **User authenticates** via Privy (Google OAuth → embedded Solana wallet)
2. **User submits order** → API validates → forwards to Matching Engine
3. **Matching Engine** runs as a dedicated Tokio task per market using `mpsc` + `oneshot` channels
4. **On match** → Backend sends settlement transaction to Solana via Anchor Client SDK
5. **Anchor program** executes on-chain, emits events
6. **Event Listener** subscribes to program logs via WebSocket, persists confirmed state to Postgres
7. **Database** becomes the canonical source of truth — only stores on-chain confirmed data

---

## Repositories

| Repository | Description | Language |
|---|---|---|
| [predix-frontend](https://github.com/aniketsahu115/predix-frontend) | Next.js 16 web application | TypeScript |
| [predix-backend](https://github.com/aniketsahu115/predix-backend) | Rust API, matching engine, event listener | Rust |
| [solana-prediction-market-program](https://github.com/aniketsahu115/solana-prediction-market-program) | Anchor smart contract | Rust |

---

## Tech Stack

### Frontend
| Technology | Purpose |
|---|---|
| Next.js 16 | React framework (App Router) |
| React 19 | UI library |
| TypeScript | Type safety |
| Tailwind CSS 4 | Styling |
| Zustand | Global state management |
| Privy SDK | Authentication & wallet management |
| @solana/web3.js | Solana blockchain interaction |
| Axios | HTTP client |
| Sonner | Toast notifications |
| next-themes | Dark/light mode |

### Backend
| Technology | Purpose |
|---|---|
| Rust (Edition 2024) | Systems programming language |
| Axum 0.8 | Async web framework |
| Tokio | Async runtime (full features) |
| SQLx 0.8 | Async Postgres driver with compile-time queries |
| Anchor Client 0.32 | Solana program interaction |
| tower-http | CORS middleware |
| jsonwebtoken | Privy JWT verification |
| AWS SDK (S3) | DigitalOcean Spaces for metadata storage |
| reqwest | HTTP client for JWKS fetching |

### On-Chain Program
| Technology | Purpose |
|---|---|
| Anchor 0.32 | Solana framework |
| SPL Token | Token standard for YES/NO mints |
| PDA-based accounts | Deterministic account derivation |

### Infrastructure
| Technology | Purpose |
|---|---|
| PostgreSQL | Primary database |
| Docker Compose | Local database provisioning |
| GitHub Actions | CI/CD (SSH deploy to VM) |
| PM2 | Process manager for API + Event Listener |
| DigitalOcean Spaces | S3-compatible object storage for market metadata |

---

## Frontend — predix-frontend

### Pages & Routes

| Route | Description | Auth |
|---|---|---|
| `/` | Home — market listing with category filters | Public |
| `/markets/[id]` | Market detail — chart, orderbook, trading panel | Public (trade requires login) |
| `/profile` | User profile & portfolio | Protected |
| `/admin` | Redirects to `/admin/markets` | Admin only |
| `/admin/markets` | Market management dashboard | Admin only |
| `/admin/markets/create` | Create new market form | Admin only |
| `/admin/markets/[id]` | Market detail (admin view) | Admin only |

### Key Components

| Component | File | Purpose |
|---|---|---|
| `Providers` | `components/provider.tsx` | Privy + Theme providers |
| `RequireAuth` | `components/RequireAuth.tsx` | Auth guard wrapper |
| `UserMenu` | `components/user-menu.tsx` | Wallet info, deposit, theme toggle, logout |
| `ThemeToggle` | `components/theme-toggle.tsx` | Dark/light mode switch |
| `Orderbook` | `components/market/orderbook.tsx` | Live bid/ask depth display |
| `PriceChart` | `components/market/price-chart.tsx` | Price visualization |
| `SplitMergeModal` | `components/market/split-merge-modal.tsx` | Split/merge token UI |
| `UserOpenOrders` | `components/market/user-open-orders.tsx` | Active orders list |
| `UserPositions` | `components/market/user-positions.tsx` | Portfolio positions |
| `UserMarketTabs` | `components/market/user-market-tabs.tsx` | Tab navigation for user data |

### Custom Hooks

| Hook | Purpose |
|---|---|
| `useTrading` | Full trading flow — split, merge, market/limit orders with delegation |
| `useUSDCBalance` | Polls USDC balance with refresh animation |

### State Management (Zustand)

| Store | Purpose |
|---|---|
| `usdcStore` | Global USDC balance with polling & refresh state |
| `useUserStore` | User session data |

### Authentication Flow
1. User clicks "Sign up" / "Log in" → Privy modal opens
2. Login methods: **Google OAuth** or **Solana Wallet**
3. Privy creates an embedded Solana wallet for users without one
4. Identity token is sent as `privy-id-token` header on API requests
5. Admin access is gated by matching email against `NEXT_PUBLIC_ADMIN_EMAIL`

---

## Backend — predix-backend

The backend is a **Rust workspace** with 5 crates:

### Workspace Crates

```
predix-backend/
├── api/              # Axum REST API server (port 3030)
├── matching/         # Order matching engine library
├── event-listener/   # Solana log subscriber (separate binary)
├── anchor-client-sdk/ # Typed SDK for on-chain program interaction
└── db/               # Database models, queries, migrations
```

### Crate: `api`

The main HTTP server built with Axum, listening on `0.0.0.0:3030`.

**Application State (`AppState`):**
- `markets: RwLock<HashMap<u64, mpsc::Sender<EngineMsg>>>` — per-market engine channels
- `rpc_client: Arc<RpcClient>` — Solana RPC connection
- `predix_sdk: Arc<PredixSdk>` — Anchor program client
- `s3: Arc<S3Client>` — DigitalOcean Spaces client
- `db_pool: Arc<PgPool>` — PostgreSQL connection pool

**Authentication:**
- Middleware extracts `privy-id-token` header
- Fetches JWKS from Privy API, validates ES256 JWT
- Extracts `AuthUser` (email, name, solana_address, wallet_id, is_admin)
- Admin middleware checks email against `ADMIN_EMAIL` env var

### Crate: `matching`

In-memory order book engine using `BTreeMap<Decimal, VecDeque<OrderEntry>>`.

**Data Structures:**
- `OrderBook` — bids (highest-first) + asks (lowest-first)
- `MarketBooks` — paired YES + NO order books per market
- `OrderEntry` — id, user_address, market_id, side, price, qty
- `Trade` — buyer_address, seller_address, price, quantity, market_id

**Matching Algorithm:**
- Price-time priority (FIFO within each price level)
- Bid orders match against best asks when `bid_price >= best_ask`
- Ask orders match against best bids when `ask_price <= best_bid`
- Partial fills supported — remainder stays on the book
- Returns: `(order_id, Vec<Trade>, remaining_qty)`

### Crate: `event-listener`

Standalone binary that subscribes to Solana program logs via WebSocket.

**Events Processed:**

| Event | Action |
|---|---|
| `MarketInitialized` | Insert new market into Postgres |
| `MatchExecuted` | Log trade execution |
| `TokensSplit` | Log split operation |
| `TokensMerged` | Log merge operation |
| `RewardsClaimed` | Log reward claim |
| `MarketSettled` | Update market status to `Resolved` in DB |

### Crate: `anchor-client-sdk`

Typed Rust SDK wrapping the on-chain program's instructions:

| Method | On-Chain Instruction | Returns |
|---|---|---|
| `create_market()` | `initialize_market` | `()` (sends tx directly) |
| `place_order()` | `execute_match_multi` | `()` (sends tx directly) |
| `split_order()` | `split_token` | Base64 partially-signed tx |
| `merge_order()` | `merge_tokens` | Base64 partially-signed tx |
| `set_winner()` | `set_winner` | Base64 partially-signed tx |

**PDA Derivation:**
- Market: `["market", market_id.to_le_bytes()]`
- Vault: `["collateral_vault", market_id.to_le_bytes()]`
- YES Mint: `["yes_mint", market_id.to_le_bytes()]`
- NO Mint: `["no_mint", market_id.to_le_bytes()]`

### Crate: `db`

Database access layer using SQLx with PostgreSQL.

**Models:** `Market`, `MarketStatus`, `MarketOutcome`  
**Queries:** `create_market`, `get_market_by_id`, `list_markets_by_status`, `list_all_markets`, `update_market_resolution`

---

## On-Chain Program — solana-prediction-market-program

Anchor program deployed on Solana Devnet.

### Instructions

| Instruction | Description |
|---|---|
| `initialize_market` | Creates Market PDA, collateral vault, YES/NO mints |
| `split_token` | Deposits USDC → mints equal YES + NO tokens |
| `merge_tokens` | Burns YES + NO tokens → returns USDC |
| `execute_match_multi` | Transfers collateral & outcome tokens between traders via delegation |
| `set_winner` | Admin sets market outcome (Yes/No) |
| `claim_reward` | Burns user tokens, pays collateral to winners |

### On-Chain Account Structure

```
Market (PDA)
├── authority: Pubkey        # Admin who created the market
├── metadata_url: String     # Link to market metadata JSON
├── collateral_mint: Pubkey  # USDC mint address
├── collateral_vault: Pubkey # PDA holding locked USDC
├── yes_mint: Pubkey         # YES outcome token mint (PDA)
├── no_mint: Pubkey          # NO outcome token mint (PDA)
├── market_id: u64           # Unique identifier
├── yes_total: u64           # Total YES tokens in circulation
├── no_total: u64            # Total NO tokens in circulation
├── outcome: MarketOutcome   # Yes | No | Undecided
├── expiration_timestamp: i64
├── is_settled: bool
└── bump: u8
```

### Token Economics

```
Split:  1 USDC  ──▶  1 YES + 1 NO    (collateral locked in vault)
Merge:  1 YES + 1 NO  ──▶  1 USDC    (collateral returned from vault)
Trade:  Buyer's USDC  ◀──▶  Seller's outcome tokens  (via delegation)
Claim:  Winning tokens burned  ──▶  USDC paid from vault
```

### Events Emitted

| Event | Fields |
|---|---|
| `MarketInitialized` | market_id, market_pda, authority, collateral_mint, collateral_vault, metadata_url, yes_mint, no_mint, expiration_timestamp |
| `MatchExecuted` | market_id, market_pda, admin, buyer, seller, fills_executed |
| `TokensSplit` | market_id, market_pda, user, amount |
| `TokensMerged` | market_id, market_pda, user, amount |
| `MarketSettled` | market_id, market_pda, outcome |
| `RewardsClaimed` | market_id, market_pda, user, amount |

---

## Database Schema

```sql
-- Custom Types
CREATE TYPE share_type AS ENUM ('yes', 'no');
CREATE TYPE order_status AS ENUM ('partially_filled', 'filled', 'canceled');
CREATE TYPE market_status AS ENUM ('open', 'closed', 'resolved');
CREATE TYPE market_outcome AS ENUM ('yes', 'no', 'not_decided');

-- Users Table
CREATE TABLE users (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name           VARCHAR(50) NOT NULL,
    email          VARCHAR(100) UNIQUE NOT NULL,
    solana_address VARCHAR(44) UNIQUE NOT NULL,
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Markets Table
CREATE TABLE markets (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    market_id     TEXT UNIQUE NOT NULL,
    market_pda    TEXT UNIQUE NOT NULL,
    metadata_url  TEXT NOT NULL,
    yes_mint      TEXT UNIQUE NOT NULL,
    no_mint       TEXT UNIQUE NOT NULL,
    usdc_vault    TEXT UNIQUE NOT NULL,
    status        market_status DEFAULT 'open',
    outcome       market_outcome DEFAULT 'not_decided',
    close_time    TIMESTAMP WITH TIME ZONE NOT NULL,
    resolve_time  TIMESTAMP WITH TIME ZONE,
    title         TEXT NOT NULL,
    description   TEXT,
    category      VARCHAR(50) NOT NULL,
    image_url     TEXT,
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_markets_status ON markets (status);
CREATE INDEX idx_markets_category ON markets (category);
CREATE INDEX idx_markets_close_time ON markets (close_time);
```

---

## API Reference

### Public Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/health-check` | Health check |
| `GET` | `/markets?status=open` | List markets by status |
| `GET` | `/markets/{id}` | Get market details |
| `GET` | `/orderbook/snapshot/{market_id}` | Get orderbook depth |

### Protected Endpoints (require `privy-id-token` header)

| Method | Path | Description |
|---|---|---|
| `POST` | `/markets/delegate` | Approve token delegation for trading |
| `POST` | `/orders/place` | Place market/limit order |
| `POST` | `/orders/split` | Split USDC into YES+NO tokens |
| `POST` | `/orders/merge` | Merge YES+NO tokens into USDC |
| `DELETE` | `/orders/cancel/{order_id}` | Cancel open order |
| `GET` | `/orderbook/open/{market_id}` | Get user's open orders |

### Admin Endpoints (require auth + admin email match)

| Method | Path | Description |
|---|---|---|
| `POST` | `/admin/market/create` | Create new market (on-chain + metadata upload) |
| `POST` | `/admin/market/set-winner` | Resolve market outcome |
| `GET` | `/admin/markets` | List all markets |

---

## Order & Trading Flow

### Buy Flow (Market/Limit Order)

```
1. Frontend calls POST /markets/delegate
   → Backend builds SPL approve instruction (delegate to Market PDA)
   → Returns partially-signed tx
   → User signs & sends via Privy wallet

2. Frontend calls POST /orders/place
   → Backend forwards to Matching Engine (via mpsc channel)
   → Engine returns matches (trades) + remaining qty
   → If trades exist: Backend verifies delegation on-chain
   → Backend calls execute_match_multi on Anchor program
   → On-chain: USDC transferred buyer→seller, tokens transferred seller→buyer
   → Returns order result to frontend
```

### Split Flow

```
1. Frontend calls POST /orders/split { market_id, collateral_mint, amount }
2. Backend builds split_token instruction via Anchor SDK
3. Returns base64 partially-signed transaction
4. Frontend deserializes, user co-signs via Privy, sends to Solana
5. On-chain: USDC deposited to vault, YES+NO tokens minted to user
```

### Merge Flow

```
1. Frontend calls POST /orders/merge { market_id, collateral_mint, amount }
2. Backend builds merge_tokens instruction
3. Returns base64 partially-signed transaction
4. User co-signs and sends
5. On-chain: YES+NO tokens burned, USDC returned from vault
```

---

## Environment Variables

### Frontend (`.env`)

| Variable | Description |
|---|---|
| `NEXT_PUBLIC_PRIVY_APP_ID` | Privy application ID |
| `NEXT_PUBLIC_PRIVY_CLIENT_ID` | Privy client ID |
| `NEXT_PUBLIC_RPC_URL` | Solana RPC endpoint |
| `NEXT_PUBLIC_WS_RPC_URL` | Solana WebSocket RPC endpoint |
| `NEXT_PUBLIC_PRIVY_SIGNER_ID` | Privy signer identifier |
| `NEXT_PUBLIC_PRIVY_SIGNER_PRIVATE_KEY` | Privy signer private key |
| `NEXT_PUBLIC_ADMIN_EMAIL` | Admin email for access control |

### Backend (`.env`)

| Variable | Description |
|---|---|
| `PRIVY_APP_ID` | Privy app ID for JWT verification |
| `PRIVY_VERIFICATION_PEM` | Privy verification public key |
| `SOLANA_RPC_URL` | Solana RPC HTTP endpoint |
| `SOLANA_WS_RPC_URL` | Solana WebSocket endpoint |
| `FEE_PAYER_PRIVATE_KEY` | Backend fee payer keypair (base58) |
| `ADMIN_EMAIL` | Admin email for middleware check |
| `DATABASE_URL` | PostgreSQL connection string |
| `PROGRAM_ID` | Deployed Anchor program ID |
| `DO_SPACES_KEY` | DigitalOcean Spaces access key |
| `DO_SPACES_SECRET` | DigitalOcean Spaces secret key |
| `DO_SPACES_ENDPOINT` | DigitalOcean Spaces endpoint URL |
| `DO_SPACES_REGION` | DigitalOcean Spaces region |
| `DO_SPACES_BUCKET` | DigitalOcean Spaces bucket name |
| `CORS_ALLOW_ORIGIN` | Allowed CORS origin |

---

## Getting Started

### Prerequisites

- **Rust** (latest stable, edition 2024)
- **Node.js** ≥ 18 (or Bun)
- **PostgreSQL** 15+
- **Solana CLI** + **Anchor CLI** 0.32
- **Docker** (optional, for local Postgres)

### 1. Clone All Repositories

```bash
git clone https://github.com/aniketsahu115/predix-frontend
git clone https://github.com/aniketsahu115/predix-backend
git clone https://github.com/aniketsahu115/solana-prediction-market-program
```

### 2. Start PostgreSQL

```bash
cd predix-backend
docker-compose up -d
```

This starts Postgres on `localhost:5432` (user: `postgres`, password: `postgres`).

### 3. Run Database Migrations

```bash
# Install sqlx-cli if needed
cargo install sqlx-cli --no-default-features --features postgres

# Run migrations
cd predix-backend
export DATABASE_URL=postgres://postgres:postgres@localhost:5432/postgres
sqlx migrate run --source db/migrations
```

### 4. Configure Environment

```bash
# Backend
cp predix-backend/.env.example predix-backend/.env
# Fill in all required values

# Frontend
cp predix-frontend/.env.example predix-frontend/.env
# Fill in all required values
```

### 5. Build & Run Backend

```bash
cd predix-backend

# Build all crates
cargo build --release

# Run API server (terminal 1)
cargo run -p api

# Run Event Listener (terminal 2)
cargo run -p event-listener
```

The API starts on `http://localhost:3030`.

### 6. Run Frontend

```bash
cd predix-frontend
bun install   # or npm install
bun run dev   # or npm run dev
```

The frontend starts on `http://localhost:3000`.

### 7. Deploy Anchor Program (Optional)

```bash
cd solana-prediction-market-program
anchor build
anchor deploy --provider.cluster devnet
anchor test
```

---

## Deployment

### Backend (GitHub Actions CI/CD)

The backend uses a GitHub Actions workflow (`.github/workflows/deploy.yml`) that:

1. SSHs into the production VM on push to `main`
2. Pulls latest code
3. Runs `cargo build --release`
4. Restarts services via PM2:
   - `pm2 restart predix-api`
   - `pm2 restart predix-event-listener`

### Frontend

Deployed via Vercel or similar Next.js hosting platform.

---

## Project Structure

```
Predix-Content/
│
├── predix-frontend/                 # Next.js 16 Web Application
│   ├── app/
│   │   ├── layout.tsx               # Root layout (Providers, Toaster)
│   │   ├── page.tsx                 # Home — market listing
│   │   ├── globals.css              # Tailwind + theme variables
│   │   ├── markets/
│   │   │   └── [id]/page.tsx        # Market detail + trading panel
│   │   ├── admin/
│   │   │   ├── layout.tsx           # Admin layout (auth guard)
│   │   │   ├── page.tsx             # Redirect → /admin/markets
│   │   │   └── markets/
│   │   │       ├── page.tsx         # Market management dashboard
│   │   │       ├── create/          # Create market form
│   │   │       └── [id]/            # Admin market detail
│   │   ├── profile/page.tsx         # User profile
│   │   └── utils/axiosInstance.ts   # Axios config
│   ├── components/
│   │   ├── provider.tsx             # Privy + ThemeProvider
│   │   ├── RequireAuth.tsx          # Auth guard
│   │   ├── user-menu.tsx            # User dropdown
│   │   ├── theme-toggle.tsx         # Dark/light toggle
│   │   └── market/
│   │       ├── orderbook.tsx        # Order book display
│   │       ├── price-chart.tsx      # Price chart
│   │       ├── split-merge-modal.tsx
│   │       ├── user-open-orders.tsx
│   │       ├── user-positions.tsx
│   │       └── user-market-tabs.tsx
│   ├── hooks/
│   │   ├── useTrading.ts            # Trading logic hook
│   │   └── useUSDCBalance.ts        # Balance polling hook
│   ├── store/
│   │   ├── usdcStore.ts             # USDC balance store
│   │   └── useUserStore.ts          # User session store
│   └── types/market.ts              # TypeScript interfaces
│
├── predix-backend/                  # Rust Workspace
│   ├── Cargo.toml                   # Workspace manifest
│   ├── docker-compose.yml           # PostgreSQL container
│   ├── .github/workflows/deploy.yml # CI/CD pipeline
│   ├── api/src/
│   │   ├── main.rs                  # Entry point (port 3030)
│   │   ├── app.rs                   # Router + CORS setup
│   │   ├── auth/
│   │   │   ├── auth.rs              # Privy JWT middleware
│   │   │   └── require_admin.rs     # Admin guard middleware
│   │   ├── engine/engine.rs         # Engine message types + runner
│   │   ├── handlers/
│   │   │   ├── admin.rs             # Create/resolve market handlers
│   │   │   ├── market.rs            # Market query handlers
│   │   │   ├── orders.rs            # Place/cancel/split/merge handlers
│   │   │   └── orderbook.rs         # Orderbook snapshot handlers
│   │   ├── routes/                  # Route definitions
│   │   ├── models/                  # Request/response DTOs
│   │   ├── state/state.rs           # AppState definition
│   │   └── utils/
│   │       ├── s3.rs                # DigitalOcean Spaces upload
│   │       └── solana.rs            # Delegation verification
│   ├── matching/src/
│   │   ├── types.rs                 # OrderEntry, Trade, Side, etc.
│   │   └── orderbook/
│   │       ├── orderbook.rs         # BTreeMap-based order book
│   │       └── market.rs            # YES/NO paired order books
│   ├── event-listener/src/
│   │   ├── main.rs                  # WebSocket log subscriber
│   │   └── types.rs                 # Event deserialization structs
│   ├── anchor-client-sdk/src/
│   │   ├── lib.rs                   # PredixSdk (create, place, split, merge, set_winner)
│   │   └── utils.rs                 # PDA derivation helpers
│   └── db/
│       ├── migrations/              # SQL migrations
│       └── src/
│           ├── lib.rs               # Connection pool setup
│           ├── models/market.rs     # Market model + enums
│           └── queries/             # SQL query functions
│
└── solana-prediction-market-program/ # Anchor Program
    ├── Anchor.toml                   # Anchor config (devnet)
    ├── programs/predix-program/src/
    │   ├── lib.rs                    # All instructions + accounts + events
    │   ├── error.rs                  # Custom error codes
    │   └── state/
    │       ├── market.rs             # Market account + MarketOutcome enum
    │       └── position.rs           # Position account (future use)
    └── tests/
        └── predix-program.ts         # Anchor integration tests
```

---

## Detailed Architecture Diagrams

### System Architecture — Full Data Flow

```
                                    ┌─────────────────────────┐
                                    │     User's Browser      │
                                    │  ┌───────────────────┐  │
                                    │  │  Next.js Frontend  │  │
                                    │  │  (React 19 + TS)   │  │
                                    │  └────────┬──────────┘  │
                                    │           │             │
                                    │  ┌────────▼──────────┐  │
                                    │  │  Privy SDK         │  │
                                    │  │  • Auth (Google)   │  │
                                    │  │  • Embedded Wallet │  │
                                    │  │  • Tx Signing      │  │
                                    │  └────────┬──────────┘  │
                                    └───────────┼─────────────┘
                                                │
                        ┌───────────────────────┼───────────────────────┐
                        │  REST API (JSON)      │  Signed Txs (base64) │
                        │  privy-id-token hdr   │                      │
                        ▼                       ▼                      │
           ┌────────────────────────┐  ┌──────────────────┐            │
           │   Axum API Server      │  │  Solana Devnet   │◄───────────┘
           │   (port 3030)          │  │  RPC / WebSocket │
           │                        │  └───────┬──────────┘
           │  ┌──────────────────┐  │          │
           │  │  Auth Middleware  │  │          │ Program Logs
           │  │  (JWT + JWKS)    │  │          │
           │  └────────┬─────────┘  │          │
           │           │            │          │
           │  ┌────────▼─────────┐  │  ┌───────▼──────────┐
           │  │  Route Handlers  │  │  │  Event Listener   │
           │  │  • /markets      │  │  │  (WebSocket sub)  │
           │  │  • /orders       │  │  │                    │
           │  │  • /orderbook    │  │  │  Decodes:          │
           │  │  • /admin        │  │  │  • MarketInit      │
           │  └────────┬─────────┘  │  │  • MatchExecuted   │
           │           │            │  │  • TokensSplit      │
           │  ┌────────▼─────────┐  │  │  • TokensMerged    │
           │  │ Matching Engine  │  │  │  • MarketSettled    │
           │  │ (per-market      │  │  │  • RewardsClaimed   │
           │  │  Tokio task)     │  │  └───────┬────────────┘
           │  │                  │  │          │
           │  │  mpsc channels   │  │          │
           │  │  oneshot replies │  │          │
           │  └────────┬─────────┘  │          │
           │           │ trades     │          │
           │  ┌────────▼─────────┐  │          │
           │  │ Anchor Client    │  │          │
           │  │ SDK              │──────────▶ Solana Program
           │  │ (build + sign    │  │   (2FoSg...zUKzE)
           │  │  transactions)   │  │
           │  └──────────────────┘  │
           └───────────┬────────────┘
                       │
              ┌────────▼────────┐
              │  PostgreSQL DB  │◄──── Event Listener writes
              │                 │      confirmed state
              │  Tables:        │
              │  • users        │
              │  • markets      │
              └─────────────────┘
```

### Matching Engine — Concurrency Model

Each market gets its own isolated Tokio task with an `mpsc` channel. This ensures:
- **Zero lock contention** between markets
- **Sequential processing** within a single market (no race conditions)
- **Instant response** via `oneshot` reply channels

```
                    ┌──────────────────────────────────────────────┐
                    │            AppState.markets                  │
                    │   RwLock<HashMap<u64, mpsc::Sender>>         │
                    └──────┬───────────┬───────────┬──────────────┘
                           │           │           │
                     market_1    market_2    market_N
                           │           │           │
                    ┌──────▼───┐ ┌─────▼────┐ ┌────▼─────┐
                    │  Tokio   │ │  Tokio   │ │  Tokio   │
                    │  Task    │ │  Task    │ │  Task    │
                    │          │ │          │ │          │
                    │ mpsc::Rx │ │ mpsc::Rx │ │ mpsc::Rx │
                    │          │ │          │ │          │
                    │ MarketBk │ │ MarketBk │ │ MarketBk │
                    │ ├─YES OB │ │ ├─YES OB │ │ ├─YES OB │
                    │ └─NO  OB │ │ └─NO  OB │ │ └─NO  OB │
                    └──────────┘ └──────────┘ └──────────┘

    EngineMsg variants:
    ┌─────────────────────────────────────────────────────────┐
    │  PlaceOrder  { side, share, order, resp: oneshot::Tx }  │
    │  CloseOrder  { side, share, price, id, resp }           │
    │  Snapshot    { resp }                                    │
    │  FindOpenOrders { user_address, market_id, resp }       │
    └─────────────────────────────────────────────────────────┘
```

### Order Book Data Structure

```
    OrderBook (per YES or NO outcome)
    ┌──────────────────────────────────────────────────┐
    │  bids: BTreeMap<Decimal, VecDeque<OrderEntry>>   │
    │        (sorted highest → lowest, price-time FIFO)│
    │                                                  │
    │  asks: BTreeMap<Decimal, VecDeque<OrderEntry>>   │
    │        (sorted lowest → highest, price-time FIFO)│
    └──────────────────────────────────────────────────┘

    MarketBooks (per market)
    ┌─────────────┐  ┌─────────────┐
    │  YES Book   │  │  NO Book    │
    │  ├─ bids    │  │  ├─ bids    │
    │  └─ asks    │  │  └─ asks    │
    └─────────────┘  └─────────────┘

    Matching Algorithm (Bid example):
    ┌──────────────────────────────────────────────┐
    │  while order.qty > 0:                        │
    │    best_ask = asks.keys().next()              │
    │    if order.price < best_ask → STOP           │
    │    queue = asks[best_ask]                     │
    │    maker = queue.front()                      │
    │    take = min(order.qty, maker.qty)            │
    │    record Trade(buyer, seller, price, take)   │
    │    if maker.qty == 0 → pop maker              │
    │    if queue empty → remove price level         │
    │  if order.qty > 0 → insert into bids          │
    └──────────────────────────────────────────────┘
```

### Solana PDA Derivation Map

All on-chain accounts are deterministically derived from the `market_id`:

```
    market_id: u64 (from UUID bytes)
         │
         ├──▶ Market PDA
         │    seeds: ["market", market_id.to_le_bytes()]
         │    Stores: authority, mints, vault, outcome, expiration
         │
         ├──▶ Collateral Vault PDA
         │    seeds: ["collateral_vault", market_id.to_le_bytes()]
         │    SPL Token Account holding locked USDC
         │
         ├──▶ YES Mint PDA
         │    seeds: ["yes_mint", market_id.to_le_bytes()]
         │    Mint authority = Market PDA
         │    Decimals = 6
         │
         └──▶ NO Mint PDA
              seeds: ["no_mint", market_id.to_le_bytes()]
              Mint authority = Market PDA
              Decimals = 6

    User Accounts (derived per user per market):
         │
         ├──▶ User Collateral ATA
         │    = getAssociatedTokenAddress(user_wallet, USDC_mint)
         │
         ├──▶ User YES ATA
         │    = getAssociatedTokenAddress(user_wallet, yes_mint_pda)
         │
         └──▶ User NO ATA
              = getAssociatedTokenAddress(user_wallet, no_mint_pda)
```

### Delegation & Settlement Flow (execute_match_multi)

The trading flow uses a **delegation pattern** where users approve the Market PDA to transfer tokens on their behalf:

```
    ┌─────────┐         ┌──────────┐         ┌──────────────┐
    │  Buyer  │         │ Backend  │         │ Solana Chain │
    └────┬────┘         └────┬─────┘         └──────┬───────┘
         │                   │                      │
    1.   │── POST /delegate ─▶│                      │
         │                   │  Build approve_checked ix
         │                   │  (delegate USDC to Market PDA)
         │◀── partial tx ───│                      │
         │                   │                      │
    2.   │── co-sign + send ──────────────────────▶│
         │                   │                      │  SPL approve
         │                   │                      │
    3.   │── POST /orders/place ▶│                  │
         │                   │                      │
         │                   │  Matching Engine:     │
         │                   │  finds matching ask   │
         │                   │                      │
    4.   │                   │── execute_match_multi ▶│
         │                   │   (remaining_accounts: │
         │                   │    buyer_collateral,   │
         │                   │    seller_collateral,   │
         │                   │    buyer_outcome_ata,   │
         │                   │    seller_outcome_ata,  │
         │                   │    buyer_owner,         │
         │                   │    seller_owner)        │
         │                   │                      │
         │                   │                      │  Verify delegations
         │                   │                      │  Transfer USDC buyer→seller
         │                   │                      │  Transfer tokens seller→buyer
         │                   │                      │  Emit MatchExecuted event
         │                   │                      │
    5.   │                   │◀── tx confirmed ─────│
         │◀── order result ──│                      │
         │                   │                      │
```

Each fill in `execute_match_multi` requires **6 remaining accounts**:
1. Buyer's collateral ATA (USDC)
2. Seller's collateral ATA (USDC)
3. Buyer's outcome token ATA (YES or NO)
4. Seller's outcome token ATA (YES or NO)
5. Buyer's wallet (owner)
6. Seller's wallet (owner)

---

## Market Lifecycle

```
    ┌──────────────────────────────────────────────────────────────┐
    │                      MARKET LIFECYCLE                        │
    └──────────────────────────────────────────────────────────────┘

    1. CREATION (Admin)
    ┌─────────────────────────────────────────────────────────────┐
    │  Admin fills form → POST /admin/market/create               │
    │  Backend:                                                    │
    │    • Generates market_id from UUID (first 8 bytes → u64)    │
    │    • Uploads metadata JSON to DigitalOcean Spaces           │
    │    • Calls initialize_market on Anchor program               │
    │    • On-chain: creates Market PDA, vault, YES/NO mints      │
    │    • Emits MarketInitialized event                           │
    │  Event Listener:                                             │
    │    • Catches event → fetches metadata from URL               │
    │    • Inserts market into Postgres (status: open)             │
    │  Backend:                                                    │
    │    • Spawns Tokio task with mpsc channel for matching engine │
    └─────────────────────────────────────────────────────────────┘
                                │
                                ▼
    2. TRADING (Users)
    ┌─────────────────────────────────────────────────────────────┐
    │  Users can:                                                  │
    │    • Split: 1 USDC → 1 YES + 1 NO (minted on-chain)        │
    │    • Merge: 1 YES + 1 NO → 1 USDC (burned on-chain)        │
    │    • Buy/Sell: Place limit/market orders on YES or NO books  │
    │                                                              │
    │  Order flow:                                                 │
    │    delegate → match off-chain → settle on-chain              │
    │                                                              │
    │  Market continues until expiration_timestamp is reached      │
    └─────────────────────────────────────────────────────────────┘
                                │
                                ▼
    3. RESOLUTION (Admin)
    ┌─────────────────────────────────────────────────────────────┐
    │  Admin determines real-world outcome                         │
    │  POST /admin/market/set-winner { market_id, outcome }       │
    │  Backend:                                                    │
    │    • Calls set_winner on Anchor program                     │
    │    • On-chain: sets is_settled=true, outcome=Yes|No          │
    │    • Emits MarketSettled event                               │
    │  Event Listener:                                             │
    │    • Updates market in DB (status: resolved, outcome: yes/no)│
    └─────────────────────────────────────────────────────────────┘
                                │
                                ▼
    4. CLAIMING (Users)
    ┌─────────────────────────────────────────────────────────────┐
    │  Winning token holders call claim_reward                     │
    │  On-chain:                                                   │
    │    • Burns ALL user's YES and NO tokens                      │
    │    • Pays USDC from vault for winning tokens only            │
    │    • Example: User holds 100 YES + 50 NO, outcome = Yes     │
    │      → Burns 100 YES + 50 NO                                │
    │      → Pays 100 USDC from vault                             │
    │  Emits RewardsClaimed event                                  │
    └─────────────────────────────────────────────────────────────┘
```

---

## Security Model

### Authentication & Authorization

| Layer | Mechanism | Details |
|---|---|---|
| Frontend Auth | Privy SDK | Google OAuth or Solana wallet connect |
| API Auth | JWT Middleware | Validates `privy-id-token` via JWKS (ES256) from Privy servers |
| Admin Guard | Email check | Compares authenticated email against `ADMIN_EMAIL` env var |
| On-chain Auth | Signer constraints | `admin` must be market authority for `set_winner` |
| Token Safety | SPL Delegation | Market PDA acts as delegate — users approve specific amounts |

### On-Chain Safety Guards

| Check | Instruction | Error |
|---|---|---|
| Market not settled | `split_token`, `merge_tokens`, `execute_match_multi` | `MarketAlreadySettled` |
| Market not expired | `split_token`, `merge_tokens`, `execute_match_multi` | `MarketExpired` |
| Market is settled | `claim_reward` | `MarketNotSettled` |
| Correct collateral mint | `execute_match_multi` | `InvalidCollateralAccount` |
| Correct token mint | `execute_match_multi` | `InvalidTokenAccount` |
| Owner matches signer | `execute_match_multi` | `InvalidOwner` |
| Sufficient delegation | `execute_match_multi` | `InvalidDelegate` |
| Valid expiration | `initialize_market` | `InvalidExpirationTimestamp` |
| Amount > 0 | All token operations | `InvalidAmount` |
| Math overflow | All arithmetic | `MathOverflow` |

### Partially-Signed Transactions

For operations requiring both admin and user signatures (split, merge, set_winner):
1. Backend constructs the instruction
2. Backend partially signs with fee payer keypair
3. Serializes to base64 and returns to frontend
4. Frontend deserializes, user co-signs via Privy embedded wallet
5. Transaction sent to Solana with both signatures

---

## On-Chain Error Codes Reference

| Error Code | Message | Context |
|---|---|---|
| `InvalidVault` | Vault balance cannot be zero | Vault validation |
| `InvalidMarket` | Invalid market | Market ID mismatch |
| `InvalidSettlementDeadline` | Invalid settlement deadline | Deadline validation |
| `MarketAlreadySettled` | Market already settled | Attempting action on settled market |
| `MarketExpired` | Market has expired | Trading after expiration |
| `InvalidAmount` | Invalid amount | Zero or negative amounts |
| `MathOverflow` | Math overflow | Arithmetic overflow in token totals |
| `InvalidTokenAccount` | Invalid token account | Wrong mint or missing delegation |
| `InvalidMarketAuthority` | Invalid admin | Non-authority calling admin functions |
| `MarketNotSettled` | Market is not settled | Claiming before resolution |
| `InvalidBuyer` | Invalid buyer | Buyer validation failure |
| `InvalidSeller` | Invalid seller | Seller validation failure |
| `InvalidTradeSide` | Invalid trade side | Invalid YES/NO specification |
| `BuyerInsufficientBalance` | Buyer does not have enough balance | Balance check failure |
| `InvalidExpirationTimestamp` | Invalid expiration timestamp | Past timestamp on creation |
| `InvalidCollateralAccount` | Invalid collateral account | Wrong collateral mint |
| `InvalidOwner` | Invalid owner | ATA owner ≠ signer |
| `InvalidAccounts` | Invalid accounts | Wrong number of remaining accounts |
| `InvalidDelegate` | Invalid delegate | Insufficient delegated amount |

---

## Market Metadata Storage

Market metadata (title, description, category, image) is stored off-chain in **DigitalOcean Spaces** (S3-compatible):

```
Upload Path: market_metadata/{market_id}.json
Access:      Public Read (ObjectCannedAcl::PublicRead)
Content:     application/json

{
  "title": "Will Bitcoin reach $100k by 2026?",
  "description": "Resolves YES if BTC/USD exceeds $100,000...",
  "category": "Crypto",
  "image_url": "https://..."
}
```

The `metadata_url` is stored on-chain in the Market PDA. When the Event Listener catches a `MarketInitialized` event, it fetches this URL to hydrate the Postgres record with title, description, category, and image_url.

---

## Balance & Wallet Management

### USDC Balance Polling (Frontend)

The frontend uses a Zustand store (`usdcStore`) with a multi-tier polling strategy:

```
    Initial Load:
      useUSDCBalance hook → fetchBalance() immediately

    Background Polling:
      setInterval(fetchBalance, 5000)  ← every 5 seconds

    Post-Trade Refresh (animated):
      startRefreshing() triggers:
        t=0s    → fetch + enable pulse animation
        t=2s    → fetch
        t=4s    → fetch
        t=6s    → fetch
        t=8s    → fetch
        t=10s   → fetch + disable animation
```

### USDC Mint (Devnet)

```
Address: 4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU
Decimals: 6
Network: Solana Devnet
```

All amounts are converted: `user_amount * 1_000_000 = on_chain_amount`

---

## Position Account (Future Use)

The on-chain program includes a `Position` account structure for future portfolio tracking:

```rust
pub struct Position {
    pub market_id: Pubkey,    // Market reference
    pub owner: Pubkey,        // User wallet
    pub yes_share: u64,       // YES tokens held
    pub no_share: u64,        // NO tokens held
    pub total: u64,           // Total collateral committed
    pub claimed: bool,        // Whether rewards were claimed
    pub bump: u8,             // PDA bump
}
```

This is defined but not yet integrated into the instruction set — designed for on-chain position tracking in a future release.

---

## Market Categories

The platform supports the following market categories:

| Category | Examples |
|---|---|
| **Politics** | Election outcomes, policy decisions |
| **Crypto** | Price targets, protocol milestones |
| **Sports** | Match results, tournament winners |
| **Economics** | GDP, inflation, employment data |
| **Finance** | Stock prices, IPOs, market indices |

Categories are stored in the `markets` table and used for frontend filtering.

---

## CI/CD Pipeline

### Backend Deployment (GitHub Actions)

```yaml
# Trigger: push to main branch
# Runner: ubuntu-latest
# Strategy: SSH into production VM

Steps:
  1. SSH into VM (appleboy/ssh-action)
  2. cd /root/Predix-backend
  3. source Rust toolchain ($HOME/.cargo/env)
  4. source Node/PM2 (NVM)
  5. git pull origin main
  6. cargo build --release
  7. pm2 restart predix-api
  8. pm2 restart predix-event-listener
  9. pm2 save
```

### Frontend Deployment

The Next.js frontend is deployable via **Vercel** with zero configuration:
- Build command: `next build --webpack`
- Output: Static + SSR pages
- Environment variables configured in Vercel dashboard

---

## API Request/Response Schemas

### Place Order

```
POST /orders/place
Authorization: privy-id-token header

Request:
{
  "market_id": "12345678901234567",    // u64 as string
  "collateral_mint": "4zMMC9...ncDU",  // USDC mint address
  "side": "Bid" | "Ask",               // Buy or Sell
  "share": "Yes" | "No",               // Outcome token
  "price": "0.65",                      // Decimal (0.001–0.999)
  "qty": "100"                          // Decimal shares
}

Response (200):
{
  "order_id": "550e8400-e29b-41d4-a716-446655440000",
  "trades": [
    {
      "buyer_address": "ABC...xyz",
      "seller_address": "DEF...uvw",
      "price": "0.65",
      "quantity": "50",
      "market_id": 12345678901234567
    }
  ],
  "remaining_qty": "50",
  "message": "Order placed successfully"
}
```

### Cancel Order

```
DELETE /orders/cancel

Request:
{
  "market_id": "12345678901234567",
  "order_id": "550e8400-e29b-41d4-a716-446655440000",
  "side": "Bid" | "Ask",
  "share": "Yes" | "No",
  "price": "0.65"
}

Response (200):
{
  "success": true,
  "message": "Order cancelled successfully"
}
```

### Split Tokens

```
POST /orders/split
Authorization: privy-id-token header

Request:
{
  "market_id": "12345678901234567",
  "collateral_mint": "4zMMC9...ncDU",
  "amount": 1000000                    // Raw u64 (6 decimals → 1 USDC)
}

Response (200):
{
  "tx_message": "base64_encoded_partially_signed_transaction",
  "message": "Split order instruction created successfully"
}
```

### Merge Tokens

```
POST /orders/merge
Authorization: privy-id-token header

Request:
{
  "market_id": "12345678901234567",
  "collateral_mint": "4zMMC9...ncDU",
  "amount": 1000000
}

Response (200):
{
  "tx_message": "base64_encoded_partially_signed_transaction",
  "message": "Merge order instruction created successfully"
}
```

### Delegate Approval

```
POST /markets/delegate
Authorization: privy-id-token header

Request:
{
  "market_id": "12345678901234567",
  "side": "Bid" | "Ask",
  "share": "Yes" | "No",
  "amount": 1000000,                   // Raw token amount to approve
  "decimals": 6
}

Response (200):
{
  "tx_message": "base64_encoded_partially_signed_transaction",
  "recent_blockhash": "GHij...klmn"
}
```

### Create Market (Admin)

```
POST /admin/market/create
Authorization: privy-id-token header (admin only)

Request:
{
  "metadata": {
    "title": "Will Bitcoin reach $100k by 2026?",
    "description": "Resolves YES if BTC/USD spot price...",
    "category": "Crypto",
    "image_url": "https://example.com/btc.png"
  },
  "collateral_mint": "4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU",
  "expiration_timestamp": 1735689600
}

Response (200):
{
  "market_id": 12345678901234567,
  "message": "Market created successfully"
}
```

### Resolve Market (Admin)

```
POST /admin/market/set-winner
Authorization: privy-id-token header (admin only)

Request:
{
  "market_id": "12345678901234567",
  "outcome": "yes" | "no" | "not_decided"
}

Response (200):
{
  "tx_message": "base64_encoded_partially_signed_transaction",
  "message": "Market resolved successfully"
}
```

### Orderbook Snapshot

```
GET /orderbook/snapshot/{market_id}
No auth required

Response (200):
{
  "yes": {
    "bids": [
      { "price": "0.78", "quantity": "250" },
      { "price": "0.75", "quantity": "100" }
    ],
    "asks": [
      { "price": "0.80", "quantity": "150" },
      { "price": "0.85", "quantity": "300" }
    ]
  },
  "no": {
    "bids": [...],
    "asks": [...]
  }
}
```

### User Open Orders

```
GET /orderbook/open/{market_id}
Authorization: privy-id-token header

Response (200):
[
  {
    "id": "550e8400-e29b-...",
    "outcome": "Yes",
    "side": "Bid",
    "price": "0.65",
    "quantity": "100"
  }
]
```

---

## Admin Dashboard

The admin panel (`/admin/*`) provides a complete market management interface:

### Admin Access Control
- Protected by `AdminLayout` which checks `user.linkedAccounts` for a Google OAuth email
- Email is compared against `NEXT_PUBLIC_ADMIN_EMAIL` environment variable
- Non-admin users are redirected to the homepage

### Admin Features

| Page | Route | Capabilities |
|---|---|---|
| **Market List** | `/admin/markets` | View all markets in a table (title, status badge, end date, image) |
| **Create Market** | `/admin/markets/create` | Form with title, resolution rules, image URL, category selector, end date |
| **Market Detail** | `/admin/markets/[id]` | View individual market, resolve outcome |

### Create Market Form Fields

| Field | Type | Required | Description |
|---|---|---|---|
| Question / Title | Text input | ✅ | The prediction question (e.g., "Will Bitcoin hit $100k?") |
| Resolution Rules | Textarea | ❌ | Detailed criteria for how the market resolves |
| Image URL | URL input | ❌ | Cover image for the market card |
| Category | Dropdown | ✅ | Politics, Crypto, Sports, Economics, Finance |
| End Date | Date picker | ✅ | Market expiration date (converted to Unix timestamp) |

### Market Creation Flow (Admin)

```
    Admin fills form
         │
         ▼
    Frontend converts date → Unix timestamp
         │
         ▼
    POST /admin/market/create
    {
      metadata: { title, description, category, image_url },
      collateral_mint: "4zMMC9...ncDU",
      expiration_timestamp: 1735689600
    }
         │
         ▼
    Backend:
    ├─ Generate market_id: UUID → first 8 bytes → u64
    ├─ Upload metadata JSON to DigitalOcean Spaces
    │   → https://{bucket}.{region}.digitaloceanspaces.com/...
    ├─ Call initialize_market on Anchor program
    │   → Creates Market PDA, Vault PDA, YES Mint, NO Mint
    ├─ Spawn Tokio matching engine task (mpsc channel)
    └─ Return { market_id, message }
         │
         ▼
    Event Listener picks up MarketInitialized event
    ├─ Fetches metadata from URL
    └─ Inserts into Postgres (status: open)
```

---

## Profile & Wallet Management

The profile page (`/profile`) provides wallet management for authenticated users:

### Features

| Feature | Description |
|---|---|
| **Balance Display** | Real-time USDC balance with pulse animation during refresh |
| **Deposit** | Opens Privy fund wallet modal (manual deposit flow for Devnet) |
| **Send USDC** | P2P transfer to any Solana address |
| **Export Private Key** | Exports embedded wallet private key via Privy SDK |
| **Logout** | Clears session and redirects to homepage |

### Send USDC Flow

```
    User enters recipient address + amount
         │
         ▼
    Frontend builds SPL Transfer instruction:
    ├─ Get sender ATA (getAssociatedTokenAddress)
    ├─ Get recipient ATA (getAssociatedTokenAddress)
    ├─ createTransferInstruction(sender_ata, recipient_ata, user, amount * 1e6)
         │
         ▼
    Transaction signed via Privy signAndSendTransaction
         │
         ▼
    Balance auto-refreshes via startRefreshing() polling
```

### User Menu (Dropdown)

The `UserMenu` component provides a dropdown accessible from any page:

| Action | Visibility | Description |
|---|---|---|
| **Deposit** | All users | Opens Privy fund wallet modal |
| **Profile** | All users | Navigates to `/profile` |
| **Admin Dashboard** | Admin only | Navigates to `/admin/markets` |
| **Theme Toggle** | All users | Switch dark/light mode |
| **Log Out** | All users | Ends session |

---

## Authentication Deep Dive

### JWT Verification Flow (Backend)

```
    1. Extract `privy-id-token` from request header

    2. Decode JWT header → get `kid` (key ID)

    3. Fetch JWKS from Privy:
       GET https://auth.privy.io/api/v1/apps/{PRIVY_APP_ID}/jwks
       → Returns array of JWK keys (ES256 / P-256 curve)

    4. Find matching key by `kid`
       → Extract x, y coordinates from JWK
       → Construct ES256 DecodingKey

    5. Validate JWT:
       ├─ Algorithm: ES256
       ├─ Issuer: "privy.io"
       ├─ Audience: PRIVY_APP_ID
       └─ Expiration: current time < exp

    6. Parse claims:
       ├─ sub: Privy user ID (e.g., "did:privy:...")
       ├─ linked_accounts: JSON array of linked accounts
       └─ custom_metadata: optional

    7. Extract AuthUser from linked_accounts:
       ├─ Find account where type = "wallet" AND chain_type = "solana"
       │   → solana_address
       ├─ Find account where type = "google_oauth"
       │   → email, name
       └─ Check if email == ADMIN_EMAIL → is_admin

    8. Inject AuthUser into request extensions
```

### Privy Claims Structure

```json
{
  "sub": "did:privy:abc123...",
  "iss": "privy.io",
  "aud": "your-privy-app-id",
  "exp": 1735689600,
  "iat": 1735603200,
  "linked_accounts": [
    {
      "type": "wallet",
      "address": "ABC...xyz",
      "chain_type": "solana",
      "wallet_client_type": "privy"
    },
    {
      "type": "google_oauth",
      "email": "user@example.com",
      "name": "John Doe",
      "subject": "google-oauth2|123..."
    }
  ]
}
```

---

## Anchor Test Suite

The on-chain program includes a comprehensive integration test suite (`tests/predix-program.ts`) using **Mocha + Chai + Anchor**:

### Test Setup

- Creates 3 test keypairs: `admin`, `user`, `user2`
- Airdrops SOL to all accounts
- Creates a collateral mint (6 decimals) controlled by `admin`
- Derives all PDAs (Market, Vault, YES Mint, NO Mint) from `marketId = 1`

### Test Scenarios

| Test Group | Test Case | Assertions |
|---|---|---|
| **Initialize Market** | ✅ Creates market with valid params | Event emitted, Market PDA populated correctly, `is_settled = false` |
| | ❌ Past expiration timestamp | Reverts with `InvalidExpirationTimestamp` |
| **Split Token** | ✅ Split 2 USDC → 2 YES + 2 NO | Vault receives 2 USDC, user gets equal YES/NO, totals updated |
| | ❌ Zero amount | Reverts with `InvalidAmount` |
| **Merge Token** | ✅ Merge 1 YES + 1 NO → 1 USDC | Vault reduced by 1, user gets 1 USDC back, totals updated |
| **Execute Match** | ✅ Two fills (0.5@0.6 + 0.5@0.4) | Buyer gets 1 YES, seller gets 1 USDC, delegation consumed |
| **Settle Market** | ✅ Set winner to YES | `is_settled = true`, `outcome = Yes`, event emitted |
| | ❌ Re-settle already settled | Reverts with `MarketAlreadySettled` |
| **Redeem Winnings** | ✅ YES holder claims after YES wins | YES tokens burned, USDC paid from vault, totals updated |

### Running Tests

```bash
cd solana-prediction-market-program
anchor test
```

### Test Execution Flow

```
    before():
      Airdrop SOL → Create collateral mint → Derive all PDAs

    Initialize Market:
      admin signs initializeMarket → verify event + on-chain state

    Split Token:
      mint collateral to user → approve to Market PDA → splitToken
      → verify YES/NO balances + vault + market totals

    Merge Token:
      mergeTokens → verify USDC returned + tokens burned + totals

    Execute Match:
      mint collateral to buyer → approve buyer USDC + seller YES
      → executeMatchMulti with 2 fills → verify all balances

    Settle Market:
      admin calls setWinner(Yes) → verify is_settled + outcome

    Redeem Winnings:
      YES holder calls claimReward → verify tokens burned + USDC paid
```

---

## Frontend UX Patterns

### Trading Panel UI

The market detail page (`/markets/[id]`) features a sophisticated trading panel:

```
    ┌──────────────────────────────────┐
    │  [Buy]  [Sell]     Market Order ▼│  ← Action + Order Type
    ├──────────────────────────────────┤
    │  ┌──────────┐  ┌──────────┐     │
    │  │   Yes    │  │    No    │     │  ← Outcome Selection
    │  │   78¢    │  │   22¢    │     │     (highlighted border)
    │  └──────────┘  └──────────┘     │
    ├──────────────────────────────────┤
    │  Limit Price      [- 50¢ +]     │  ← Only for limit orders
    ├──────────────────────────────────┤
    │  Amount                         │
    │  ┌──────────────────────┐       │
    │  │ 0                USDC│       │  ← Quantity input
    │  └──────────────────────┘       │
    │  Available: 1234.56 USDC        │
    ├──────────────────────────────────┤
    │  Average price         0.78¢    │
    │  Estimated cost       $100.00   │  ← Calculated stats
    │  Payout if Yes        $128.21   │
    ├──────────────────────────────────┤
    │  ┌──────────────────────────┐   │
    │  │       Buy Yes            │   │  ← Action button
    │  └──────────────────────────┘   │     (green/red based on action)
    └──────────────────────────────────┘
```

### Order Types

| Type | Description | UI Behavior |
|---|---|---|
| **Market Order** | Execute at best available price | Amount in USDC, no price input |
| **Limit Order** | Set specific price (0.1¢ – 99.9¢) | Stepper input with ±1¢ buttons |
| **Split Order** | Convert USDC → YES + NO tokens | Opens `SplitMergeModal` |
| **Merge Order** | Convert YES + NO → USDC | Opens `SplitMergeModal` |

### Responsive Design

- **Desktop (lg+)**: 12-column grid — 8 cols for chart/orderbook, 4 cols for sticky trading panel
- **Mobile**: Single column — content stacked vertically
- **Header**: Sticky with backdrop blur (`backdrop-blur-md`)
- **Trading Panel**: Sticky positioned at `top: 96px` on desktop

### Theme System

- Uses `next-themes` with class-based dark mode (`.dark`)
- CSS variables for core colors:
  - Light: `--background: #ffffff`, `--foreground: #171717`
  - Dark: `--background: #09090b` (Zinc 950), `--foreground: #ededed`
- Brand color: `#07C285` (Predix Green)
- Tailwind CSS 4 with `@import "tailwindcss"` and `@theme inline`

---

## HTTP Client Configuration

The frontend uses a centralized Axios instance:

```typescript
const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_BASE_URL,
  headers: { "Content-Type": "application/json" },
});
```

All API calls pass the `privy-id-token` header for authentication:
```typescript
const { data } = await api.post("/orders/place", body, {
  headers: { "privy-id-token": identityToken },
});
```

---

## Glossary

| Term | Definition |
|---|---|
| **ATA** | Associated Token Account — deterministic SPL token account per wallet per mint |
| **Bid** | Buy order — wanting to purchase outcome shares |
| **Ask** | Sell order — wanting to sell outcome shares |
| **Collateral** | USDC used as the base currency for all market operations |
| **Delegation** | SPL `approve_checked` — allows a PDA to transfer tokens on behalf of a user |
| **Market PDA** | Program Derived Address storing market state on-chain |
| **Matching Engine** | Off-chain service that matches buy/sell orders using price-time priority |
| **Outcome Tokens** | YES and NO SPL tokens representing market positions |
| **Partially-Signed Tx** | Transaction signed by backend fee payer, awaiting user co-signature |
| **PDA** | Program Derived Address — deterministic Solana account owned by a program |
| **Settlement** | On-chain execution of matched trades via `execute_match_multi` |
| **Split** | Converting collateral (USDC) into equal YES + NO outcome tokens |
| **Merge** | Converting equal YES + NO tokens back into collateral (USDC) |
| **Vault** | PDA-owned token account holding locked collateral for a market |
| **JWKS** | JSON Web Key Set — public keys used to verify Privy JWTs |
| **Fee Payer** | Backend-controlled keypair that pays Solana transaction fees |
| **Remaining Accounts** | Anchor pattern for passing variable-length account lists to instructions |
| **Event Listener** | Standalone service subscribing to Solana program logs via WebSocket |

---

## Known Limitations & Future Work

| Area | Current State | Planned |
|---|---|---|
| **Order History** | `TODO` in `routes/markets.rs` and `routes/orderbook.rs` | Persisted trade history with pagination |
| **Positions Tracking** | Calculated from on-chain token balances | On-chain `Position` account (struct exists) |
| **Market Order Pricing** | Uses hardcoded reference prices (78¢/22¢) | Dynamic pricing from orderbook midpoint |
| **Multi-Outcome Markets** | Binary only (YES/NO) | Multi-outcome categorical markets |
| **Admin Authorization** | Email-based check | On-chain authority verification |
| **Rate Limiting** | None | Request throttling per user/IP |
| **WebSocket Updates** | Frontend polls every 5s | Real-time WebSocket push for orderbook & balance |
| **Mainnet Deployment** | Devnet only | Mainnet with audited contracts |

---

## License

This project is maintained by the Predix team. See individual repository licenses for details.
