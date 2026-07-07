---
inclusion: always
---

# Project Context — Stellar Guardian

## What This Project Is

Stellar Guardian is a **non-custodial, decentralized trust and escrow infrastructure** built on the Stellar network using Soroban smart contracts (Rust/WASM). It is not a wallet, not a payment gateway, and not a centralized escrow service. It is a programmable trust layer — a protocol and application platform that allows two untrusted parties to transact with cryptographic safety guarantees.

---

## Core Problem Being Solved

Peer-to-peer and cross-border commerce suffers from a fundamental trust asymmetry: one party must perform first, exposing themselves to fraud. Centralized escrow solutions (Escrow.com, Upwork, PayPal) solve this but introduce high fees (3–10%), geographic exclusion, arbitrary account freezes, and custodial risk. EVM-based smart contract escrows (Ethereum/Kleros) solve custody but fail on gas cost predictability, onboarding friction, and state management.

Stellar Guardian solves all of these simultaneously using the Stellar network's sub-cent fees, ~5-second finality, Soroban's optimized state model, and native stablecoin (USDC/EURC) interoperability.

---

## Target Users

| Role | Description |
|---|---|
| Buyer / Payer | Purchases goods or services and funds the escrow |
| Seller / Payee | Delivers goods or services and claims payment |
| Freelancer | Milestone-based worker seeking guaranteed payment |
| B2B Merchant | Cross-border trade finance participant |
| NGO / Donor | Traceable donation disbursement |
| Arbiter | Third-party dispute resolver (trusted, multisig, or pooled) |

---

## Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| Smart Contracts | Rust → Soroban (WASM) | Gas-optimized, type-safe, auditable |
| Frontend | Next.js (TypeScript) + TailwindCSS | SSR, ecosystem maturity |
| Wallet Integration | `@creit.tech/stellar-wallets-kit` | Aggregates Freighter, Albedo, Hana |
| Backend (Phase 2+) | Go (Golang) | Concurrency, low memory, microservice-ready |
| Database | PostgreSQL + Redis | Normalized read cache + rate limiting |
| Indexer | Mercury + Zephyr VM | Real-time on-chain event indexing to PostgreSQL |
| Storage | IPFS via Pinata | Immutable dispute evidence |
| Auth | SEP-10 (Stellar Web Auth) | Non-custodial wallet-based authentication |
| Fiat On-Ramp | SEP-24 / SEP-38 | Card/bank-to-stablecoin, anchor-based |

---

## Architectural Decisions (Locked)

| ADR | Decision | Status |
|---|---|---|
| ADR-001 | Modular Monolith (not microservices) for initial release | Approved |
| ADR-002 | Mercury + Zephyr VM for off-chain indexing (not custom RPC scraper) | Approved |
| ADR-003 | Soroban Persistent Storage for active escrows; Instance Storage for config; programmatic TTL extension on every interaction | Approved |
| ADR-004 | No backend until Level 3–4 of Journey to Mastery; frontend talks directly to Soroban contracts in early phases | Approved |
| ADR-005 | Turborepo monorepo with pnpm workspaces | Approved |

---

## Monorepo Structure (Target)

```
stellar-guardian/
├── apps/
│   ├── web/          # Next.js customer portal
│   ├── admin/        # Admin panel
│   └── docs/         # Documentation site
├── packages/
│   ├── ui/           # Shared components
│   ├── types/        # Shared TypeScript types
│   ├── validation/   # Zod schemas
│   ├── api-client/   # API SDK
│   ├── shared/       # Common utilities
│   └── config/       # Shared configuration
├── contracts/
│   ├── escrow/       # Core escrow contract
│   ├── dispute/      # Dispute resolution contract
│   ├── reputation/   # Reputation system contract
│   └── marketplace/  # Marketplace contract
├── infrastructure/
│   ├── docker/
│   ├── nginx/
│   └── monitoring/
├── tooling/
│   ├── scripts/
│   └── generators/
├── docs/             # All documentation lives here
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

---

## Journey to Mastery — Implementation Phases

| Belt | Phase | Key Deliverable |
|---|---|---|
| White Belt | Local sandbox | Time-locked escrow contract, local Docker |
| Yellow Belt | Testnet deployment | Multi-milestone escrow on public testnet |
| Orange Belt | Custom auth & multisig | 2-of-3 multisig escrows, fee-bump sponsorship |
| Green Belt | Data indexing | Mercury Retroshades + PostgreSQL read-cache |
| Blue Belt | Fiat ramps | SEP-24/SEP-38 checkout experience |
| Black Belt | Mainnet + decentralized court | Game-theoretic juror pool, global launch |

---

## Security Baseline (Non-Negotiable)

- Every state-altering contract function must call `Address.require_auth()`.
- No admin key may directly drain active, non-disputed escrows.
- All active escrow state uses Persistent Storage with programmatic TTL extension.
- Soroban SDK pinned to `>=25.3.0` (CVE-2026-32322 fix).
- Fixed-point math library pinned to `>=1.3.1` (CVE-2026-24783 fix).
- Scout Audit static analysis runs on every CI pull request.
- Emergency pause via 3-of-5 multisig admin control.

---

## Business Model

- **Primary Revenue:** 0.5% platform service fee on completed escrows (capped at 50 USDC per transaction).
- **Secondary Revenue:** Dispute filing fees (deterrent mechanism) + white-label integration fees.
- **Donation escrows:** Fee waived.

---

## What This Project Is NOT

- Not a custodial wallet or exchange.
- Not an EVM/Ethereum project.
- Not a centralized SaaS escrow.
- Not a token/ICO project.
- Not specific to one geographic market.
