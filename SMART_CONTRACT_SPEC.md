# Smart Contract Specification

**TrustBridge Protocol — TrustBridgeDeal.sol**

> **Status:** Specification only. Contracts have not yet been written or deployed. This document is the developer brief for Phase 2 (Build).

---

## Technical stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Foundation** | Safe (Gnosis Safe) SDK | Do NOT write a multisig wallet from scratch. Safe is battle-tested, audited, and the architectural core of TrustBridge's trust model. |
| **Language** | Solidity 0.8.x | Built-in overflow protection. Industry standard. |
| **Framework** | Hardhat or Foundry | Developer's preference — both acceptable. |
| **Security libraries** | OpenZeppelin ReentrancyGuard, SafeERC20, Ownable | Non-negotiable. These cover the three most common smart contract vulnerability classes. |
| **Primary chain** | Base | Low gas (~$0.03/tx), EVM-compatible, Coinbase institutional backing. |
| **Secondary chain** | Polygon | Same contracts — network config change only. |
| **Price oracle** | Chainlink | For all on-chain price band verification. Never CoinGecko for contract logic. |

---

## Deal state machine

```
createDeal()     acceptDeal()    confirmTerms()   depositAssets()
    │                │                │                │
PENDING ────────► NEGOTIATING ──────► AGREED ─────────► FUNDED
                                                        │
                                               (both deposited)
                                                        │
                                                     LOCKED
                                                   ╱    │    ╲
                              releaseFunds()       │    │     │  openDispute()
                                    │              │    │     │
                                COMPLETED       EXPIRED  DISPUTED ──► RESOLVED
                              (atomic swap)   (claimRefund)
```

---

## Core function specification

### `createDeal()`

**Who calls it:** Party A

| Parameter | Type | Validation |
|-----------|------|-----------|
| `partyBAddress` | `address` | Must not be zero address. Must not equal msg.sender. |
| `assetOffered` | `address` | Must be on approved token whitelist. |
| `amountOffered` | `uint256` | Must be greater than zero. |
| `assetRequested` | `address` | Must be on approved token whitelist. |
| `amountRequested` | `uint256` | Must be greater than zero. |
| `expiryTimestamp` | `uint256` | Must be at least 30 minutes from now. Maximum 365 days. |
| `priceBandTolerance` | `uint16` | 0 = band off. 1–5000 = basis points (50 = 0.5%, 1000 = 10%). |
| `autoRelease` | `bool` | True = 15-minute auto-release after both confirm delivery. False = manual (Enterprise only). |

**Key logic:**
- Generate unique `dealId` (hash of block data + msg.sender + nonce)
- Write state: `PENDING`
- Store all parameters on-chain
- Emit `DealCreated(dealId, partyA, partyB, assetOffered, amountOffered, assetRequested, amountRequested, expiryTimestamp)`

---

### `acceptDeal()`

**Who calls it:** Party B

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. |

**Key logic:**
- `require(msg.sender == deal.partyB)` — exact address match only
- `require(deal.state == PENDING)`
- `require(block.timestamp < deal.expiryTimestamp)`
- Move state: `PENDING → NEGOTIATING`
- Record `negotiationStartedAt` timestamp
- Emit `DealAccepted(dealId, partyB, timestamp)`

---

### `confirmTerms()`

**Who calls it:** Both Party A and Party B independently

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. |
| `termsHash` | `bytes32` | SHA-256 hash of agreed terms document. Both parties must submit identical hash. |

**Key logic:**
- Record confirmation per party
- If both confirmed AND both hashes match: store `termsHash` on-chain, move state `NEGOTIATING → AGREED`
- If hashes do not match: revert — parties have not agreed on the same terms
- Emit `TermsConfirmed(dealId, partyA, partyB, termsHash)` when both confirmed
- Reset confirmations if either party calls `adjustDeal()` after confirming

---

### `depositAssets()`

**Who calls it:** Party A and Party B independently

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be `AGREED`. |

**Key logic:**
- `require(deal.state == AGREED)`
- `require(block.timestamp < deal.expiryTimestamp)`
- Accept exact agreed amount via `SafeERC20.safeTransferFrom(msg.sender, address(this), amount)`
- Record per-party deposit status
- If both parties have deposited: auto-transition state `AGREED → FUNDED → LOCKED`
- The LOCKED state activates the Safe 2-of-2 multisig requirement
- Emit `AssetDeposited(dealId, party, asset, amount)`
- Emit `VaultLocked(dealId)` when both funded

---

### `releaseFunds()`

**Who calls it:** Both Party A and Party B independently (after ConfirmDeliveryModal on frontend)

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be `LOCKED`. |

**Key logic — order is CRITICAL:**
1. `nonReentrant` modifier — must be first
2. `require(deal.state == LOCKED)`
3. `require(block.timestamp < deal.expiryTimestamp)`
4. Record release confirmation per party
5. If both parties have confirmed AND (autoRelease OR 15 minutes elapsed):
   - **UPDATE STATE FIRST:** `deal.state = COMPLETED` — checks-effects-interactions, non-negotiable
   - Check price band if `priceBandTolerance > 0` — Chainlink oracle, revert if breach
   - Calculate fee: `min(dealValue * feeRate, feeCap)` — plan-dependent
   - Transfer fee to `platformWallet` via SafeERC20
   - Transfer Party B's asset to Party A via SafeERC20
   - Transfer Party A's asset to Party B via SafeERC20
   - Emit `DealCompleted(dealId, partyA, partyB, fee, timestamp)`

**Critical:** State must be set to COMPLETED before any transfer. No exceptions.

---

### `claimRefund()`

**Who calls it:** Anyone (permissionless after expiry)

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. Must be past expiry. State must not be COMPLETED or RESOLVED. |

**Key logic:**
- `require(block.timestamp >= deal.expiryTimestamp)`
- `require(deal.state != COMPLETED && deal.state != RESOLVED)`
- **UPDATE STATE FIRST:** `deal.state = EXPIRED`
- Return Party A deposit to Party A (if deposited)
- Return Party B deposit to Party B (if deposited)
- Zero fee — always
- Emit `DealExpired(dealId, timestamp)`
- Emit `RefundIssued(dealId, party, asset, amount)` for each party

---

### `cancelDeal()`

**Who calls it:** Party A only

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. |

**Key logic:**
- `require(msg.sender == deal.partyA)`
- `require(deal.state == PENDING || deal.state == NEGOTIATING || deal.state == AGREED || deal.state == FUNDED_PARTIAL)`
- Cannot cancel once state is `LOCKED` — both parties have deposited
- **UPDATE STATE FIRST:** `deal.state = CANCELLED`
- Return Party A deposit if deposited (FUNDED_PARTIAL state)
- Zero fee
- Emit `DealCancelled(dealId, partyA, cxlReference, timestamp)`

---

### `openDispute()`

**Who calls it:** Either Party A or Party B

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be `LOCKED`. |
| `reason` | `string` | Free text, max 500 characters. Stored as event data, not on-chain storage. |

**Key logic:**
- `require(deal.state == LOCKED)`
- `require(msg.sender == deal.partyA || msg.sender == deal.partyB)`
- Require dispute deposit: `$20 flat if dealValue < $10,000 / 1% if dealValue >= $10,000`
- Accept dispute deposit via SafeERC20
- **UPDATE STATE:** `deal.state = DISPUTED`
- Freeze all funds — neither release nor refund possible until resolved
- Start 7-day resolution window
- Emit `DisputeOpened(dealId, filer, reason, disputeDeposit, timestamp)`

---

### `resolveDispute()`

**Who calls it:** Admin wallet (v1) / Kleros arbitration contract (v2)

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be `DISPUTED`. |
| `winner` | `address` | Must be either partyA or partyB address. |

**Key logic:**
- `onlyAdmin` modifier in v1
- `require(deal.state == DISPUTED)`
- **UPDATE STATE FIRST:** `deal.state = RESOLVED`
- If winner == partyA: transfer both assets to partyA, return dispute deposit to partyA
- If winner == partyB: transfer both assets to partyB, return dispute deposit to partyB
- Emit `DisputeResolved(dealId, winner, timestamp)`

---

## Fee configuration (immutable at deployment)

Fee configuration is set in the constructor and cannot be changed after deployment. This is a trust guarantee — users can verify the fee rate on-chain.

| Plan | Rate | Cap |
|------|------|-----|
| Free | 1% of deal value | $10 maximum |
| Pro | 0.5% of deal value | $25 maximum |
| Enterprise | 0.25% of deal value | Negotiable |
| White-label | 70% operator / 30% TrustBridge | Automatic on-chain split |
| Dispute deposit | $20 flat (< $10K) / 1% (≥ $10K) | — |

**Fee is charged at `releaseFunds()` only.** Zero fee on expiry, cancellation, or dispute loss.

---

## Events (full list)

```solidity
event DealCreated(bytes32 indexed dealId, address partyA, address partyB, address assetOffered, uint256 amountOffered, address assetRequested, uint256 amountRequested, uint256 expiryTimestamp);
event DealAccepted(bytes32 indexed dealId, address partyB, uint256 timestamp);
event TermsConfirmed(bytes32 indexed dealId, address partyA, address partyB, bytes32 termsHash);
event AssetDeposited(bytes32 indexed dealId, address party, address asset, uint256 amount);
event VaultLocked(bytes32 indexed dealId);
event DealCompleted(bytes32 indexed dealId, address partyA, address partyB, uint256 fee, uint256 timestamp);
event DealExpired(bytes32 indexed dealId, uint256 timestamp);
event RefundIssued(bytes32 indexed dealId, address party, address asset, uint256 amount);
event DealCancelled(bytes32 indexed dealId, address partyA, bytes32 cxlReference, uint256 timestamp);
event DisputeOpened(bytes32 indexed dealId, address filer, string reason, uint256 disputeDeposit, uint256 timestamp);
event DisputeResolved(bytes32 indexed dealId, address winner, uint256 timestamp);
```

---

## Test coverage requirements

Before the security audit, the test suite must achieve:

- **95%+ line coverage** across all functions
- **100% coverage** of all fund-moving functions (depositAssets, releaseFunds, claimRefund, resolveDispute)
- All state machine transitions tested (valid and invalid)
- All revert conditions tested explicitly
- Reentrancy attack vectors tested
- Oracle manipulation scenarios tested
- Edge cases: zero amounts, expired deals, duplicate calls, wrong caller

---

## What this specification does NOT cover

- TrustBridge Vault contracts (M-of-N) — Phase 4, not in scope for Phase 2
- White-label configuration system — handled at frontend layer
- Admin dashboard — off-chain
- KYC integration — Phase 3 requirement
- Kleros arbitration module — v2, not v1

---

*Last updated: May 2026 · TrustBridge Protocol · trustbridgeprotocol.xyz*
