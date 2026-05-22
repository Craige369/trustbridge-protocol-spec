# Known Limitations Register

**TrustBridge Protocol — Prototype Build, May 2026 — v2**

> This document catalogues all known limitations, prototype-only behaviours, and pre-production requirements in the current TrustBridge build. Published openly to demonstrate technical awareness and professional discipline. v2 update: VARA Travel Rule threshold corrected to AED 3,500 and regulatory classification item added following full 13-document VARA rulebook review.

---

## How to read this document

| Category | What it means |
|----------|--------------|
| **Prototype infrastructure** | Deliberate simplifications for rapid prototyping. Expected. Will be replaced in Phase 2. |
| **Known UI limitations** | Present in current prototype. Known and understood. Do not affect core deal flow. |
| **Pre-production requirements** | Not yet built. Must exist before any mainnet deployment with real funds. |
| **Confirmed resolved** | Bugs identified and fixed during May 2026 build session. |
| **Immutable decisions** | Not limitations — permanent architectural decisions that will never change. |

---

## Section 1 — Prototype infrastructure

| Ref | Component | Limitation | Production replacement |
|-----|-----------|-----------|----------------------|
| P-01 | `dealStore.js` | In-memory state only. Page refresh wipes everything. | Supabase or PlanetScale with row-level security. |
| P-02 | Dashboard, CreateDeal | Hardcoded deal ID `#0042` throughout. | Server-generated unique IDs from contract events. |
| P-03 | `PlanSwitcher.jsx` | Prototype-only plan switching — does not exist in production. | Remove entirely. Plan from authenticated user account. |
| P-04 | `Dashboard.jsx` | Simulated wallet connection via local React state. | Real wallet via Wagmi + viem. MetaMask and WalletConnect. |
| P-05 | Multiple screens | Hardcoded mock wallet addresses. | Real addresses from connected wallet provider. |
| P-06 | Share links | Invite link hardcoded to non-functional URL. | Real invite links generated server-side with cryptographic token. |

---

## Section 2 — Known UI limitations

| Ref | Component | Limitation | Resolution target |
|-----|-----------|-----------|------------------|
| U-01 | `PriceBandLiveIndicator` | Always shows ETH/USDT regardless of deal assets. Simulated +10.4% movement hardcoded. | Before operator onboarding. Live CoinGecko for display; Chainlink for contract logic. |
| U-02 | `Dashboard.jsx` | Party B completed deals not shown as "Received". | Mainnet launch. Requires backend persistence. |
| U-03 | Multiple screens | Progress tracker only on Create Deal success panel. | Before operator onboarding. StepTracker on all five screens. |
| U-04 | `LockFund.jsx` | Escrow Summary header still shows `Deal #0042`. | Before first real operator demo. Resolves with real deal IDs. |
| U-05 | `DealRoom.jsx` | Chat messages not persisted on page refresh. | Mainnet launch. Requires encrypted backend storage. |
| U-06 | Multiple screens | No wallet-connected lock state on Deal Room, Lock & Fund, Execute. | Before operator onboarding. Wallet guard on all screens. |

---

## Section 3 — Pre-production requirements

| Ref | Area | Requirement | Phase |
|-----|------|------------|-------|
| R-01 | Smart contracts | `TrustBridgeDeal.sol` does not yet exist. All 9 core functions must be written, tested, and audited. See [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md). | Phase 2 |
| R-02 | Security audit | No audit conducted. Professional audit (CertiK or Trail of Bits) mandatory. Budget: $15,000–$25,000. | Phase 2, before mainnet |
| R-03 | Price oracle | Chainlink integration not yet built. CoinGecko for UI display only. | Phase 2 |
| R-04 | Real wallet integration | Wagmi + viem for wallet connection and transaction signing. No real transactions in prototype. | Phase 2 |
| R-05 | Backend persistence | Supabase or PlanetScale with row-level security. FATF Travel Rule field in schema from day one — **populated for all deals above AED 3,500 (~$950 USD)**. Admin audit log as append-only table. | Phase 2 |
| R-06 | Email notifications | SendGrid for deal event notifications. No notification system in prototype. | Phase 2 |
| R-07 | Domain and SSL | `trustbridgeprotocol.xyz` registered May 2026. SSL via Netlify. Typo domains before mainnet. | trustbridgeprotocol.xyz complete |
| R-08 | KYC / AML thresholds | **Two separate VARA thresholds:** (1) FATF Travel Rule: **AED 3,500 (~$950 USD)**. Originator name, wallet address, and beneficiary info required above this threshold. Schema Travel Rule field must be populated at this level. Free plan deals ($5,000) exceed this threshold — Travel Rule applies from mainnet day one. (2) CDD occasional transaction: AED 55,000 (~$15,000 USD). Enhanced due diligence above this level. These are separate obligations. | Phase 3, before Pro plan mainnet |
| R-09 | VARA regulatory classification | Full 13-document VARA rulebook review completed May 2026. TrustBridge's strongest argument: non-custodial DLT infrastructure — not a VASP. Neither the protocol nor any single party holds or controls user assets. Three actions required: (1) Submit pre-application VARA enquiry (vara.ae/en/contact) — free. (2) Obtain formal UAE Web3 lawyer legal opinion on non-custodial protocol exemption. Budget AED 8,000–15,000. (3) Consider jurisdictional structure: UAE Free Zone (short term) → Cyprus holding company (medium term, founder is Cypriot citizen) → BVI/Cayman (long term/Series A). | Before mainnet |

---

## Section 4 — Confirmed resolved (May 2026)

| Ref | Component | Bug description | Status |
|-----|-----------|----------------|--------|
| F-01 | `TimeLimitSelector.jsx` | Timer showed 24h regardless of chip selected. | Fixed May 2026 |
| F-02 | `dealStore.js` | Adjust Deal showed original amounts not latest version. | Fixed May 2026 |
| F-03 | `LockFund.jsx` / `BreachResponseFlow.jsx` | Black screen on price band breach during renegotiation. | Fixed May 2026 |
| F-04 | `CreateDeal.jsx` | ExtremeConditions acknowledgment didn't unlock Create Deal. | Fixed May 2026 |
| F-05 | `LockFund.jsx` | CXL reference changed on every checkbox tick. | Fixed May 2026 — `useRef` applied |
| F-06 | `CreateDeal.jsx` / `TimeLimitSelector.jsx` | Custom time validation accepted exactly 24h and 60min. | Fixed May 2026 — `>= 24` and `>= 60` |
| F-07 | `LockFund.jsx` / `Execute.jsx` | Renegotiated amounts not flowing to Escrow Summary, Cancel Modal, Cancelled screen. | Fixed May 2026 |
| F-08 | `LockFund.jsx` / `Execute.jsx` | Timer and price band not freezing on cancellation. | Fixed May 2026 — `dealEnded` flag |
| F-09 | `DealSuccessPanel.jsx` / `LockFund.jsx` | Deal expiry countdown dropped days component. | Fixed May 2026 — days now render as `Xd HH:MM:SS` |

---

## Section 5 — Immutable design decisions

| Ref | Decision | Rationale |
|-----|----------|-----------|
| I-01 | 2-of-2 multisig — cannot be waived for any user, deal, or circumstance | The entire trust model depends on neither party acting alone. |
| I-02 | Time-lock auto-refund — no admin override | Admin override creates a centralised attack surface. Refund must be unstoppable. |
| I-03 | Non-custodial — TrustBridge cannot access, freeze, or move user funds | Changes regulatory classification and destroys the core value proposition. |
| I-04 | Fee on completion only — never on creation, expiry, or cancellation | Aligns TrustBridge incentives with users. Failed deals never cost platform fees. |
| I-05 | ConfirmDeliveryModal — mandatory, both parties, every deal, never removable | Direct architectural response to the Bybit $1.5B UI spoofing attack (February 2025). |
| I-06 | Chainlink oracle — on-chain price verification only, never CoinGecko | CoinGecko can be manipulated. Chainlink is manipulation-resistant and verifiable. |
| I-07 | Checks-effects-interactions — state updated before all transfers | Protection against reentrancy attacks. Non-negotiable on all fund-moving functions. |

---

## For grant reviewers

Sections 1 and 2 are expected prototype behaviour. Section 3 is the Phase 2 grant scope — R-09 is new in v2 and reflects a full 13-document VARA rulebook review. The Travel Rule threshold correction in R-08 (AED 3,500, not AED 55,000) reflects confirmed VARA Compliance and Risk Management Rulebook requirements. Section 4 demonstrates active development discipline. Section 5 confirms security decisions are fixed.

---

*Document: v2 — May 2026 · Craig Evans (founder) · trustbridgeprotocol.xyz*  
*Responsible disclosure: craig@trustbridgeprotocol.xyz — smart contracts not yet deployed, no live funds at risk*
