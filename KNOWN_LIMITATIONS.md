# Known Limitations Register

**TrustBridge Protocol — Prototype Build, May 2026**

> This document catalogues all known limitations, prototype-only behaviours, and pre-production requirements in the current TrustBridge build. It is published openly to demonstrate technical awareness and professional discipline. A prototype that honestly documents its constraints is more credible than one that claims completeness.

---

## How to read this document

| Category | What it means |
|----------|--------------|
| **Prototype infrastructure** | Deliberate simplifications made for rapid prototyping. Expected. Will be replaced in Phase 2. |
| **Known UI limitations** | Present in current prototype. Known and understood. Do not affect the core deal flow. |
| **Pre-production requirements** | Not yet built. Must exist before any mainnet deployment with real funds. |
| **Confirmed resolved** | Bugs identified and fixed during the May 2026 build session. |
| **Immutable decisions** | Not limitations — permanent architectural decisions that will never change. |

---

## Section 1 — Prototype infrastructure

Deliberate simplifications for rapid prototyping. Every item has a defined production replacement.

| Ref | Component | Limitation | Production replacement |
|-----|-----------|-----------|----------------------|
| P-01 | `dealStore.js` | In-memory state only. All deal data, timer state, party confirmations, and adjustment history live in JavaScript memory. Page refresh wipes everything. | Supabase or PlanetScale with row-level security. dealStore getters/setters become API calls. |
| P-02 | Dashboard, CreateDeal | Hardcoded deal ID `#0042` throughout. Every deal shows the same reference number. | Server-generated unique IDs from contract events at deal creation. |
| P-03 | `PlanSwitcher.jsx` | Prototype-only toggle that lets any user switch between Free, Pro, and Enterprise plans instantly. Does not exist in production. | Remove entirely. Plan determined by authenticated user account. |
| P-04 | `Dashboard.jsx` | Simulated wallet connection via local React state toggle. No real Web3 wallet detection. | Real wallet connection via Wagmi + viem. MetaMask and WalletConnect providers. |
| P-05 | Multiple screens | Hardcoded mock wallet addresses used for display. Not connected to real wallets. | Real wallet addresses from connected wallet provider, injected at deal creation. |
| P-06 | Share links | Invite link hardcoded to `trustbridge.app/deal/0042?invite=abc123xyz`. Not functional. | Real invite links generated server-side with unique deal ID and cryptographic invite token. |

---

## Section 2 — Known UI limitations

Present in current prototype. Do not affect core deal flow. Will be resolved before operator onboarding.

| Ref | Component | Limitation | Resolution target |
|-----|-----------|-----------|------------------|
| U-01 | `PriceBandLiveIndicator` | Always shows ETH/USDT as asset pair regardless of deal assets. Simulated rate movement (+10.4%) hardcoded. | Before operator onboarding. Needs live CoinGecko API for display; Chainlink for contract logic. |
| U-02 | `Dashboard.jsx` | Party B completed deals do not appear in Dashboard marked as "Received". Party B has no deal history view. | Mainnet launch. Requires backend persistence (P-01) and wallet-linked deal query. |
| U-03 | Multiple screens | Progress tracker (Created → Deal Room → Lock & Fund → Execute → Complete) only shows on Create Deal success panel. Not on other screens. | Before operator onboarding. StepTracker component to be added to all five screens. |
| U-04 | `LockFund.jsx` | Escrow Summary header still shows `Deal #0042` publicly. Deal IDs removed from Dashboard table but not from Escrow Summary. | Before first real operator demo. Resolves with real deal IDs (P-02). |
| U-05 | `DealRoom.jsx` | Chat messages not persisted. Page refresh clears all chat history and adjustment audit trail. | Mainnet launch. Requires backend message persistence with encrypted storage. |
| U-06 | Multiple screens | No wallet-connected lock state on Deal Room, Lock & Fund, or Execute screens. Dashboard implements two-state wallet model correctly. | Before operator onboarding. Wallet guard component to be added to all screens. |

---

## Section 3 — Pre-production requirements

Must be built before any mainnet deployment with real user funds.

| Ref | Area | Requirement | Phase |
|-----|------|------------|-------|
| R-01 | Smart contracts | `TrustBridgeDeal.sol` does not yet exist. The prototype demonstrates UX and deal flow only. All nine core functions must be written, tested, and audited before real funds are involved. See [SMART_CONTRACT_SPEC.md](SMART_CONTRACT_SPEC.md). | Phase 2 |
| R-02 | Security audit | No security audit has been conducted. A professional audit (CertiK, Trail of Bits, or equivalent) is mandatory before mainnet. Budget: $15,000–$25,000. The prototype will not move to production without a clean audit report. | Phase 2, before mainnet |
| R-03 | Price oracle | Chainlink integration for on-chain price band verification at `releaseFunds()`. Prototype uses simulated rate. CoinGecko is UI display only — never for contract logic. | Phase 2, part of smart contract development |
| R-04 | Real wallet integration | Wagmi + viem for wallet connection, transaction signing, and on-chain interaction. MetaMask and WalletConnect providers. Prototype simulates wallet connection — no real transactions sent. | Phase 2, alongside smart contract integration |
| R-05 | Backend persistence | Supabase or PlanetScale with row-level security to replace `dealStore.js`. Real-time deal state, chat storage, audit trail, party confirmation tracking. FATF Travel Rule field in schema from day one. Admin audit log as append-only table. | Phase 2, block on testnet deployment |
| R-06 | Email notifications | SendGrid integration for deal event notifications: counterparty joined, deposit made, timer expiring (24h warning), deal completed, deal expired. No notification system exists in prototype. | Phase 2, before operator onboarding |
| R-07 | Domain and SSL | `trustbridgeprotocol.xyz` registered May 2026. SSL configured via Netlify. Typo domains (`trustbr1dge.xyz` etc.) to be registered before mainnet to prevent phishing. | trustbridgeprotocol.xyz complete. Typo domains: Phase 3 |
| R-08 | KYC threshold | No KYC process implemented. UAE VARA requires KYC at AED 55,000 (~$15,000 USD). Free plan limit ($5,000) is well below threshold. Pro plan deals can approach the limit. KYC process design required before Pro plan mainnet. | Phase 3, before Pro plan mainnet |

---

## Section 4 — Confirmed resolved (May 2026 build session)

All bugs fixed and verified during the May 2026 build session.

| Ref | Component | Bug description | Fix applied |
|-----|-----------|----------------|------------|
| F-01 | `TimeLimitSelector.jsx` | Timer showed 24h regardless of chip selected. `setDealTimeLimitSeconds()` not firing on chip click. | Fixed: calls on every chip click and custom input change, before `setDealCreatedAt()`. |
| F-02 | `dealStore.js` | Adjust Deal pre-populated form with original amounts instead of latest adjusted version. | Fixed: `setAdjustData()` overwrites completely on each submission. Snapshots previous version for audit trail. |
| F-03 | `LockFund.jsx` / `BreachResponseFlow.jsx` | Black screen when price band breach triggered during renegotiation. Parent `navigate()` fired while modal was open. | Fixed: `breachActive` guard blocks all parent navigation. Race-condition protection inside `setTimeout`. |
| F-04 | `CreateDeal.jsx` | ExtremeConditions acknowledgment did not unlock Create Deal button. | Fixed: `priceAcknowledged = customPriceAcknowledged \|\| extremeAcknowledged`. |
| F-05 | `LockFund.jsx` | CXL reference number changed on every checkbox tick. `cancelRef` declared as plain `const`, recalculating on every render. | Fixed: `useRef` captures value once at mount. |
| F-06 | `CreateDeal.jsx` / `TimeLimitSelector.jsx` | Custom time validation accepted exactly 24h and 60min as valid. | Fixed: `>= 24` hours and `>= 60` minutes. Days remain `> 365`. |
| F-07 | `LockFund.jsx` / `Execute.jsx` | Renegotiated deal amounts not flowing to Escrow Summary, Final Cancel Modal, or Cancelled screen after breach renegotiation. | Fixed: `dealStore.setAdjustData()` called immediately on renegotiation acceptance before any state change. |
| F-08 | `LockFund.jsx` / `Execute.jsx` | Deal countdown timer and price band indicator continued updating after cancellation. | Fixed: `dealEnded` state flag wired to timer `useEffect` and `PriceBandLiveIndicator` frozen prop. |
| F-09 | `DealSuccessPanel.jsx` / `LockFund.jsx` | Deal expiry countdown dropped the days component. 6-day 9-hour deal showed as `09:00:00`. | Fixed: `formatCountdown()` updated to handle days. Renders as `6d 09:00:00`. |

---

## Section 5 — Immutable design decisions

Not limitations. Permanent architectural decisions that will never change.

| Ref | Decision | Rationale |
|-----|----------|-----------|
| I-01 | 2-of-2 multisig — cannot be waived for any user, deal, or circumstance | The entire trust model depends on neither party acting alone. |
| I-02 | Time-lock auto-refund — no admin override | Admin override creates a centralised attack surface. The refund must be unstoppable. |
| I-03 | Non-custodial — TrustBridge cannot access, freeze, or move user funds | Changes regulatory classification and destroys the core value proposition. |
| I-04 | Fee on completion only — never on creation, expiry, or cancellation | Aligns TrustBridge incentives with users. Failed deals never cost platform fees. |
| I-05 | ConfirmDeliveryModal — mandatory, both parties, every deal, never removable | Direct architectural response to the Bybit $1.5B UI spoofing attack (February 2025). |
| I-06 | Chainlink oracle — on-chain price verification only, never CoinGecko | CoinGecko can be manipulated. Chainlink is manipulation-resistant and verifiable. |
| I-07 | Checks-effects-interactions — state updated before all transfers | Protection against reentrancy attacks. Non-negotiable on all fund-moving functions. |

---

## For grant reviewers

The items in Sections 1 and 2 are expected in any working prototype and represent no unusual risk. The items in Section 3 define the exact scope of Phase 2 (Build) — precisely what the Safe Foundation Spontaneous Grant is intended to fund. Section 4 demonstrates that active bug resolution is ongoing and disciplined. Section 5 confirms that critical security decisions are fixed by design.

TrustBridge is not presented as a finished product. It is presented as a fully specified, prototype-validated, architecturally sound foundation that requires smart contract development, security audit, and backend infrastructure to become production-ready. The grant funds that gap.

---

*Document maintained by: Craig Evans (founder)*
*Last updated: May 2026*
*Next update: when new limitations are identified or resolved*
*Contact: craig@trustbridgeprotocol.xyz*
