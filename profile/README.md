# EuStellar-Pay

**Real-time payroll infrastructure on Stellar and Soroban.**

EuPay streams salaries per-second to workers anywhere in the world. Employers deposit funds into a non-custodial Soroban vault, create streams with a per-second rate, and workers claim earnings whenever they choose. No platform fees. No custodians. Fully on-chain.

---

## The problem

Traditional payroll settles weekly or monthly. Workers in high-inflation economies, gig workers, and contractors bear all the timing risk — they deliver work up front and wait weeks to be paid. Cross-border payroll adds FX fees, correspondent banking delays, and compliance overhead on top of that.

Soroban makes a better model possible: a streaming contract that accrues earnings second-by-second and lets workers claim at any time. Stellar's 5-second finality and near-zero fees make it practical at any scale.

---

## How it works

```
Employer deposits XLM/USDC into treasury_vault contract
  ↓
Employer creates a stream: worker address + rate per second + duration
  ↓
payroll_stream contract tracks elapsed active time per stream
  ↓
Worker calls claimable_amount() — reads elapsed × rate − already claimed
  ↓
Worker submits a claim transaction — tokens transfer immediately on-chain
  ↓
Employer can pause, resume, cancel, or top up the stream at any time
```

Pause/resume accurately excludes paused time from accrual. Cancel settles remaining unearned funds back to the employer. The treasury vault tracks per-stream allocations so funds cannot be double-spent across streams.

---

## Contracts

Both contracts are deployed on Stellar Testnet (Soroban).

### `payroll_stream`

The core streaming contract. Tracks active time with pause-aware arithmetic.

| Function | Description |
|---|---|
| `create_stream` | Opens a stream with a worker address, token, rate/s, duration, initial deposit |
| `fund_stream` | Adds more tokens to extend a stream's runway |
| `claim` | Transfers claimable earnings to the worker; auto-completes at end time |
| `pause_stream` | Records `paused_at` timestamp; accrual stops |
| `resume_stream` | Computes pause gap and adds to `pause_duration`; accrual resumes |
| `cancel_stream` | Settles claimable to worker, returns remainder to employer |
| `admin_cancel` | Emergency cancel by admin |
| `claimable_amount` | View: elapsed active seconds × rate − already claimed, capped at remaining |
| `active_seconds` | View: total non-paused seconds elapsed |
| `remaining_funds` | View: initial deposit − total claimed |

Status states: `Active → Paused → Active → Completed / Cancelled`

### `treasury_vault`

Multi-token vault with per-stream allocation accounting.

| Function | Description |
|---|---|
| `deposit` | Employer deposits any Stellar token |
| `withdraw` | Admin withdraws unallocated funds |
| `allocate` | Locks funds against a specific stream ID |
| `release_alloc` | Returns allocation to free balance when a stream completes |
| `reclaim_alloc` | Admin reclaims orphaned allocations |
| `emergency_drain` | Admin drains full token balance to themselves |
| `balance` | View: free token balance |
| `stream_allocation` | View: how much is locked per stream × token pair |

Solvency guarantee: `allocated + free = total_deposited` at all times.

---

## Repositories

| Repo | Stack | Description |
|---|---|---|
| [EuPay-smartcontract](https://github.com/EuStellar-Pay/EuPay-smartcontract) | Rust · Soroban SDK | `payroll_stream` and `treasury_vault` contracts with full test suites |
| [EuPay-backend](https://github.com/EuStellar-Pay/EuPay-backend) | TypeScript · Express · stellar-sdk | REST API: contract reads via simulation, unsigned XDR builds for writes, Ed25519 challenge-response auth, Soroban event indexer, rate limiting, health checks |
| [EuPay-frontend](https://github.com/EuStellar-Pay/EuPay-frontend) | React · TypeScript · Tailwind | Freighter wallet integration, employer and worker dashboards, create-stream and deposit modals, real-time claim flow, on-chain event log |
| [EuPay-mobile](https://github.com/EuStellar-Pay/EuPay-mobile) | React Native · Expo | Mobile companion for workers to monitor streams and trigger claims |

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    EuPay Frontend                    │
│   React + Tailwind + @stellar/freighter-api          │
│                                                      │
│  Employer Dashboard      Worker Dashboard            │
│  ├ Create stream         ├ Track streams by ID       │
│  ├ Deposit to vault      ├ Claimable amount (live)   │
│  ├ Pause / Resume        ├ One-click claim           │
│  ├ Cancel / Fund         └ On-chain event log        │
│  └ Event log per stream                              │
└────────────────┬───────────────────────────────┬─────┘
                 │ REST (unsigned XDR)            │ reads
                 ▼                                ▼
┌─────────────────────────────┐    ┌──────────────────────────┐
│       EuPay Backend         │    │   Soroban RPC (Testnet)   │
│  Express · TypeScript       │    │   simulation reads        │
│                             │    │   getEvents               │
│  /api/auth     Ed25519 auth │    └──────────────────────────┘
│  /api/streams  CRUD + XDR   │                 │
│  /api/workers  claim XDR    │                 │ on-chain
│  /api/employers vault XDR   │                 ▼
│  /api/**/events paginated   │    ┌──────────────────────────┐
│  /health/deep  RPC + contr. │    │    Soroban Contracts      │
└─────────────────────────────┘    │                          │
                                   │  payroll_stream          │
                                   │  treasury_vault          │
                                   └──────────────────────────┘
                 ▲
                 │ signed XDR submitted directly
       ┌─────────┴──────────┐
       │  Freighter Wallet  │  ←  user signs in browser
       └────────────────────┘
```

The backend never holds private keys. Write operations return an unsigned XDR envelope. The frontend signs it with Freighter and submits directly to Horizon — the backend is never in the signing path.

---

## Non-custodial design

Every token transfer happens in a signed transaction that the user approves in their Freighter wallet. The backend builds the transaction (to handle fee estimation and sequence numbers) but the private key never leaves the user's device. This means:

- EuStellar-Pay cannot move funds on your behalf
- A compromised backend cannot drain wallets
- Employers and workers retain full control at all times

---

## Getting started

### Prerequisites
- [Freighter wallet](https://freighter.app) browser extension
- Stellar Testnet account funded via [Friendbot](https://friendbot.stellar.org)
- Node.js 20+, Rust + `cargo` (for contracts)

### Run locally

```bash
# Contracts
cd EuPay-smartcontract
cargo test                          # run test suites
stellar contract build              # build WASM

# Backend
cd EuPay-backend
cp .env.example .env                # fill in contract IDs + RPC URL
npm install && npm run dev          # http://localhost:3001

# Frontend
cd EuPay-frontend
cp .env.example .env                # fill in API URL + network
npm install && npm run dev          # http://localhost:5173
```

### Environment variables

| Variable | Description |
|---|---|
| `STELLAR_RPC_URL` | Soroban RPC endpoint |
| `STELLAR_NETWORK` | `testnet` / `mainnet` |
| `CONTRACT_PAYROLL_STREAM` | Deployed payroll_stream contract ID |
| `CONTRACT_TREASURY_VAULT` | Deployed treasury_vault contract ID |
| `SIM_SOURCE_ADDRESS` | Public key used for read simulations |
| `VITE_API_URL` | Backend URL for the frontend |
| `VITE_HORIZON_URL` | Horizon endpoint for tx submission |

---

## Test coverage

**Contracts (Rust):** `payroll_stream` — create and claim, pause/resume timing, cancel with correct settlement, solvency invariant across repeated claims, stream auto-completion at end time, self-stream rejection, fund extension, admin cancel. `treasury_vault` — deposit and balance, admin withdraw, allocate and release, reclaim unused allocation, insufficient funds rejection, emergency drain, double-initialize rejection, multi-token isolation.

**Backend (Jest):** `requireAuth` unit tests covering nonce format, signature verification, nonce consumption and replay prevention. Route validation tests for the nonce endpoint via supertest.

---

## License

Apache 2.0 · Built on [Stellar](https://stellar.org) and [Soroban](https://soroban.stellar.org)
