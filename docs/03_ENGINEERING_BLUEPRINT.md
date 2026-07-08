# Stellar Guardian — Engineering Blueprint

> **Status:** Draft
> **Version:** 1.0
> **Last Updated:** 2026-07-08
> **Owner:** Engineering

---

## Table of Contents

1. [Purpose & Scope](#1-purpose--scope)
2. [Prerequisites & Toolchain Setup](#2-prerequisites--toolchain-setup)
3. [Monorepo Structure & Conventions](#3-monorepo-structure--conventions)
4. [Smart Contract Implementation](#4-smart-contract-implementation)
5. [Frontend Implementation (Next.js 14)](#5-frontend-implementation-nextjs-14)
6. [Backend Implementation (Go REST API)](#6-backend-implementation-go-rest-api)
7. [Data Layer Implementation](#7-data-layer-implementation)
8. [Authentication Implementation](#8-authentication-implementation)
9. [Event Indexing Implementation](#9-event-indexing-implementation)
10. [Error Handling Standards](#10-error-handling-standards)
11. [Testing Strategy Per Layer](#11-testing-strategy-per-layer)
12. [Build Pipeline & CI/CD](#12-build-pipeline--cicd)
13. [Local Development Setup](#13-local-development-setup)
14. [Environment Configuration](#14-environment-configuration)
15. [Code Style & Review Standards](#15-code-style--review-standards)
16. [Dependency Management](#16-dependency-management)
17. [Phase-Gated Implementation Plan](#17-phase-gated-implementation-plan)
18. [Glossary](#18-glossary)

---

## 1. Purpose & Scope

This document defines *how* each layer of Stellar Guardian is implemented — the concrete engineering patterns, code organization, tooling choices, and implementation standards that every contributor must follow.

**Prerequisite reading:**
- [`01_PRODUCT_DISCOVERY.md`](./01_PRODUCT_DISCOVERY.md) — product context and user requirements
- [`02_ARCHITECTURE_REVIEW.md`](./02_ARCHITECTURE_REVIEW.md) — system structure, ADRs, component boundaries

**Scope boundaries:**
- This document covers implementation standards and engineering patterns. It does not reproduce architectural decisions already documented in `02_ARCHITECTURE_REVIEW.md` — it references them by section.
- Contract interface specifications and state machine details are in `08_SMART_CONTRACT_SPEC.md`.
- Database schema is in `06_DATABASE_DESIGN.md`.
- API endpoint contracts are in `07_API_SPECIFICATION.md`.
- Security controls and threat model are in `10_SECURITY.md`.

---

## 2. Prerequisites & Toolchain Setup

Every contributor must have the following installed before working on any part of the codebase.

### 2.1 Required Tools

| Tool | Minimum Version | Purpose | Install |
|---|---|---|---|
| **Node.js** | 20 LTS | Next.js web app and tooling scripts | `nvm install 20` |
| **pnpm** | 9.x | Monorepo package manager (ADR-005) | `npm install -g pnpm@9` |
| **Rust** | stable (≥ 1.78) | Soroban smart contract development | `rustup install stable` |
| **Soroban CLI** | ≥ 0.9.4 | Contract build, deploy, and test | `cargo install --locked soroban-cli` |
| **Go** | 1.22+ | REST API (Green Belt+) | `go.dev/dl` |
| **Docker** | 24+ | Local development environment | `docs.docker.com/get-docker` |
| **Docker Compose** | 2.x | Multi-service local orchestration | Bundled with Docker Desktop |
| **Git** | 2.40+ | Version control | `git-scm.com` |

### 2.2 Optional but Recommended

| Tool | Purpose |
|---|---|
| `cargo-watch` | Auto-recompile contracts on file change |
| `cargo-audit` | Dependency vulnerability scanning |
| `golangci-lint` | Go linter (required in CI) |
| `scout-audit` | Soroban contract static analysis (required in CI) |
| VS Code + `rust-analyzer` | Rust IDE support |
| VS Code + `golang.go` | Go IDE support |

### 2.3 Rust Target Setup

Soroban contracts compile to WASM. After installing Rust, add the WASM target:

```bash
rustup target add wasm32-unknown-unknown
```

Verify the Soroban CLI can build contracts:

```bash
soroban contract build --help
```

---

## 3. Monorepo Structure & Conventions

Stellar Guardian uses a **Turborepo monorepo with pnpm workspaces** (ADR-005). All JavaScript/TypeScript application code, shared packages, and tooling live in one repository. Rust contracts use Cargo workspaces within the `contracts/` directory.

### 3.1 Directory Layout

```
stellar-guardian/
├── apps/
│   ├── web/              # Next.js 14 — customer portal
│   ├── admin/            # Next.js — platform operator panel
│   └── docs/             # Documentation site (Nextra or similar)
├── packages/
│   ├── ui/               # Shared React components (TailwindCSS)
│   ├── types/            # Shared TypeScript types and interfaces
│   ├── validation/       # Zod schemas — input validation + contract param encoding
│   ├── api-client/       # SDK: REST API + Soroban RPC client
│   ├── shared/           # Common utilities (currency, date, address formatting)
│   └── config/           # Shared ESLint, TypeScript, Tailwind configuration
├── contracts/
│   ├── escrow/           # EscrowAgreement — core escrow contract
│   ├── dispute/          # DisputeResolution — dispute state machine
│   ├── reputation/       # ReputationLedger — anonymous review system
│   ├── marketplace/      # MarketplaceCoordinator — payment links + B2B
│   └── Cargo.toml        # Cargo workspace root
├── infrastructure/
│   ├── docker/           # Docker Compose files
│   ├── nginx/            # NGINX reverse proxy config
│   └── monitoring/       # Prometheus + Grafana config
├── tooling/
│   ├── scripts/          # Build, deploy, contract interaction scripts
│   └── generators/       # Code generators for new packages/apps
├── docs/                 # All documentation
├── turbo.json            # Turborepo pipeline config
├── pnpm-workspace.yaml   # pnpm workspace definitions
└── package.json          # Root package.json
```

### 3.2 pnpm Workspace Configuration

`pnpm-workspace.yaml`:

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

All package names use the `@stellar-guardian/` scope:

| Directory | Package Name |
|---|---|
| `packages/ui` | `@stellar-guardian/ui` |
| `packages/types` | `@stellar-guardian/types` |
| `packages/validation` | `@stellar-guardian/validation` |
| `packages/api-client` | `@stellar-guardian/api-client` |
| `packages/shared` | `@stellar-guardian/shared` |
| `packages/config` | `@stellar-guardian/config` |

### 3.3 Turborepo Pipeline

`turbo.json` defines the task dependency graph. Key pipeline rules:

```json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "lint": {
      "outputs": []
    },
    "type-check": {
      "dependsOn": ["^build"],
      "outputs": []
    }
  }
}
```

- `^build` means: build all dependencies first before building this package.
- Running `pnpm turbo build` from the root builds everything in the correct order with caching.

### 3.4 Package Dependency Rules

These rules are enforced by ESLint import boundaries. Violations fail CI.

| Package | May Import From | May NOT Import From |
|---|---|---|
| `apps/web` | `ui`, `types`, `validation`, `api-client`, `shared` | `apps/admin`, `apps/api` |
| `apps/admin` | `ui`, `types`, `api-client` | `apps/web`, `packages/validation` (direct) |
| `packages/ui` | `types`, `shared` | `api-client`, `validation` |
| `packages/api-client` | `types`, `validation` | `ui`, `shared` |
| `packages/validation` | `types` | All others |
| `packages/types` | Nothing | All others |
| `packages/shared` | Nothing | All others |

> **Rule:** No circular dependencies. `packages/types` and `packages/shared` are leaf nodes — they import nothing from this monorepo.

---

## 4. Smart Contract Implementation

All on-chain logic runs in **Rust compiled to Soroban WASM**. This section defines the implementation standards for all four contracts.

### 4.1 Cargo Workspace Setup

`contracts/Cargo.toml`:

```toml
[workspace]
members = [
    "escrow",
    "dispute",
    "reputation",
    "marketplace",
]
resolver = "2"

[workspace.dependencies]
soroban-sdk = { version = "=25.3.0", features = ["testutils"] }
soroban-fixed-point-math = { version = "=1.3.1" }
```

> **Pinning rationale:** Exact version pins (`=25.3.0`, `=1.3.1`) are used for production contracts per Recommendation §27.1 in `02_ARCHITECTURE_REVIEW.md`. CVE-2026-32322 (Soroban SDK — Fr scalar-field equality bug affecting BN254/BLS12-381 pairing-based cryptography, directly relevant to the nullifier scheme in `contracts/reputation/`) requires `soroban-sdk ≥ 25.3.0`. The crate `soroban-fixed-point-math` is the actively maintained Soroban-native fixed-point library — **not** `fixed-point-math`, which is deprecated on crates.io at v0.0.2. It is pinned at `=1.3.1` for auditable fee arithmetic; there is no verified CVE associated with it.

### 4.2 Contract File Structure

Each contract follows this directory layout:

```
contracts/escrow/
├── Cargo.toml
└── src/
    ├── lib.rs          # Contract entry point — exports public interface
    ├── contract.rs     # #[contract] struct and #[contractimpl] block
    ├── storage.rs      # Storage key definitions and read/write helpers
    ├── types.rs        # #[contracttype] structs and enums
    ├── events.rs       # Event emission functions
    ├── errors.rs       # #[contracterror] enum
    └── test.rs         # Unit and integration tests (#[cfg(test)])
```

### 4.3 Contract Implementation Pattern

Every contract follows this structure in `contract.rs`:

```rust
use soroban_sdk::{contract, contractimpl, Address, Env};
use crate::{storage, types, events, errors};

#[contract]
pub struct EscrowAgreement;

#[contractimpl]
impl EscrowAgreement {
    /// Initialize a new escrow agreement.
    /// Caller must be the payer — enforced by require_auth().
    pub fn initialize_escrow(
        env: Env,
        payer: Address,
        payee: Address,
        milestones: Vec<MilestoneDef>,
        claim_after: u64,
    ) -> Result<EscrowId, EscrowError> {
        payer.require_auth();                    // Auth gate — ALWAYS first
        storage::extend_instance_ttl(&env);      // TTL extension — ALWAYS after auth
        // ... business logic
        events::emit_escrow_initialized(&env, &escrow_id);
        Ok(escrow_id)
    }
}
```

**Non-negotiable implementation rules for every state-altering function:**
1. `require_auth()` is called **first**, before any state read or write
2. `extend_ttl()` is called on all touched Persistent Storage entries
3. An event is emitted for every state transition
4. Integer arithmetic uses the `soroban-fixed-point-math` library for fee calculations — no floating point

### 4.4 Storage Key Conventions

`storage.rs` defines all storage keys as typed enums:

```rust
use soroban_sdk::contracttype;

#[contracttype]
pub enum EscrowDataKey {
    Escrow(EscrowId),
    MilestoneList(EscrowId),
    Config,
}
```

- `Persistent` storage: active escrow state, milestone definitions, dispute records
- `Instance` storage: contract configuration (fee rate, admin address, paused flag, treasury address)
- `Temporary` storage: never used for cross-call state

### 4.5 Error Handling in Contracts

All contract errors use `#[contracterror]` with explicit integer discriminants:

```rust
use soroban_sdk::contracterror;

#[contracterror]
#[derive(Copy, Clone, Debug, Eq, PartialEq)]
#[repr(u32)]
pub enum EscrowError {
    NotFound = 1,
    Unauthorized = 2,
    InvalidState = 3,
    InvalidAmount = 4,
    TimelockNotReached = 5,
    AlreadyFunded = 6,
    MilestoneIndexOutOfBounds = 7,
    ContractPaused = 8,
}
```

Discriminants are stable across upgrades — never reorder or reassign them.

### 4.6 Contract Testing

Tests live in `src/test.rs` using the `soroban-sdk` test utilities:

```rust
#[cfg(test)]
mod test {
    use super::*;
    use soroban_sdk::testutils::{Address as _, AuthorizedFunction, Ledger};
    use soroban_sdk::{token, Env};

    #[test]
    fn test_initialize_and_fund_escrow() {
        let env = Env::default();
        env.mock_all_auths();
        // ... test setup and assertions
    }
}
```

Run tests with:

```bash
cargo test -p escrow
```

### 4.7 Building Contracts

```bash
# Build all contracts
soroban contract build

# Build a specific contract
soroban contract build --package escrow

# Output: contracts/target/wasm32-unknown-unknown/release/escrow.wasm
```

### 4.8 Deploying to Testnet

```bash
# Deploy EscrowAgreement to testnet
soroban contract deploy \
  --wasm contracts/target/wasm32-unknown-unknown/release/escrow.wasm \
  --source <DEPLOYER_KEYPAIR> \
  --network testnet

# Store the returned contract ID in environment config
```

Contract IDs are stored in `tooling/scripts/contract-ids.json` per network (testnet, mainnet).

---

## 5. Frontend Implementation (Next.js 14)

The web application is a **Next.js 14 App Router** application using TypeScript and TailwindCSS.

### 5.1 App Router Conventions

```
apps/web/
├── app/
│   ├── layout.tsx          # Root layout — providers, fonts, global styles
│   ├── page.tsx            # Landing page (server component)
│   ├── (auth)/
│   │   └── connect/        # Wallet connection flow
│   ├── escrows/
│   │   ├── page.tsx        # Escrow list (server component, SSR)
│   │   ├── [id]/
│   │   │   └── page.tsx    # Escrow detail (server component)
│   │   └── new/
│   │       └── page.tsx    # Create escrow (client component)
│   ├── disputes/
│   ├── reputation/
│   └── marketplace/
├── components/             # Page-level components (client/server mixed)
├── hooks/                  # Custom React hooks (wallet, escrow, auth)
├── lib/                    # Utility functions, RPC client, contract calls
├── providers/              # React context providers (wallet, theme)
└── public/                 # Static assets
```

### 5.2 Server vs Client Component Split

| Component Type | Use When | Examples |
|---|---|---|
| **Server Component** (default) | Public data, SEO-critical pages, no interactivity | Escrow detail page, reputation profile, landing page |
| **Client Component** (`'use client'`) | Wallet interaction, form state, real-time updates | Escrow creation form, funding button, dispute submission |

> **Rule:** Never import wallet kit, `@stellar/stellar-sdk`, or browser-only APIs in Server Components. These must be in Client Components or loaded dynamically with `next/dynamic`.

### 5.3 Wallet Integration

Wallet connection uses `@creit.tech/stellar-wallets-kit`:

```typescript
// providers/WalletProvider.tsx
'use client';
import { StellarWalletsKit, WalletNetwork, FREIGHTER_ID } from '@creit.tech/stellar-wallets-kit';

const kit = new StellarWalletsKit({
  network: WalletNetwork.TESTNET, // from env: NEXT_PUBLIC_STELLAR_NETWORK
  selectedWalletId: FREIGHTER_ID,
});
```

The wallet provider exposes: `connect()`, `disconnect()`, `signTransaction()`, `publicKey`.

Private keys never enter application code. The kit handles all signing internally.

### 5.4 Direct Soroban RPC (White/Yellow Belt)

Before the Go API exists, the frontend calls Soroban directly via `@stellar/stellar-sdk`:

```typescript
// lib/soroban.ts
import { Contract, TransactionBuilder, Networks } from '@stellar/stellar-sdk';

export async function fundEscrow(
  escrowId: string,
  amount: bigint,
  walletKit: StellarWalletsKit
): Promise<string> {
  const contract = new Contract(ESCROW_CONTRACT_ID);
  const operation = contract.call('fund_escrow', ...[escrowId, amount]);
  // build → sign → submit → poll for result
}
```

### 5.5 Environment Variables

All environment variables for `apps/web/` are prefixed `NEXT_PUBLIC_` for client-side access:

```bash
NEXT_PUBLIC_STELLAR_NETWORK=testnet          # testnet | mainnet
NEXT_PUBLIC_STELLAR_RPC_URL=https://soroban-testnet.stellar.org
NEXT_PUBLIC_ESCROW_CONTRACT_ID=C...
NEXT_PUBLIC_DISPUTE_CONTRACT_ID=C...
NEXT_PUBLIC_REPUTATION_CONTRACT_ID=C...
NEXT_PUBLIC_MARKETPLACE_CONTRACT_ID=C...
NEXT_PUBLIC_API_URL=https://api.stellarguardian.io  # Green Belt+
```

Server-only variables (no `NEXT_PUBLIC_` prefix) are never exposed to the browser.

---

## 6. Backend Implementation (Go REST API)

The Go API is a **modular monolith** (ADR-001) introduced at Green Belt. It is a single deployable binary with strong internal module boundaries enforced via Go interfaces.

### 6.1 Project Layout

```
apps/api/
├── cmd/
│   └── server/
│       └── main.go         # Entry point — wires dependencies, starts server
├── internal/
│   ├── auth/               # SEP-10 challenge/verify, JWT issuance
│   ├── escrow/             # Escrow read queries, contract invocation proxy
│   ├── dispute/            # Dispute records, arbiter coordination
│   ├── reputation/         # Score queries, review submission
│   ├── marketplace/        # Checkout sessions, SEP-38 proxy
│   ├── notification/       # Mercury webhook handler, SSE stream
│   ├── worker/
│   │   └── ttl/            # TTL monitoring worker
│   ├── metrics/            # Prometheus metrics definitions
│   ├── repository/         # PostgreSQL (pgx) + Redis data access layer
│   ├── middleware/         # Auth, rate limiting, logging middleware
│   └── config/             # Config loading from environment
├── pkg/
│   ├── stellar/            # Stellar RPC client wrapper
│   └── ipfs/               # Pinata IPFS client
├── migrations/             # SQL migration files (goose)
├── Dockerfile
└── go.mod
```

### 6.2 Module Boundary Enforcement

Each internal module exposes a typed interface. No module imports another module's concrete types directly:

```go
// internal/escrow/service.go
package escrow

type Service interface {
    GetEscrow(ctx context.Context, id string) (*Escrow, error)
    ListEscrows(ctx context.Context, filter EscrowFilter) ([]*Escrow, error)
}

// internal/dispute/service.go — imports escrow.Service interface, NOT escrow struct
package dispute

type Service interface {
    OpenDispute(ctx context.Context, req OpenDisputeRequest) (*Dispute, error)
}
```

> **Enforcement rule:** `go vet` + a custom linter rule checks that no file in `internal/moduleA` imports `internal/moduleB` via concrete types. Only interface imports are permitted.

### 6.3 HTTP Router

The API uses the standard `net/http` router (Go 1.22+ supports path parameters natively):

```go
// cmd/server/main.go
mux := http.NewServeMux()
mux.HandleFunc("GET /api/v1/escrows", escrowHandler.List)
mux.HandleFunc("GET /api/v1/escrows/{id}", escrowHandler.Get)
mux.HandleFunc("POST /api/v1/auth/challenge", authHandler.Challenge)
mux.HandleFunc("POST /api/v1/auth/token", authHandler.Token)
```

All handlers follow this signature:

```go
func (h *EscrowHandler) Get(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    escrow, err := h.service.GetEscrow(r.Context(), id)
    if err != nil {
        writeError(w, err)
        return
    }
    writeJSON(w, http.StatusOK, escrow)
}
```

### 6.4 Repository Layer

The repository layer uses `pgx/v5` for PostgreSQL and `go-redis/v9` for Redis:

```go
// internal/repository/escrow_repo.go
package repository

type EscrowRepository interface {
    FindByID(ctx context.Context, id string) (*EscrowRow, error)
    ListByAddress(ctx context.Context, address string, limit, offset int) ([]*EscrowRow, error)
}

type pgEscrowRepository struct {
    pool *pgxpool.Pool
}
```

All SQL queries are parameterized. Raw string interpolation into SQL is prohibited.

### 6.5 TTL Monitoring Worker

The TTL worker runs as a goroutine started at API boot:

```go
// internal/worker/ttl/worker.go
func (w *TTLWorker) Run(ctx context.Context) {
    ticker := time.NewTicker(w.interval) // default: 5 minutes
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            w.checkEscrowTTLs(ctx)
        case <-ctx.Done():
            return
        }
    }
}
```

When an escrow TTL falls below the safety threshold (configurable, default: 48 hours), the worker:
1. Logs a `WARN` entry with the escrow ID and remaining TTL
2. Increments the `stellar_guardian_ttl_warnings_total` Prometheus counter
3. Optionally submits an `extend_ttl` transaction (when `TTL_AUTO_EXTEND=true`)

---

## 7. Data Layer Implementation

### 7.1 PostgreSQL Connection

The Go API uses `pgxpool` for connection pooling:

```go
// internal/config/database.go
pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
// DATABASE_URL format: postgres://user:pass@host:5432/stellar_guardian
```

Pool configuration:
- `MaxConns`: 20
- `MinConns`: 2
- `MaxConnLifetime`: 1 hour
- `HealthCheckPeriod`: 30 seconds

### 7.2 Database Migrations

Migrations use **goose** with SQL files in `apps/api/migrations/`:

```bash
# Run all pending migrations
goose -dir apps/api/migrations postgres "$DATABASE_URL" up

# Roll back last migration
goose -dir apps/api/migrations postgres "$DATABASE_URL" down
```

Migration file naming: `{version}_{description}.sql` (e.g., `001_create_escrow_agreements.sql`).

All migration files are idempotent and tested in CI before merge.

### 7.3 Redis Connection

```go
// internal/config/redis.go
rdb := redis.NewClient(&redis.Options{
    Addr:     os.Getenv("REDIS_URL"), // redis://localhost:6379
    Password: os.Getenv("REDIS_PASSWORD"),
    DB:       0,
})
```

Key naming conventions:

| Purpose | Key Pattern | TTL |
|---|---|---|
| SEP-10 challenge nonce | `sep10:nonce:{account}` | 15 minutes |
| Rate limit counter | `ratelimit:{account}:{endpoint}` | 1 minute |
| JWT revocation (optional) | `jwt:revoked:{jti}` | Token expiry |

### 7.4 Core Table Definitions

These are the conceptual table definitions. Full schema with indexes and constraints is in `06_DATABASE_DESIGN.md`.

```sql
-- Denormalized escrow state mirrored from on-chain events
CREATE TABLE escrow_agreements (
    id              TEXT PRIMARY KEY,          -- on-chain escrow ID
    payer           TEXT NOT NULL,             -- Stellar account address
    payee           TEXT NOT NULL,
    state           TEXT NOT NULL,             -- Pending|Active|Disputed|Completed|Refunded|Expired
    asset_code      TEXT NOT NULL DEFAULT 'USDC',
    asset_issuer    TEXT NOT NULL,
    total_amount    NUMERIC(20, 7) NOT NULL,
    claim_after     TIMESTAMPTZ,
    donation_flag   BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    ledger_sequence BIGINT NOT NULL            -- last ledger that updated this row
);

CREATE TABLE escrow_milestones (
    escrow_id       TEXT NOT NULL REFERENCES escrow_agreements(id),
    index           INTEGER NOT NULL,
    description_hash TEXT NOT NULL,            -- IPFS CID or SHA-256 of description
    amount          NUMERIC(20, 7) NOT NULL,
    status          TEXT NOT NULL,             -- Pending|Released|Disputed
    PRIMARY KEY (escrow_id, index)
);

CREATE TABLE dispute_records (
    id              TEXT PRIMARY KEY,
    escrow_id       TEXT NOT NULL REFERENCES escrow_agreements(id),
    initiator       TEXT NOT NULL,
    state           TEXT NOT NULL,             -- Open|Resolved
    evidence_hash   TEXT,                      -- IPFS CID
    ruling          TEXT,                      -- release|refund|split
    created_at      TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ
);

CREATE TABLE reputation_scores (
    address         TEXT PRIMARY KEY,
    score           NUMERIC(4, 2) NOT NULL DEFAULT 0,
    review_count    INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Append-only audit log of all on-chain events
CREATE TABLE platform_events (
    id              BIGSERIAL PRIMARY KEY,
    event_type      TEXT NOT NULL,
    contract_id     TEXT NOT NULL,
    ledger_sequence BIGINT NOT NULL,
    transaction_hash TEXT NOT NULL,
    payload         JSONB NOT NULL,
    indexed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 8. Authentication Implementation

### 8.1 SEP-10 Flow Implementation

The full SEP-10 implementation in the Go auth module:

```go
// internal/auth/sep10.go

// Step 1: Generate challenge
func (s *Service) GenerateChallenge(ctx context.Context, account string) (string, error) {
    nonce := generateSecureNonce()
    // Build unsigned Stellar transaction with manage_data operations
    challenge := buildSEP10Challenge(account, nonce, s.serverKeypair)
    // Store nonce in Redis with 15-minute TTL
    s.redis.Set(ctx, "sep10:nonce:"+account, nonce, 15*time.Minute)
    return challenge.ToXDR(), nil
}

// Step 2: Verify signed challenge and issue JWT
func (s *Service) VerifyChallenge(ctx context.Context, signedXDR string) (string, error) {
    tx := parseAndValidateChallenge(signedXDR)
    account := extractAccount(tx)
    // Verify nonce exists in Redis
    if !s.redis.Exists(ctx, "sep10:nonce:"+account) {
        return "", ErrChallengeExpired
    }
    // Verify signatures against account's signers on Stellar network
    if err := verifySignatures(tx, s.horizonClient); err != nil {
        return "", ErrInvalidSignature
    }
    // Delete nonce (single-use)
    s.redis.Del(ctx, "sep10:nonce:"+account)
    // Issue JWT scoped to account
    return s.issueJWT(account)
}
```

### 8.2 JWT Configuration

```go
// internal/auth/jwt.go
type Claims struct {
    Account string `json:"sub"`   // Stellar account address
    jwt.RegisteredClaims
}

// JWT lifetime: 24 hours
// Algorithm: HS256 with HMAC secret from JWT_SECRET env var
// Storage: In-memory only — never in localStorage or cookies
```

### 8.3 Auth Middleware

All protected API routes use the auth middleware:

```go
// internal/middleware/auth.go
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := extractBearerToken(r)
        claims, err := validateJWT(token)
        if err != nil {
            writeError(w, http.StatusUnauthorized, "invalid_token")
            return
        }
        ctx := context.WithValue(r.Context(), AccountKey, claims.Account)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

---

## 9. Event Indexing Implementation

Mercury + Zephyr VM handles event streaming from Soroban to PostgreSQL (ADR-002, Green Belt+).

### 9.1 Zephyr VM Program Structure

Each contract has a corresponding Zephyr program that defines the event schema and transformation logic:

```
tooling/zephyr/
├── escrow_indexer.rs       # Handles EscrowAgreement events
├── dispute_indexer.rs      # Handles DisputeResolution events
├── reputation_indexer.rs   # Handles ReputationLedger events
└── marketplace_indexer.rs  # Handles MarketplaceCoordinator events
```

### 9.2 Event Processing Pattern

Each Zephyr program follows this pattern:

```rust
// tooling/zephyr/escrow_indexer.rs
use zephyr_sdk::{prelude::*, DatabaseInteract};

#[derive(DatabaseInteract)]
pub struct EscrowAgreementRow {
    pub id: String,
    pub payer: String,
    pub payee: String,
    pub state: String,
    pub total_amount: i128,
    pub claim_after: Option<i64>,
    pub ledger_sequence: i64,
}

#[no_mangle]
pub extern "C" fn on_close() {
    let env = EnvClient::new();
    // Filter for EscrowAgreement contract events
    for event in env.reader().pretty().soroban_events() {
        if event.contract == ESCROW_CONTRACT_ID {
            process_escrow_event(&env, event);
        }
    }
}
```

### 9.3 Fallback RPC Polling

When Mercury is unavailable, the Go API falls back to direct Stellar RPC polling:

```go
// pkg/stellar/poller.go
func (p *Poller) PollEscrowState(ctx context.Context, escrowID string) (*EscrowState, error) {
    result, err := p.rpcClient.GetContractData(ctx, ESCROW_CONTRACT_ID, escrowID)
    if err != nil {
        return nil, fmt.Errorf("rpc poll failed: %w", err)
    }
    return parseEscrowState(result), nil
}
```

Polling interval under fallback: 30 seconds. The UI displays a "Data may be delayed" banner when Mercury lag exceeds 30 seconds.

---

## 10. Error Handling Standards

### 10.1 Contract Errors (Rust)

Contract errors use `#[contracterror]` enums returned as `Result<T, ContractError>`. Errors are surfaced to callers as Soroban SDK error codes. Never panic in contract code — always return a typed error.

### 10.2 API Errors (Go)

All API errors follow a consistent JSON envelope:

```json
{
  "error": {
    "code": "escrow_not_found",
    "message": "Escrow with ID C... does not exist",
    "request_id": "req_abc123"
  }
}
```

Error code conventions:

| HTTP Status | Error Code Pattern | Example |
|---|---|---|
| 400 | `invalid_{field}` | `invalid_escrow_id` |
| 401 | `unauthorized` | `invalid_token` |
| 403 | `forbidden` | `insufficient_signers` |
| 404 | `{resource}_not_found` | `escrow_not_found` |
| 409 | `{resource}_{conflict}` | `escrow_already_funded` |
| 500 | `internal_error` | `internal_error` |
| 503 | `service_unavailable` | `rpc_unavailable` |

### 10.3 Frontend Errors (TypeScript)

All Soroban RPC calls are wrapped in typed error handlers:

```typescript
// lib/errors.ts
export class SorobanError extends Error {
  constructor(
    public readonly code: string,
    public readonly contractError?: number,
    message?: string
  ) {
    super(message ?? code);
  }
}

export function parseSorobanError(raw: unknown): SorobanError {
  // Map Soroban SDK error codes to typed SorobanError instances
}
```

User-facing error messages never expose internal error codes or stack traces.

---

## 11. Testing Strategy Per Layer

### 11.1 Smart Contract Tests (Rust)

| Test Type | Tool | Location | What It Covers |
|---|---|---|---|
| Unit tests | `cargo test` | `src/test.rs` | Individual function logic, state transitions |
| Integration tests | `soroban-sdk` test env | `src/test.rs` | Multi-step flows (create → fund → approve) |
| Property tests | `proptest` crate | `src/test.rs` | Fee calculation invariants, state machine exhaustion |
| Static analysis | `scout-audit` | CI only | Known vulnerability patterns |

Run all contract tests:

```bash
cargo test --workspace
```

### 11.2 Frontend Tests (TypeScript)

| Test Type | Tool | Location | What It Covers |
|---|---|---|---|
| Unit tests | Vitest | `*.test.ts` | Utility functions, formatters, validators |
| Component tests | Vitest + Testing Library | `*.test.tsx` | Component rendering, user interactions |
| E2E tests | Playwright | `e2e/` | Full user flows (wallet connect → create escrow) |

Run frontend tests:

```bash
pnpm turbo test --filter=web
```

### 11.3 API Tests (Go)

| Test Type | Tool | Location | What It Covers |
|---|---|---|---|
| Unit tests | `testing` stdlib | `*_test.go` | Service logic, repository mock behavior |
| Integration tests | `testcontainers-go` | `*_integration_test.go` | Real PostgreSQL + Redis, HTTP handler behavior |
| Contract tests | `net/http/httptest` | `*_handler_test.go` | API response format, status codes |

Run API tests:

```bash
go test ./... -v
```

---

## 12. Build Pipeline & CI/CD

### 12.1 GitHub Actions Workflow Structure

```
.github/workflows/
├── ci.yml              # PR validation: lint, type-check, test, build
├── deploy-testnet.yml  # Deploy contracts + API to testnet on merge to main
└── deploy-mainnet.yml  # Deploy to mainnet (manual trigger, requires approval)
```

### 12.2 CI Pipeline (ci.yml)

Every pull request runs these checks in order:

```yaml
jobs:
  contracts:
    steps:
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo test --workspace
      - run: scout-audit                    # Security static analysis (required)

  frontend:
    steps:
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo lint
      - run: pnpm turbo type-check
      - run: pnpm turbo test --filter=web
      - run: pnpm turbo build

  api:
    steps:
      - run: go vet ./...
      - run: golangci-lint run
      - run: go test ./... -race -count=1
```

All jobs must pass. Any failure blocks merge.

### 12.3 Testnet Deployment

On merge to `main`:

1. Build contract WASM artifacts
2. Deploy updated contracts to Stellar Testnet (if WASM hash changed)
3. Update `tooling/scripts/contract-ids.json` with new IDs
4. Deploy Go API to staging environment
5. Run smoke tests against testnet

### 12.4 Mainnet Deployment Gate

Mainnet deployment requires:
- All CI checks passing on a release tag
- External security audit completed and findings resolved
- Manual approval from 2 maintainers in GitHub Environments

---

## 13. Local Development Setup

### 13.1 First-Time Setup

```bash
# 1. Clone the repository
git clone https://github.com/stellar-guardian/stellar-guardian.git
cd stellar-guardian

# 2. Install Node.js dependencies
pnpm install

# 3. Install Rust and Soroban targets
rustup install stable
rustup target add wasm32-unknown-unknown
cargo install --locked soroban-cli

# 4. Copy environment template
cp .env.example .env.local
# Edit .env.local with your testnet credentials

# 5. Start local services (PostgreSQL, Redis)
docker compose -f infrastructure/docker/docker-compose.dev.yml up -d

# 6. Run database migrations (Green Belt+)
cd apps/api && goose -dir migrations postgres "$DATABASE_URL" up

# 7. Start the web app
pnpm turbo dev --filter=web
```

### 13.2 Local Stellar Network

For White/Yellow Belt development without testnet dependency:

```bash
# Start a local Stellar standalone network
docker compose -f infrastructure/docker/docker-compose.stellar.yml up -d

# This starts:
# - stellar/quickstart with standalone mode
# - Soroban RPC on localhost:8000
```

Set `NEXT_PUBLIC_STELLAR_NETWORK=standalone` and `NEXT_PUBLIC_STELLAR_RPC_URL=http://localhost:8000` in `.env.local`.

### 13.3 Contract Development Loop

```bash
# Build and deploy to local network in one command
cd contracts
cargo build --target wasm32-unknown-unknown --release

soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/escrow.wasm \
  --source alice \
  --network standalone

# Invoke a function for testing
soroban contract invoke \
  --id <CONTRACT_ID> \
  --source alice \
  --network standalone \
  -- initialize_escrow \
  --payer GABC... \
  --payee GDEF...
```

---

## 14. Environment Configuration

### 14.1 Environment Files

| File | Purpose | Committed? |
|---|---|---|
| `.env.example` | Template with all required vars (no secrets) | ✅ Yes |
| `.env.local` | Local development overrides | ❌ No (gitignored) |
| `.env.test` | Test environment values | ✅ Yes (no real secrets) |

### 14.2 Complete Environment Variable Reference

**Shared (all apps):**

```bash
STELLAR_NETWORK=testnet                    # testnet | mainnet | standalone
STELLAR_RPC_URL=https://soroban-testnet.stellar.org
STELLAR_HORIZON_URL=https://horizon-testnet.stellar.org
```

**Frontend (`apps/web/`):**

```bash
NEXT_PUBLIC_STELLAR_NETWORK=testnet
NEXT_PUBLIC_STELLAR_RPC_URL=https://soroban-testnet.stellar.org
NEXT_PUBLIC_ESCROW_CONTRACT_ID=C...
NEXT_PUBLIC_DISPUTE_CONTRACT_ID=C...
NEXT_PUBLIC_REPUTATION_CONTRACT_ID=C...
NEXT_PUBLIC_MARKETPLACE_CONTRACT_ID=C...
NEXT_PUBLIC_API_URL=http://localhost:8080
```

**Backend (`apps/api/`):**

```bash
DATABASE_URL=postgres://sg_user:pass@localhost:5432/stellar_guardian
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=
JWT_SECRET=<min 32 chars — generate with: openssl rand -hex 32>
SERVER_KEYPAIR=<Stellar keypair for SEP-10 server signing>
ESCROW_CONTRACT_ID=C...
TTL_AUTO_EXTEND=false
TTL_WARNING_THRESHOLD_HOURS=48
PORT=8080
```

**Monitoring:**

```bash
PROMETHEUS_PORT=9090
GRAFANA_PORT=3001
SENTRY_DSN=https://...@sentry.io/...
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

> **Security rule:** No secret values are committed to version control. All secrets are injected at deploy time via environment variables or a secrets manager. The `.env.example` file contains only placeholder values.

---

## 15. Code Style & Review Standards

### 15.1 Rust (Contracts)

- Format: `rustfmt` with default settings — enforced by `cargo fmt --check` in CI
- Lint: `clippy` with `-D warnings` — all clippy warnings are errors in CI
- Documentation: All public functions have doc comments (`///`)
- Naming: `snake_case` for functions and variables, `PascalCase` for types and enums
- Error handling: All fallible operations return `Result<T, ContractError>` — never `unwrap()` in production code

### 15.2 TypeScript (Frontend)

- Format: Prettier with project config in `packages/config/prettier.config.js`
- Lint: ESLint with `@stellar-guardian/config/eslint` preset
- Type safety: `strict: true` in all `tsconfig.json` files — no `any` without explicit justification
- Naming: `camelCase` for variables/functions, `PascalCase` for components/types, `SCREAMING_SNAKE_CASE` for constants
- Imports: Absolute imports using `@/` alias for `apps/web/` internal imports

### 15.3 Go (API)

- Format: `gofmt` — enforced by `golangci-lint` in CI
- Lint: `golangci-lint` with project config in `apps/api/.golangci.yml`
- Error handling: All errors wrapped with `fmt.Errorf("context: %w", err)` — never silently discarded
- Naming: Standard Go conventions (`camelCase` private, `PascalCase` exported)
- Context: All I/O operations accept `context.Context` as the first parameter

### 15.4 Pull Request Standards

Every PR must:
- Pass all CI checks (contracts, frontend, API)
- Include tests for new functionality
- Reference the relevant spec document section if implementing a documented feature
- Have a description explaining what changed and why (not just what)
- Be reviewed by at least one other engineer before merge

---

## 16. Dependency Management

### 16.1 JavaScript/TypeScript

- Package manager: `pnpm` only — `npm` and `yarn` are prohibited (ADR-005)
- Lock file: `pnpm-lock.yaml` is committed and kept up to date
- Adding dependencies: `pnpm add <package> --filter <app-or-package>`
- Version pinning: Prefer exact versions (`"1.2.3"` not `"^1.2.3"`) for security-sensitive packages
- Audit: `pnpm audit` runs in CI; high/critical vulnerabilities block merge

### 16.2 Rust (Contracts)

- Lock file: `contracts/Cargo.lock` is committed (binary crates commit lock files)
- Adding dependencies: Add to `[workspace.dependencies]` in `contracts/Cargo.toml` first
- Version pinning: Exact pins (`=x.y.z`) for `soroban-sdk` and `soroban-fixed-point-math` — see §4.1
- Audit: `cargo audit` runs in CI

### 16.3 Go (API)

- Module file: `apps/api/go.mod` — committed and kept tidy
- Adding dependencies: `go get <package>@<version>` followed by `go mod tidy`
- Version pinning: Go modules use minimum version selection — specify exact versions in `go.mod`
- Audit: `govulncheck` runs in CI

---

## 17. Phase-Gated Implementation Plan

Each Journey to Mastery phase has a concrete implementation scope. Engineers must not implement Black Belt features during White Belt — this is both a project constraint and a security principle.

| Belt | Implementation Scope | Exit Criteria |
|---|---|---|
| **White Belt** | `contracts/escrow/` — `initialize_escrow`, `fund_escrow`, `approve_milestone`, `claim_after_expiry`; local Docker; basic Next.js page | `cargo test` passes; local fund + release cycle works end-to-end |
| **Yellow Belt** | Multi-milestone escrow on Testnet; wallet kit integration in web app; `open_dispute` stub | All escrow state transitions work on Testnet via wallet signing |
| **Orange Belt** | 2-of-3 multisig escrow support; `check-auth` custom authorization; fee-bump sponsorship (minimal signing service introduced — see ADR-004) | Multisig escrow created and released via 2 different wallets; fee-bump sponsorship confirmed |
| **Green Belt** | Go REST API; PostgreSQL + Redis; Mercury Zephyr indexer; `contracts/dispute/`; TTL worker | Full read layer operational; dispute flow works on Testnet |
| **Blue Belt** | SEP-24/SEP-38 anchor integration; `contracts/reputation/`; IPFS evidence upload | End-to-end fiat-to-escrow flow works with a live SEP-24 anchor |
| **Black Belt** | `contracts/marketplace/`; decentralized juror pool; Mainnet deployment; external audit complete | No critical audit findings; $0 active escrow value on Testnet migrated to Mainnet procedures |

> **Hard gates:** No Mainnet deployment without a completed external Rust/Soroban security audit. This gate cannot be waived regardless of schedule pressure.

---

## 18. Glossary

| Term | Definition |
|---|---|
| **App Router** | Next.js 14 routing system based on the `app/` directory. Supports React Server Components and nested layouts. |
| **Cargo workspace** | A Rust feature that groups multiple crates under one `Cargo.toml`, sharing dependencies and build artifacts. |
| **goose** | A database migration tool for Go using SQL files with up/down migrations. |
| **pgx** | A PostgreSQL driver for Go (`pgx/v5`) providing low-level query execution and connection pooling. |
| **proptest** | A Rust property-based testing library. Generates random inputs to find edge cases in pure functions. |
| **Scout Audit** | A static analysis tool for Soroban smart contracts that detects known vulnerability patterns. Required in CI. |
| **Server Component** | A React component that renders on the server. Cannot use browser APIs, event handlers, or React state. |
| **Turborepo** | A build system for JavaScript/TypeScript monorepos that caches task outputs and parallelizes builds. |
| **Zephyr VM** | The Mercury indexer's programmable transformation layer. Written in Rust; processes on-chain events into structured database rows. |
| **pgxpool** | The connection pool implementation in `pgx/v5`. Manages a pool of reusable PostgreSQL connections. |
| **testcontainers-go** | A Go library that spins up real Docker containers (PostgreSQL, Redis) for integration testing. |
| **golangci-lint** | A Go linter aggregator. Runs multiple linters in parallel and enforces code quality in CI. |
| **govulncheck** | The official Go vulnerability checker. Scans `go.mod` against the Go vulnerability database. |

---

*Document classification: Internal — Engineering*
*Previous document: [`02_ARCHITECTURE_REVIEW.md`](./02_ARCHITECTURE_REVIEW.md)*
*Next document: [`04_PRODUCT_PLAN.md`](./04_PRODUCT_PLAN.md)*
*Revision notes: v1.0 — Initial authored version. Covers White Belt through Black Belt implementation standards. References `02_ARCHITECTURE_REVIEW.md` for all architectural decisions. Implementation details for contract interfaces deferred to `08_SMART_CONTRACT_SPEC.md`; database schema to `06_DATABASE_DESIGN.md`; API endpoint contracts to `07_API_SPECIFICATION.md`.*
