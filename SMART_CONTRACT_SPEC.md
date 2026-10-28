# TrustBridge Protocol — Smart Contract Specification

**Version:** 1.1 — October 2026  
**Status:** Specification — contracts not yet written, audited or deployed  
**Target chain:** Base (primary) · Polygon (secondary)  
**Target architecture:** Safe (Gnosis Safe) 2-of-2 multisig smart account

---

## Overview

`TrustBridgeDeal.sol` is the core smart contract for TrustBridge Protocol v1. It is designed to provide non-custodial deal-room escrow on a Safe-powered 2-of-2 multisig architecture. No TrustBridge-controlled key, and no single party, can move user funds at any point.

The exact custody model (how deposits are held so that refunds and auto-release work without an admin key) is one of the architecture decisions listed below, confirmed with the development team before build begins.

### What changed in v1.1

- `cancelDeal()` now returns every deposit held, whichever party made it
- Deal expiry is fixed at creation; `acceptDeal()` no longer starts or resets the timer
- The fee tier is no longer a caller-supplied parameter
- `createDeal()` accepts a signers list and threshold from day one
- The single-party auto-release path is specified explicitly
- Admin powers are explicitly limited to configuration
- Open architecture decisions are listed in their own section

---

## Important: v1 dispute resolution architecture

TrustBridge Protocol v1 does **not** include `openDispute()` or `resolveDispute()` functions. This is a deliberate architectural decision.

**Rationale:** A conventional dispute mechanism where a person reviews evidence and decides who receives funds would make TrustBridge a custodian for the duration of any dispute. That would compromise the non-custodial position under UAE VARA and introduce discretionary control over user funds.

**v1 approach:** All deal outcomes are governed by contract logic:
- Both parties confirm → atomic swap executes
- One party confirms and the other does not respond within the window → single-party auto-release completes the deal
- Timer expires → `claimRefund()` returns all deposits

**"Raise a Concern"** exists in the UI as a timestamped record only. It makes no contract call and does not affect deal state.

**v2 roadmap:** Kleros decentralised arbitration — a randomly selected jury reviews evidence and a smart contract executes the ruling. No person at TrustBridge makes any decision about funds.

---

## Deal state machine (v1)

```
createDeal()     acceptDeal()     confirmTerms()   depositAssets()       depositAssets()
PENDING    →    NEGOTIATING   →    AGREED     →    FUNDED_PARTIAL   →    LOCKED

From LOCKED:
  releaseFunds()      → COMPLETED  (both parties confirm)
  auto-release path   → COMPLETED  (one party confirms, response window passes)
  claimRefund()       → EXPIRED    (deal expiry reached — permissionless)

From PENDING, NEGOTIATING, AGREED or FUNDED_PARTIAL:
  cancelDeal()        → CANCELLED  (Party A only)
  claimRefund()       → EXPIRED    (deal expiry reached — permissionless)
```

Deal expiry is set once at `createDeal()` and is a single continuous countdown. No function resets or extends it.

---

## Core functions — v1 (7 functions + auto-release path)

### `createDeal()`

**Who calls it:** Party A

| Parameter | Type | Validation |
|-----------|------|-----------|
| `assetOffered` | `address` | Must be on the approved token whitelist. |
| `amountOffered` | `uint256` | Deal value ≥ $10 USD equivalent at current Chainlink price. |
| `assetRequested` | `address` | Must be on the approved token whitelist. |
| `amountRequested` | `uint256` | Deal value ≥ $10 USD equivalent. |
| `expiry` | `uint256` | > `block.timestamp` + 15 minutes. ≤ tier maximum (Free 7d, Pro 30d, Enterprise 365d). |
| `priceBandBps` | `uint16` | Basis points. 0 = price band OFF (Pro / Enterprise only). |
| `signers` | `address[]` | v1 passes exactly 2. Present from day one so M-of-N needs no rewrite. |
| `threshold` | `uint8` | v1 passes 2. |

Party B binding is handled per architecture decision D-02. The fee tier is **not** a parameter: it comes from a source the caller cannot set (decision D-04).

**Key logic:**
- `require(approvedTokens[assetOffered] && approvedTokens[assetRequested])`
- `require(dealValueUSD >= MIN_DEAL_VALUE_USD)`
- State → PENDING
- Emit `DealCreated(dealId, partyA, assetOffered, amountOffered, assetRequested, amountRequested, expiry, priceBandBps, timestamp)`

---

### `acceptDeal()`

**Who calls it:** Party B

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be PENDING. `block.timestamp < expiry`. |

**Key logic:**
- Party B is bound to the deal per decision D-02 — no later substitution possible
- State → NEGOTIATING
- Does not start, reset or extend the deal expiry
- Emit `DealAccepted(dealId, partyB, timestamp)`

---

### `confirmTerms()`

**Who calls it:** Either party

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be NEGOTIATING. |
| `termsHash` | `bytes32` | Hash of the agreed terms. |

**Key logic:**
- Store hash against the calling party
- When both hashes are stored and match: State → AGREED
- Emit `TermsConfirmed(dealId, party, termsHash, timestamp)`

---

### `depositAssets()`

**Who calls it:** Either party

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be AGREED or FUNDED_PARTIAL. Caller has not already deposited. |

**Key logic:**
- `SafeERC20.safeTransferFrom` — exact agreed amount, no more, no less
- Compliance gate above AED 55,000 (~$15,000) handled per decision D-05
- First deposit: State → FUNDED_PARTIAL
- Second deposit: State → LOCKED
- Emit `AssetDeposited(dealId, party, asset, amount, timestamp)`; emit `VaultLocked(dealId, timestamp)` when both are funded

---

### `releaseFunds()`

**Who calls it:** Requires both parties' confirmation

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be LOCKED. |

**Key logic:**
- `nonReentrant` — mandatory
- Chainlink price band check if `priceBandBps > 0`, with stale-feed handling; breach handling per decision D-06
- **UPDATE STATE FIRST:** `deal.state = COMPLETED`
- `feeAmount = (dealValue * feeBps) / 10000`, capped per plan — round DOWN, wei precision
- Misconfigured fee wallet → revert, never silent failure
- Transfer Party A's asset to Party B and Party B's asset to Party A, net of fee
- Transfer fee (with operator split where applicable)
- Emit `FundsReleased(dealId, partyA, partyB, feeAmount, timestamp)`

---

### Auto-release path

**Triggered when:** one party has confirmed delivery and the other has not responded within the response window (Free / Pro: 15 minutes; Enterprise: operator-configurable, default 2 hours).

**Key logic:**
- Mechanism per decision D-03 — no admin key involved
- Same checks, fee and transfer rules as `releaseFunds()`
- Emit `AutoReleaseTriggered(dealId, confirmingParty, timestamp)`

---

### `claimRefund()`

**Who calls it:** Anyone — permissionless once expiry is reached

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. `block.timestamp >= deal.expiry`. State not COMPLETED, CANCELLED or EXPIRED. |

**Key logic:**
- `nonReentrant`
- **UPDATE STATE FIRST:** `deal.state = EXPIRED`
- Return every deposit held to the party that made it
- Zero fee — always
- Emit `DealExpired(dealId, timestamp)`

---

### `cancelDeal()`

**Who calls it:** Party A only, before LOCKED

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be PENDING, NEGOTIATING, AGREED or FUNDED_PARTIAL. |

**Key logic:**
- `require(msg.sender == deal.partyA)`
- `nonReentrant`
- **UPDATE STATE FIRST:** `deal.state = CANCELLED`
- Return every deposit held to the party that made it — Party A's **or** Party B's
- Zero fee — always
- Emit `DealCancelled(dealId, cancelRef, timestamp)`

---

## Architecture decisions — confirmed before build

| # | Decision | Summary |
|---|----------|---------|
| D-01 | Custody model | A 2-of-2 Safe alone cannot execute a permissionless refund or a one-sided auto-release. Options: a per-deal Safe with a narrowly scoped TrustBridge module, or an escrow contract holding deposits directly. |
| D-02 | Party B binding | Baseline: single-use invite, first acceptor bound, Party A confirms the bound wallet before funding. Alternative: exact address entered at creation. |
| D-03 | Single-party auto-release | How a one-sided confirmation completes after the window with no admin key. Depends on D-01. |
| D-04 | Fee tier source | Never caller-declared. Operator registry, signed attestation, or per-operator deployment. |
| D-05 | Compliance gate | Whether the AED 55,000 enhanced due diligence threshold is enforced on-chain or at the operator / application layer. |
| D-06 | Price band breach | On-chain handling of a breach at release and of the renegotiation window; unresolved windows cancel and return all deposits. |

---

## Admin and owner powers

Admin roles are limited to configuration: the token whitelist, the fee wallet, and the operator registry. **No admin function can move, freeze, redirect or delay user funds**, and no admin action can override a deal outcome.

---

## Fee configuration

| Plan | Rate | Cap |
|------|------|-----|
| Free | 1% of deal value | $10 maximum |
| Pro | 0.5% of deal value | $25 maximum |
| Enterprise | 0.25% of deal value | Negotiable |
| White-label split | 70% operator / 30% TrustBridge | Automatic on-chain split |

Fee charged on completion only — never on expiry or cancellation. Rounded DOWN in favour of the user, wei precision. Misconfigured fee wallet = revert.

---

## Token whitelist

Enforced at `createDeal()`. **v1 approved tokens:** USDC, USDT, ETH, WBTC, DAI, WETH.

## Minimum deal value

$10 USD equivalent, enforced at `createDeal()` against the Chainlink price. Prevents gas griefing.

---

## Events (10)

| Event | Emitted by | Key data |
|-------|-----------|---------|
| `DealCreated` | `createDeal()` | dealId, partyA, assets, amounts, expiry, priceBand |
| `DealAccepted` | `acceptDeal()` | dealId, partyB, timestamp |
| `TermsConfirmed` | `confirmTerms()` | dealId, party, termsHash, timestamp |
| `AssetDeposited` | `depositAssets()` | dealId, party, asset, amount, timestamp |
| `VaultLocked` | `depositAssets()` | dealId, timestamp |
| `FundsReleased` | `releaseFunds()` | dealId, partyA, partyB, feeAmount, timestamp |
| `AutoReleaseTriggered` | auto-release path | dealId, confirmingParty, timestamp |
| `DealExpired` | `claimRefund()` | dealId, timestamp |
| `DealCancelled` | `cancelDeal()` | dealId, cancelRef, timestamp |
| `PriceBandBreach` | release check | dealId, currentRate, agreedRate, bandBps |

---

## Security requirements

See [SECURITY.md](SECURITY.md) for the full threat model.

**Mandatory on every fund-moving function:**
- `nonReentrant` (OpenZeppelin ReentrancyGuard)
- State updated BEFORE every transfer (checks-effects-interactions)
- `SafeERC20` for all token transfers

**Test coverage:**
- 95%+ line coverage; 100% on fund-moving functions
- Every revert condition, reentrancy, timing-attack and gas-griefing scenario tested
- Fuzz and invariant tests on deposit, release, refund and cancellation paths

---

## Dependencies

| Library | Usage |
|---------|-------|
| OpenZeppelin 5.x | ReentrancyGuard, SafeERC20, access control (configuration only) |
| Safe | 2-of-2 multisig smart account architecture |
| Chainlink Data Feeds | On-chain price band and minimum value checks |
| Foundry or Hardhat | Development, testing, deployment |

---

## Deployment checklist

- [ ] Architecture decisions D-01 to D-06 confirmed and documented
- [ ] All 7 v1 functions and the auto-release path implemented and tested
- [ ] 95%+ test coverage confirmed
- [ ] Static analysis (Slither) — no unresolved critical or high findings
- [ ] Independent security audit — report published on this repository
- [ ] Base Sepolia deployment verified
- [ ] 100 testnet deals completed across all outcome paths
- [ ] Fee wallet address verified on-chain
- [ ] Token whitelist set and verified
- [ ] VARA legal opinion obtained before mainnet

---

*Specification v1.1 · October 2026 · TrustBridge Protocol · trustbridgeprotocol.xyz*  
*Responsible disclosure: craig@trustbridgeprotocol.xyz*
