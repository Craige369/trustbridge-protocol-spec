# TrustBridge Protocol

**Non-custodial smart contract escrow infrastructure — Safe-powered · Base + Polygon · Dubai, UAE**

> "Trust without a middleman."

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Status: Prototype](https://img.shields.io/badge/Status-Prototype-orange.svg)]()
[![Chain: Base](https://img.shields.io/badge/Chain-Base-blue.svg)]()
[![Powered by Safe](https://img.shields.io/badge/Powered%20by-Safe-green.svg)]()

---

## What is TrustBridge Protocol?

TrustBridge Protocol is a non-custodial escrow infrastructure product built on Safe (Gnosis Safe) 2-of-2 multisig smart account architecture. It enables two parties who do not trust each other to lock, negotiate, and exchange virtual assets with no custodian, no intermediary, and no single point of failure.

Every TrustBridge deal creates a Safe smart account. Neither party nor TrustBridge ever holds, controls, or has discretion over user funds at any point.

**Current status:** React/JSX prototype complete (22 components, full 5-screen deal flow). Smart contracts not yet deployed. Live prototype at [app.trustbridgeprotocol.xyz](https://app.trustbridgeprotocol.xyz).

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

### Deal state machine (v1)

```
PENDING → NEGOTIATING → AGREED → FUNDED → LOCKED → COMPLETED
                                                  ↘ EXPIRED (auto-refund)
                                                  ↘ CANCELLED
```

> **Note on dispute resolution:** TrustBridge Protocol v1 does not include a discretionary dispute resolution mechanism. This is a deliberate architectural decision that preserves the clean non-custodial argument. All deal outcomes are governed entirely by smart contract logic. A "Raise a Concern" flow exists in the UI as a timestamped record only — it does not affect vault state or fund movement. Kleros decentralised arbitration is planned for v2.

---

## Core smart contract functions (v1 — 7 functions)

See [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md) for full specification with parameters and security requirements.

| Function | What it does |
|----------|-------------|
| `createDeal()` | Party A creates deal on-chain. Generates unique dealId. Enforces token whitelist and $10 minimum. Emits DealCreated. |
| `acceptDeal()` | Party B joins. Verifies caller matches partyB address exactly. Starts timer. |
| `confirmTerms()` | Both parties call. Stores termsHash on-chain when both confirmed. |
| `depositAssets()` | Accepts exact agreed amounts via SafeERC20. Auto-locks when both deposited. |
| `releaseFunds()` | Requires both signatures. State → COMPLETED first. Atomic transfer. Fee deducted (rounded down, favour user). |
| `claimRefund()` | Permissionless after expiry. Returns both deposits. Zero fee always. |
| `cancelDeal()` | Party A only, before LOCKED. Returns deposit if applicable. Zero fee. |

---

## Security architecture

See [SECURITY.md](SECURITY.md) for the full threat model and required defences.

### Immutable design decisions

These decisions are permanent and cannot be changed in any version of TrustBridge:

| Decision | Why it cannot change |
|----------|---------------------|
| 2-of-2 multisig — cannot be waived | Entire trust model depends on neither party acting alone |
| Time-lock auto-refund — no admin override | Admin override creates centralised attack surface |
| Non-custodial — TrustBridge cannot access user funds | Changes regulatory classification. Core value proposition. |
| Fee on completion only — never on expiry or cancel | Aligns TrustBridge incentives with users |
| ConfirmDeliveryModal — mandatory, never removable | Direct response to Bybit $1.5B UI spoofing attack (Feb 2025) |
| Chainlink oracle only — never CoinGecko for on-chain logic | CoinGecko manipulable. Chainlink verifiable on-chain. |
| Checks-effects-interactions — state before all transfers | Protection against reentrancy attacks |
| No discretionary dispute resolution in v1 | Preserves clean non-custodial VARA argument |

---

## Plan tiers

| Parameter | Free | Pro | Enterprise |
|-----------|------|-----|------------|
| Max deal value | $5,000 | $500,000 | Unlimited |
| Max timer | 7 days | 30 days | 365 days |
| Platform fee | 1% max $10 | 0.5% max $25 | 0.25% |
| Single-party auto-release | 15 minutes | 15 minutes | Configurable — default 2 hours |
| Price band | ±15% fixed | 1–50% custom | Configurable, or OFF |
| Same-asset deals | Not permitted | Permitted | Permitted |
| Party B fee | Always free | Always free | Always free |
| Approved tokens | USDC, USDT, ETH, WBTC, DAI, WETH | Same | Same + operator additions |
| Minimum deal value | $10 USD | $10 USD | $10 USD |

---

## AML/CFT thresholds (UAE VARA)

| Threshold | Obligation |
|-----------|-----------|
| AED 3,500 (~$950 USD) | FATF Travel Rule — originator and beneficiary information required |
| AED 55,000 (~$15,000 USD) | Enhanced CDD — full identity verification required |

Travel Rule field included in database schema from mainnet day one.

---

## Known limitations

See [KNOWN_LIMITATIONS.md](KNOWN_LIMITATIONS.md) for the full register of prototype limitations, pre-production requirements, and resolved bugs.

**TrustBridge publishes this register openly to demonstrate technical awareness and professional discipline.**

---

## Roadmap

| Phase | Status | Milestone |
|-------|--------|-----------|
| Phase 1 — Validate | ✅ Complete | Prototype built, market validated |
| Phase 2 — Build | 🔄 In progress | Safe SDK contracts, audit, Base Sepolia testnet |
| Phase 3 — Launch | ⏳ Planned | Mainnet, white-label operators, $10K MRR |
| Phase 4 — Expand | ⏳ Planned | TrustBridge Vault, M-of-N, Series A |

---

## Repository structure

This is the **public specification repository**. It contains:

- `README.md` — this file
- `SMART_CONTRACT_SPEC.md` — full function specification with parameters and security requirements
- `SECURITY.md` — security architecture, threat model, and immutable defences
- `KNOWN_LIMITATIONS.md` — prototype limitations, pre-production requirements, resolved bugs
- `LICENSE` — MIT licence

The private development repository (`github.com/Craige369/trustbridge`) contains the React/JSX prototype codebase and is available for review under NDA.

---

## Contact

**Craig Evans** — Founder, TrustBridge Protocol  
Dubai, UAE · May 2026  
[trustbridgeprotocol.xyz](https://trustbridgeprotocol.xyz)  
craig@trustbridgeprotocol.xyz

**Responsible disclosure:** craig@trustbridgeprotocol.xyz  
Smart contracts are not yet deployed. No live funds are at risk.
