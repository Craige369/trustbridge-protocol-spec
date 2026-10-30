# TrustBridge Protocol — Smart Contract Specification

**Version:** 1.2 — October 2026  
**Status:** Specification — contracts not yet written, audited or deployed  
**Target chain:** Base (primary). Other EVM chains added by configuration (see D-09)  
**Architecture:** One Safe 2-of-2 smart account per deal, with a TrustBridge module and guard

---

## Overview

TrustBridge Protocol v1 is designed to provide non-custodial deal-room escrow. Each deal gets its own Safe smart account, owned by the two parties. A small TrustBridge module can move funds out of that Safe in only a handful of rule-based ways, and a TrustBridge guard blocks every other movement. No TrustBridge-controlled key, and no single party, can move user funds outside the rules both parties agreed at deal creation.

### What changed in v1.2

All architecture decisions (D-01 to D-09) are now confirmed and specified:
- **D-01** One Safe per deal + narrowly scoped module + guard
- **D-02** Invite link (front-running resistant) with optional address lock, and Party A confirmation of the counterparty
- **D-03** On-chain delivery confirmations; anyone can trigger auto-release after the window
- **D-04** Fee tier: Free by default, signed Pro voucher, Enterprise operator registry
- **D-05** Compliance checks at the application / operator layer, not in the contracts
- **D-06** Price band breach opens a renegotiation window; unresolved means cancel and refund
- **D-07** Deal expiry changes only when both parties sign
- **D-08** No emergency "break-glass" mode in v1
- **D-09** Base first; any EVM chain added through configuration, not code changes

Also added: mutual cancel at any stage, independent per-party refunds, a pause switch for new deals, and a guarded-launch deal value cap.

**Function count:** 7 core lifecycle functions plus 4 supporting functions.

---

## v1 dispute resolution architecture

TrustBridge Protocol v1 does **not** include `openDispute()` or `resolveDispute()`. A dispute mechanism in which a person decides who receives funds would make TrustBridge a custodian for the duration of the dispute and would compromise the non-custodial position under UAE VARA.

Funds move only in these ways, all governed by contract rules agreed at deal creation:
- **Both parties confirm delivery** → atomic swap
- **One party confirms, the other does not respond within the window** → single-party auto-release
- **Both parties agree to cancel** → each deposit returned
- **Party A cancels before the deal is fully funded** → each deposit returned
- **Expiry, or an unresolved price breach** → `claimRefund()` returns each deposit

"Raise a Concern" exists in the UI as a timestamped record only. It makes no contract call. Kleros decentralised arbitration is on the v2 roadmap.

---

## Contracts

| Contract | Role | Holds funds? |
|----------|------|--------------|
| `TrustBridgeFactory` | Validates and records new deals, deploys each deal's Safe, holds on-chain settings | Never |
| Deal Safe (one per deal) | Official, unmodified Safe proxy. Owners: Party A and Party B. Threshold: 2 | Yes — that deal's deposits only |
| `TrustBridgeModule` | Enabled on every deal Safe. Keeps each deal's record and executes the permitted fund movements | Never |
| `TrustBridgeGuard` | Set on every deal Safe. Blocks all owner-initiated Safe transactions, including changes to owners, threshold, modules or guard | Never |
| Price source adapter | Swappable price component. On Base: Chainlink with staleness checks | Never |

Deal Safes are created through Safe's official proxy factory from the official audited Safe contracts. Each deal's funds are isolated in its own Safe: a problem affecting one deal cannot reach another deal's funds.

ETH is handled as WETH; the application wraps ETH before deposit. The contracts handle ERC-20 tokens only.

### Party wallets

- Parties keep using the **same wallet for every deal** — only the vault (the deal's Safe) is new per deal.
- A party can be a personal wallet **or a smart-contract wallet**, including a company's own Safe multisig (for example, an OTC desk whose directors approve each action internally). Contracts identify parties by the calling address only, never by `tx.origin` or any personal-wallet-only check, and this is covered by tests using a Safe as a party.
- **Payouts always go to the party address recorded in the deal.** There is no alternative payout address in v1 — a redirect field is a common target for manipulation. Parties move funds onward themselves if needed.

---

## Deal state machine (v1)

```
createDeal()           acceptDeal()        confirmCounterparty(yes)    confirmTerms() ×2
PENDING        →       JOINED        →     NEGOTIATING          →      AGREED
                         │ confirmCounterparty(no) → back to PENDING (new invite)

depositAssets()        depositAssets()
AGREED  →  FUNDED_PARTIAL  →  LOCKED

From LOCKED:
  confirmDelivery() ×2 → releaseFunds()        → COMPLETED
  confirmDelivery() ×1 + window → triggerAutoRelease() → COMPLETED
  price breach at release (Pro / Enterprise)  → RENEGOTIATING
  price breach at release (Free)              → CANCELLED (refunds paid)

From RENEGOTIATING:
  amendTerms() by both parties  → LOCKED (confirmations reset)
  window lapses → claimRefund() → CANCELLED

From any state before COMPLETED:
  cancelDeal() by both parties  → CANCELLED
  expiry reached → claimRefund() → EXPIRED

From PENDING, JOINED, NEGOTIATING, AGREED or FUNDED_PARTIAL:
  cancelDeal() by Party A alone → CANCELLED
```

Deal expiry is set at `createDeal()` and changes only through `amendTerms()` signed by both parties (D-07). Expiry always takes precedence: once reached, `claimRefund()` is available whatever else is in progress.

---

## Core functions (7)

### 1. `createDeal()` — Party A

| Parameter | Validation |
|-----------|-----------|
| `assetOffered`, `amountOffered` | Token on this chain's whitelist |
| `assetRequested`, `amountRequested` | Token on this chain's whitelist |
| `expiry` | ≥ now + 15 minutes and ≤ the plan's maximum (Free 7d / Pro 30d / Enterprise 365d) |
| `priceBandBps` | 0 = off (Pro / Enterprise only); Free fixed at ±15% |
| `autoReleaseWindow` | Free / Pro: 15 minutes, mandatory. Enterprise: operator-configurable, can be off |
| `inviteSigner` | Address of a one-time invite key (see D-02) |
| `lockedCounterparty` | Optional. If set, only this address can accept |
| `signers`, `threshold` | v1: 2 signers, threshold 2. Present from day one for M-of-N |
| `proVoucher` | Optional signed voucher proving Pro (D-04) |
| `operatorId` | Optional Enterprise operator (D-04) |

**Checks:** new deals not paused · deal value ≥ $10 and ≤ the guarded-launch cap (via the price source) · fee tier resolved (D-04) · fee cap converted to token units and locked.  
**Effects:** all deal rules locked for the life of the deal · state → PENDING · emit `DealCreated`.

### 2. `acceptDeal()` — Party B

Two ways to join (D-02):
- **Invite link:** the link carries a one-time invite key. Party B's app signs "accept deal X for wallet Y" with it, and the contract checks the signature against `inviteSigner`. Because the signature names Party B's own wallet, someone who copies it from the network cannot use it to join in their place. Single use.
- **Address lock:** if `lockedCounterparty` is set, the caller must be that address.

**Effects:** Party B recorded · state → JOINED · emit `CounterpartyJoined`. Does not change expiry.

### 3. `confirmTerms()` — each party

Each party submits the `termsHash` of the agreed terms. When both match: state → AGREED · emit `TermsConfirmed`.

### 4. `depositAssets()` — each party

Pulls the exact agreed amount from the caller into the deal's Safe via `SafeERC20` (requires a token approval first). First deposit → FUNDED_PARTIAL; second → LOCKED. Emit `AssetDeposited`, then `VaultLocked`.

### 5. `releaseFunds()` — anyone, once both parties have confirmed delivery

Called automatically by the second `confirmDelivery()`, and callable by anyone afterwards.
- `nonReentrant`
- Price band check via the price source if `priceBandBps > 0`. A stale feed reverts with no state change (try again later). A breach opens RENEGOTIATING (Pro / Enterprise) or cancels with refunds (Free) — D-06
- **State → COMPLETED before any transfer**
- Fee = deal value × locked rate, capped at the locked cap, **rounded down**, wei precision
- Atomic swap: each party receives the other's asset net of fee; fee to the fee wallet, with operator split where applicable. All transfers succeed or none do
- Emit `FundsReleased`

### 6. `claimRefund()` — anyone

Available once expiry is reached, or once a renegotiation window lapses unresolved.
- `nonReentrant`
- **State → EXPIRED or CANCELLED before any transfer**
- Each party's deposit returned **independently**. If one transfer fails (for example, a token issuer has frozen that address), that refund stays recorded as unpaid and can be retried later by calling `claimRefund()` again. The other party's refund is unaffected
- Zero fee, always
- Emit `RefundPaid` per party (or `RefundPending` if a transfer failed)

### 7. `cancelDeal()` — Party A alone, or both parties together

- **Party A alone:** any state before LOCKED
- **Both parties (mutual cancel):** any state before COMPLETED. Each party calls `cancelDeal()`; the second call executes
- `nonReentrant` · **state → CANCELLED before any transfer** · independent refunds as in `claimRefund()` · zero fee · emit `DealCancelled` with cancellation reference

---

## Supporting functions (4)

### 8. `confirmCounterparty()` — Party A

After Party B joins, Party A confirms "this is my counterparty" (D-02).
- **Yes:** the factory deploys the deal's Safe (owners A and B, threshold 2, module enabled, guard set) · state → NEGOTIATING · emit `CounterpartyConfirmed`
- **No:** Party B is removed, a new `inviteSigner` must be supplied · state → PENDING

No deposit is possible before this confirmation.

### 9. `confirmDelivery()` — each party, state LOCKED

Records the party's confirmation and timestamp on-chain (D-03). The first confirmation starts the auto-release window (if enabled). The second calls `releaseFunds()`. Emit `DeliveryConfirmed`.

### 10. `triggerAutoRelease()` — anyone

Requires: auto-release enabled · exactly one party has confirmed · the window has passed since that confirmation. Executes the same path, checks and fee rules as `releaseFunds()`. Callers gain no power by calling — they can only trigger the outcome the rules already decided. Emit `AutoReleaseTriggered`.

### 11. `amendTerms()` — both parties

Each party calls with identical new values; the second call executes. Used for:
- **Amendments** — new terms and/or a new expiry (D-07)
- **Renegotiation** — accepting new terms after a price breach (D-06). The deal returns to LOCKED and delivery confirmations reset

If an amendment before LOCKED reduces a party's required deposit after they have deposited, the difference is refunded; if it increases it, the party tops up through `depositAssets()`. After LOCKED, amendments cannot increase amounts — the parties cancel and create a new deal instead. Emit `TermsAmended`.

---

## Fee tier resolution (D-04)

| Situation | Fee applied |
|-----------|------------|
| No proof supplied | Free: 1%, max $10 |
| Valid Pro voucher (signed by the TrustBridge voucher key, names the caller's wallet, not expired) | Pro: 0.5%, max $25 |
| Registered Enterprise operator | Operator's registered rate (0.25% standard, negotiable) and 70 / 30 split |

The fee tier is never a free-form caller parameter. The fee rate and cap are locked at creation. The voucher key and the registry can only affect the fee level of new deals — never any deposit.

---

## Settings and admin powers

| Setting | Change process |
|---------|---------------|
| Token whitelist, price sources (per chain) | Admin multisig, 7-day public notice, new deals only |
| Fee rates and caps, fee wallet, operator registry, voucher key | Admin multisig, 7-day public notice, new deals only |
| Plan timer maximums, minimum deal value, guarded-launch cap | Admin multisig, 7-day public notice, new deals only |
| Pause new deals | Admin multisig, immediate. Stops `createDeal()` only — every existing deal continues normally |

**No admin function can move, freeze, redirect or delay user funds, change an existing deal's rules, or override any outcome.** There is no emergency "break-glass" mode in v1 (D-08). Larger changes are made by deploying a new factory version: new deals use the new version, and existing deals finish under the rules they started with.

---

## Multi-chain configuration (D-09)

No chain-specific addresses are written into the contract code. Each chain has its own configuration, set at deployment:
- Safe contract addresses
- Price source per token (Chainlink adapter on Base; any other chain decided before its deployment)
- Token whitelist
- Fee wallet and operator registry

Adding an EVM chain is a configuration and deployment task, followed by a chain-specific audit review. Non-EVM chains (for example Solana) and cross-chain deals are out of scope.

---

## Price checks

- On-chain prices come only from the configured price source — on Base, Chainlink Data Feeds. CoinGecko is used for UI display only
- Every read checks staleness and round completeness. Stale data blocks the price-dependent action rather than using an old price
- Pair rates are derived from USD feeds (for example ETH/USD ÷ USDC/USD)
- Refunds, cancellations and expiry never depend on the price source

---

## Fee configuration

| Plan | Rate | Cap |
|------|------|-----|
| Free | 1% of deal value | $10 maximum |
| Pro | 0.5% of deal value | $25 maximum |
| Enterprise | 0.25% of deal value | Negotiable |
| White-label split | 70% operator / 30% TrustBridge | Automatic on-chain |

Fee charged on completion only — never on creation, expiry or cancellation. Rounded down in the user's favour, wei precision. A misconfigured fee wallet reverts.

---

## Token whitelist and minimum value

**v1 tokens on Base:** USDC, USDT, ETH (as WETH), WBTC, DAI, WETH.  
**Minimum deal value:** $10 USD equivalent. **Guarded launch:** a maximum deal value applies for the first months of mainnet, raised over time.

---

## Events

| Event | Emitted by |
|-------|-----------|
| `DealCreated` | `createDeal()` |
| `CounterpartyJoined` | `acceptDeal()` |
| `CounterpartyConfirmed` / `CounterpartyRejected` | `confirmCounterparty()` |
| `TermsConfirmed` | `confirmTerms()` |
| `AssetDeposited`, `VaultLocked` | `depositAssets()` |
| `DeliveryConfirmed` | `confirmDelivery()` |
| `FundsReleased` | `releaseFunds()` |
| `AutoReleaseTriggered` | `triggerAutoRelease()` |
| `PriceBandBreach`, `RenegotiationOpened` | release checks |
| `TermsAmended` | `amendTerms()` |
| `RefundPaid` / `RefundPending` | `claimRefund()`, `cancelDeal()` |
| `DealExpired`, `DealCancelled` | `claimRefund()`, `cancelDeal()` |
| `SettingChangeScheduled`, `SettingChanged`, `NewDealsPaused`, `NewDealsResumed` | admin |

---

## Security requirements

See [SECURITY.md](SECURITY.md) for the full threat model.

**Mandatory on every fund-moving path:** `nonReentrant` · state updated before every transfer · `SafeERC20`.

**Invariants proven by tests:**
- Funds in a deal Safe can only ever leave to Party A, Party B or the recorded fee recipients
- Total paid out never exceeds total deposited
- No owner-initiated Safe transaction can succeed while the guard is set
- No admin or settings change can affect an existing deal
- One party's failed refund never blocks the other party's refund
- Every payout goes only to the party address recorded in the deal
- All party actions work when a party is a smart-contract wallet (Safe)

**Test coverage:** 95%+ line coverage; 100% on fund-moving paths; fuzz and invariant tests on deposit, release, auto-release, refund, cancellation and amendment.

---

## Dependencies

| Library | Usage |
|---------|-------|
| OpenZeppelin 5.x | ReentrancyGuard, SafeERC20, EIP-712 signatures, access control (configuration only) |
| Safe (official contracts, unmodified) | Deal Safe proxies, module and guard interfaces |
| Chainlink Data Feeds | Price source adapter on Base |
| Foundry | Development, testing, deployment |

---

## Details to confirm during build

- Which leg of the swap the completion fee is deducted from (to match the prototype's fee display)
- Pro voucher validity period and renewal flow
- Guarded-launch cap amount and schedule
- Chainlink feed availability on Base Sepolia for each whitelisted token (mock price source for testnet-only gaps)

Enterprise agent seats, roles, per-agent deal attribution and consolidated reporting are application-layer features (operator dashboard) and need no contract changes.

---

## Deployment checklist

- [ ] Factory, module, guard and Base price adapter implemented
- [ ] All 11 functions and invariants tested; 95%+ coverage
- [ ] Static analysis (Slither) — no unresolved critical or high findings
- [ ] Independent senior Solidity / Safe review
- [ ] Independent security audit — report published on this repository, naming the audited version
- [ ] Base Sepolia deployment verified; two-party demo deals completed
- [ ] 100 testnet deals across every outcome path
- [ ] Admin multisig, timelock, fee wallet and whitelist verified on-chain
- [ ] Guarded-launch cap set
- [ ] VARA legal opinion obtained before mainnet

---

*Specification v1.2 · October 2026 · TrustBridge Protocol · trustbridgeprotocol.xyz*  
*Responsible disclosure: craig@trustbridgeprotocol.xyz*
