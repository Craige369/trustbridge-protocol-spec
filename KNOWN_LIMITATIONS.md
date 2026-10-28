# Known Limitations Register

**TrustBridge Protocol — Prototype Build — v3 — October 2026**

> This document catalogues all known limitations, prototype-only behaviours, and pre-production requirements in the current TrustBridge build. It is published openly to demonstrate technical awareness and professional discipline.
>
> **v3 update:** adds the four-role stress test findings (Free user, Pro user, Enterprise OTC operator, bad actor): requirements R-10 to R-15, confirmed plan tier limits, and seven further UI fixes. Aligned with [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md) v1.1: 7 core functions plus a single-party auto-release path, no dispute resolution in v1, and six architecture decisions to be confirmed before build (R-16).

**Scope:** the React/JSX prototype. This register does not cover production smart contracts, which have not yet been written.

---

## How to read this document

| Category | What it means |
|----------|--------------|
| **Prototype infrastructure** | Deliberate simplifications for rapid prototyping. Expected. Will be replaced in Phase 2. |
| **Known UI limitations** | Present in the current prototype. Known and understood. Do not affect the core deal flow. |
| **Pre-production requirements** | Not yet built. Must exist before any mainnet deployment with real funds. |
| **Plan tier limits** | Confirmed limits, enforced at `createDeal()` and in the UI. |
| **Confirmed resolved** | Bugs identified and fixed during development. |
| **Immutable decisions** | Not limitations — permanent architectural decisions that will never change. |

---

## Section 1 — Prototype infrastructure

| Ref | Component | Limitation | Production replacement |
|-----|-----------|-----------|----------------------|
| P-01 | `dealStore.js` | Prototype state only — not an authenticated, persistent backend. | Supabase or PlanetScale with row-level security. |
| P-02 | Dashboard, CreateDeal | Hardcoded deal ID `#0042` throughout. | Server-generated unique IDs from contract events. |
| P-03 | `PlanSwitcher.jsx` | Prototype-only plan switching — does not exist in production. | Remove entirely. Plan from authenticated user account. |
| P-04 | `Dashboard.jsx` | Simulated wallet connection via local React state. | Real wallet via Wagmi + viem. MetaMask and WalletConnect. |
| P-05 | Multiple screens | Hardcoded mock wallet addresses. | Real addresses from connected wallet provider. |
| P-06 | Share links | Invite link hardcoded to non-functional URL. | Real single-use invite links with cryptographic token. |

---

## Section 2 — Known UI limitations

| Ref | Component | Limitation | Resolution target |
|-----|-----------|-----------|------------------|
| U-01 | `PriceBandLiveIndicator` | Always shows ETH/USDT regardless of deal assets. Simulated movement hardcoded. | Before operator onboarding. Live CoinGecko for display; Chainlink for contract logic. |
| U-02 | `Dashboard.jsx` | Party B completed deals not shown as "Received". | Mainnet launch. Requires backend persistence. |
| U-03 | Multiple screens | Progress tracker only on Create Deal success panel. | Before operator onboarding. |
| U-04 | `LockFund.jsx` | Escrow Summary header still shows `Deal #0042`. | Before first operator demo. Resolves with real deal IDs. |
| U-05 | `DealRoom.jsx` | Chat not stored in an encrypted backend. | Mainnet launch. |
| U-06 | Multiple screens | No wallet-connected lock state on Deal Room, Lock & Fund, Execute. | Before operator onboarding. |
| U-07 | `Dashboard.jsx` | No search, filter or pagination. Unusable above ~10 active deals for Pro users. | Before Pro plan operator onboarding. Server-side search and filter. |
| U-08 | `Execute.jsx` | Legacy dispute screens remain in the codebase, dormant and greyed out. "Raise a Concern" is the active v1 feature. | Kept dormant until Kleros integration in v2. |

---

## Section 3 — Pre-production requirements

| Ref | Area | Requirement | Phase |
|-----|------|------------|-------|
| R-01 | Smart contracts | `TrustBridgeDeal.sol` does not yet exist. The 7 core v1 functions and the single-party auto-release path must be written, tested and audited. See [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md). | Phase 2 |
| R-02 | Security audit | No audit conducted. Independent professional audit mandatory before mainnet. Budget: $15,000–$25,000. | Phase 2, before mainnet |
| R-03 | Price oracle | Chainlink integration not yet built, including staleness handling. CoinGecko for UI display only. | Phase 2 |
| R-04 | Real wallet integration | Wagmi + viem for wallet connection and transaction signing. No real transactions in prototype. | Phase 2 |
| R-05 | Backend persistence | Supabase or PlanetScale with row-level security. FATF Travel Rule field in schema from day one — **populated for all deals above AED 3,500 (~$950 USD)**. Admin audit log as append-only table. | Phase 2 |
| R-06 | Email notifications | Deal event notifications. No notification system in prototype. | Phase 2 |
| R-07 | Domain and SSL | `trustbridgeprotocol.xyz` registered May 2026. Typo domains before mainnet. | trustbridgeprotocol.xyz complete |
| R-08 | KYC / AML thresholds | **Two separate VARA thresholds:** (1) FATF Travel Rule: **AED 3,500 (~$950 USD)** — originator name, wallet address and beneficiary info required above this level; Free plan deals ($5,000) can exceed it, so it applies from mainnet day one. (2) CDD occasional transaction: **AED 55,000 (~$15,000 USD)** — enhanced due diligence above this level. Where the CDD check is enforced (on-chain or at the operator/application layer) is architecture decision D-05. | Phase 3 — specified before Phase 2 |
| R-09 | VARA regulatory classification | Non-custodial DLT infrastructure argument — not a VASP. Pre-application VARA enquiry and formal UAE legal opinion required before mainnet. | Before mainnet |
| R-10 | Enterprise operator onboarding | No onboarding flow exists. Operators need: domain configuration, fee split setup, brand configuration, an operator dashboard with revenue metrics, and an Operator Agreement defining the liability split. | Spec: Phase 2. Build: Phase 3 |
| R-11 | Approved token whitelist | USDC, USDT, ETH, WBTC, DAI, WETH, enforced at `createDeal()`. UI shows a verified badge for approved tokens and a warning for unverified addresses. Enterprise token additions via a governance process (to be defined). | Specified. Build: Phase 2 |
| R-12 | Minimum deal value | $10 USD equivalent enforced at `createDeal()` against Chainlink. Prevents gas griefing. | Specified. Build: Phase 2 |
| R-13 | Fee precision at large deal sizes | Always round down in favour of the user; fee taken from deal proceeds; wei precision; misconfigured fee wallet reverts rather than failing silently. | Specified. Build: Phase 2 |
| R-14 | Renegotiation connection loss | If a renegotiation window expires without acceptance or decline, the deal cancels and both deposits return. No ambiguous mid-renegotiation state. | Specified. Build: Phase 2 |
| R-15 | Public operator registry | A public list of approved white-label operators on trustbridgeprotocol.xyz, protecting users against operator impersonation. | Phase 3 |
| R-16 | Architecture decisions | Six decisions confirmed before build: custody model (D-01), Party B binding (D-02), single-party auto-release mechanism (D-03), fee tier source (D-04), compliance gate placement (D-05), price band breach handling (D-06). See [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md). | Before Phase 2 M1 |

---

## Section 4 — Plan tier limits

| Parameter | Free | Pro | Enterprise |
|-----------|------|-----|-----------|
| Maximum deal value | $5,000 | $500,000 | Unlimited |
| Maximum timer | 7 days | 30 days | 365 days |
| Long-duration acknowledgment | N/A — 7-day max | Deals over 7 days | Deals over 30 days |
| Single-party auto-release window | 15 minutes | 15 minutes | Configurable — default 2 hours |
| Same-asset deals | Not permitted | Permitted, with education note | Permitted, with education note |
| Minimum deal value | $10 USD equivalent | $10 USD equivalent | $10 USD equivalent |
| Approved tokens | USDC, USDT, ETH, WBTC, DAI, WETH | Same | Same + operator additions |

---

## Section 5 — Confirmed resolved

| Ref | Component | Bug description | Status |
|-----|-----------|----------------|--------|
| F-01 | `TimeLimitSelector.jsx` | Timer showed 24h regardless of chip selected. | Fixed May 2026 |
| F-02 | `dealStore.js` | Adjust Deal showed original amounts, not the latest version. | Fixed May 2026 |
| F-03 | `LockFund.jsx` / `BreachResponseFlow.jsx` | Black screen on price band breach during renegotiation. | Fixed May 2026 |
| F-04 | `CreateDeal.jsx` | ExtremeConditions acknowledgment didn't unlock Create Deal. | Fixed May 2026 |
| F-05 | `LockFund.jsx` | CXL reference changed on every checkbox tick. | Fixed May 2026 |
| F-06 | `CreateDeal.jsx` / `TimeLimitSelector.jsx` | Custom time validation accepted exactly 24h and 60min. | Fixed May 2026 |
| F-07 | `LockFund.jsx` / `Execute.jsx` | Renegotiated amounts not flowing downstream. | Fixed May 2026 |
| F-08 | `LockFund.jsx` / `Execute.jsx` | Timer and price band not freezing on cancellation. | Fixed May 2026 |
| F-09 | `DealSuccessPanel.jsx` / `LockFund.jsx` | Deal expiry countdown dropped the days component. | Fixed May 2026 |
| F-10 | Dashboard / CreateDeal | No wrong-network detection. Base network check and warning banner added. | Fixed May 2026 |
| F-11 | `Execute.jsx` | No auto-release explanation. Countdown and plain-English banner added. | Fixed May 2026 |
| F-12 | `CreateDeal.jsx` | Same-asset deals allowed on Free with a meaningless price band. Error block and education note added. | Fixed May 2026 |
| F-13 | `DealRoom.jsx` | No Party B join notification. Toast and system chat message added. | Fixed May 2026 |
| F-14 | `TimeLimitSelector.jsx` | No tier-based timer maximums. Free 7d / Pro 30d / Enterprise 365d enforced. | Fixed May 2026 |
| F-15 | `CreateDeal.jsx` | No token whitelist UI. Verified badges and unverified warning added. | Fixed May 2026 |
| F-16 | `CreateDeal.jsx` | No minimum deal value. $10 minimum added. | Fixed May 2026 |

---

## Section 6 — Immutable design decisions

| Ref | Decision | Rationale |
|-----|----------|-----------|
| I-01 | 2-of-2 multisig — cannot be waived for any user, deal or circumstance | No party can move funds at will. The only one-sided outcomes are rule-based (auto-release after the window, refund at expiry), agreed at deal creation and enforced by contract logic. |
| I-02 | Time-lock auto-refund — no admin override | Admin override creates a centralised attack surface. Refund must be unstoppable. |
| I-03 | Non-custodial — TrustBridge cannot access, freeze or move user funds | Changes regulatory classification and destroys the core value proposition. |
| I-04 | Fee on completion only — never on creation, expiry or cancellation | Aligns TrustBridge incentives with users. Failed deals never cost platform fees. |
| I-05 | ConfirmDeliveryModal — mandatory, both parties, every deal, never removable | Direct architectural response to the Bybit $1.5B UI spoofing attack (February 2025). |
| I-06 | Chainlink oracle — on-chain price verification only, never CoinGecko | CoinGecko can be manipulated. Chainlink is manipulation-resistant and verifiable. |
| I-07 | Checks-effects-interactions — state updated before all transfers | Protection against reentrancy. Non-negotiable on all fund-moving functions. |
| I-08 | No discretionary dispute resolution in v1 | Preserves the non-custodial position under UAE VARA. Kleros decentralised arbitration in v2. |

---

## For grant reviewers and developers

Sections 1 and 2 are expected prototype behaviour. Section 3 is the Phase 2 scope, with R-16 listing the decisions settled with the development team before M1. Section 5 shows active development discipline. Section 6 confirms the security decisions are fixed. The stress test found no fatal architectural problems — the gaps were specification or UI issues.

---

*Document v3 — October 2026 · Craig Evans (founder) · trustbridgeprotocol.xyz*  
*Responsible disclosure: craig@trustbridgeprotocol.xyz — smart contracts not yet deployed, no live funds at risk*
