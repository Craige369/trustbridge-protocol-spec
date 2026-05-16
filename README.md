# TrustBridge Protocol

**Safe-powered non-custodial escrow infrastructure for Web3**

[![Status](https://img.shields.io/badge/status-prototype%20%2F%20pre--audit-amber)](https://trustbridgeprotocol.xyz)
[![Chain](https://img.shields.io/badge/chain-Base%20%2B%20Polygon-blue)](https://base.org)
[![Architecture](https://img.shields.io/badge/architecture-Safe%202--of--2%20multisig-purple)](https://safe.global)
[![Licence](https://img.shields.io/badge/licence-MIT%20(spec%20layer)-green)](LICENSE)

---

## What TrustBridge Protocol is

TrustBridge Protocol is a Safe-native trust workflow layer that enables two parties who do not trust each other to lock, negotiate, and exchange crypto assets with no custodian, no intermediary, and no single point of failure.

Every TrustBridge deal creates a **Safe 2-of-2 multisig smart account**. Every vault is locked until both parties approve release. Every failed deal auto-refunds both deposits via an on-chain time lock. Neither party nor TrustBridge ever holds user funds.

> **Current status:** Working React/JSX prototype — full 5-screen deal flow functional, all core security mechanisms in place. Smart contract layer is in specification stage. This repository contains the smart contract specification, security architecture, and known limitations register. Production contracts have not yet been written or audited.

---

## Live prototype

A working prototype demonstrating the complete 5-screen deal flow is 
available for review on request. The prototype implements all core security 
mechanisms including the ConfirmDeliveryModal, price band breach flow, 
Adjust Deal audit trail, and dispute mechanism.

To request prototype access: craig@trustbridgeprotocol.xyz

The prototype demonstrates the complete user journey:

1. **Create Deal** — Party A defines assets, amounts, counterparty, time limit, price band tolerance
2. **Deal Room** — Both parties negotiate, share files, confirm terms. Party B sees full terms before connecting wallet.
3. **Lock & Fund** — Both parties deposit into the Safe 2-of-2 vault. Funds locked.
4. **Execute & Complete** — ConfirmDeliveryModal fires for both parties before any release (anti-Bybit defence). Auto-release timer activates.
5. **Dashboard** — Deal history, PDF receipts, operator metrics.

---

## Two products

| Product | What it does | Safe architecture |
|---------|-------------|-------------------|
| **TrustBridge Deal** | Time-limited P2P escrow exchange. Both parties deposit agreed assets. Both must approve release. Timer auto-refunds on expiry. Price band breach protection via Chainlink oracle. | Safe 2-of-2 multisig vault |
| **TrustBridge Vault** | Indefinite shared-control vault for business partners, DAOs, joint ventures, and vesting arrangements. Proportional return, yield integration, milestone-based release. *(Phase 4 — not yet built)* | Safe M-of-N multisig — any threshold |

---

## Architecture overview

```
┌─────────────────────────────────────────────────────────┐
│                    TrustBridge Protocol                  │
├──────────────────────────┬──────────────────────────────┤
│      TrustBridge Deal    │     TrustBridge Vault        │
│      (Phase 2 MVP)       │     (Phase 4 expansion)      │
├──────────────────────────┴──────────────────────────────┤
│                   TrustBridgeDeal.sol                    │
│         Safe SDK · Solidity 0.8.x · Hardhat/Foundry     │
│    OpenZeppelin ReentrancyGuard · SafeERC20 · Ownable   │
├─────────────────────────────────────────────────────────┤
│              Safe 2-of-2 Multisig Vault                 │
│         (every deal = one Safe smart account)           │
├────────────────────────┬────────────────────────────────┤
│     Base (primary)     │     Polygon (secondary)        │
└────────────────────────┴────────────────────────────────┘
```

### Deal state machine

```
PENDING → NEGOTIATING → AGREED → FUNDED → LOCKED → COMPLETED
                                                  ↘ EXPIRED
                                                  ↘ DISPUTED → RESOLVED
```

---

## Core smart contract functions

See [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md) for full specification with parameters and security requirements.

| Function | What it does |
|----------|-------------|
| `createDeal()` | Party A creates deal on-chain. Generates unique dealId. Emits DealCreated. |
| `acceptDeal()` | Party B joins. Verifies caller matches partyB. Starts timer. |
| `confirmTerms()` | Both parties call. Stores termsHash on-chain when both confirmed. |
| `depositAssets()` | Accepts exact agreed amounts via SafeERC20. Auto-locks when both deposited. |
| `releaseFunds()` | Requires both signatures. State → COMPLETED first. Atomic transfer. Fee deducted. |
| `claimRefund()` | Callable after expiry by anyone. Returns both deposits. Zero fee. |
| `cancelDeal()` | Party A only, before LOCKED. Returns deposit if applicable. |
| `openDispute()` | Either party post-LOCKED. Requires dispute deposit. Freezes funds. |
| `resolveDispute()` | Admin only (v1). Kleros arbitration (v2). |

---

## Security architecture

See [SECURITY.md](SECURITY.md) for the full threat model and required defences.

### Immutable design decisions

These cannot change in any version of TrustBridge for any user, operator, or circumstance:

1. **2-of-2 multisig** — cannot be waived. Neither party can act alone.
2. **Time-lock auto-refund** — no admin override. The refund is unstoppable by design.
3. **Non-custodial** — TrustBridge cannot access, freeze, or move user funds. Ever.
4. **Fee on completion only** — never on creation, expiry, or cancellation.
5. **ConfirmDeliveryModal** — mandatory before every fund release. Direct response to the Bybit $1.5B UI spoofing attack (February 2025).
6. **Chainlink oracle** — for all on-chain price verification. Never CoinGecko.
7. **Checks-effects-interactions** — state updated before all transfers. Non-negotiable.

---

## Plan tiers

| | Free | Pro ($99/mo) | Enterprise ($299/mo) |
|--|------|-------------|----------------------|
| **Fee** | 1% max $10 | 0.5% max $25 | 0.25% |
| **Deal limit** | $5,000 | $500,000 | Unlimited |
| **Active deals** | 5 | Unlimited | Unlimited |
| **Time limits** | 30min – 7d fixed | Custom | Custom |
| **Price band** | ±15% fixed | Configurable | Configurable + OFF |
| **Auto-release** | Mandatory 15min | Mandatory 15min | Toggleable |
| **White-label** | ✗ | ✗ | ✓ |
| **API access** | ✗ | ✗ | ✓ |
| **Party B fee** | Always free | Always free | Always free |

---

## Fee routing (on-chain)

Fee is charged at `releaseFunds()` only — never on creation, expiry, or cancellation.

- **White-label split:** 70% operator wallet / 30% TrustBridge wallet — automatic on-chain split
- **Fee config is immutable** at deployment — set in constructor, verifiable on-chain
- **Expired / cancelled deals:** zero fee, always

---

## Development roadmap

| Phase | Timeline | Key deliverables |
|-------|----------|-----------------|
| **1 — Validate** | Complete | Prototype live, specification complete, domain registered, grant application submitted |
| **2 — Build** | Months 2–4 | Smart contracts on Safe SDK, security audit, Base Sepolia testnet, 100 test deals |
| **3 — Launch** | Months 4–6 | Mainnet live, first white-label operator, seed raise |
| **4 — Expand** | Months 7–12 | TrustBridge Vault, M-of-N multi-party, yield integration, Series A preparation |

---

## Repository contents

| File | Contents |
|------|----------|
| [README.md](README.md) | This file — product overview and architecture |
| [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md) | Full function specification with parameters and security requirements |
| [SECURITY.md](SECURITY.md) | Threat model, attack vectors, and required defences |
| [KNOWN_LIMITATIONS.md](KNOWN_LIMITATIONS.md) | Prototype limitations and pre-production requirements |
| [LICENSE](LICENSE) | MIT licence — applies to specification documents in this repository |

---

## What is NOT in this repository

The React/JSX prototype codebase (22 components, full deal flow) is maintained in a private repository. It is available for review by serious technical co-founder candidates and grant reviewers under NDA.

The smart contracts referenced in this specification do not yet exist. This repository contains the specification from which they will be built.

---

## Contact

**Craig Evans** — Founder, TrustBridge Protocol
Dubai, UAE · May 2026

- Web: [trustbridgeprotocol.xyz](https://trustbridgeprotocol.xyz)
- Prototype: [trust-bridge-secure.base44.app](https://trust-bridge-secure.base44.app)
- Grant application: Safe Foundation Spontaneous Grant (in progress)

---

> *TrustBridge Protocol is designed for Safe-powered smart account architecture. Current build is a prototype — not audited production contracts. Language such as "designed to be non-custodial" and "Safe-powered target architecture" reflects intended production design, not current deployed state.*
