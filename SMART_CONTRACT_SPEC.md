# TrustBridge Protocol — Smart Contract Specification

**Version:** 1.0 — May 2026  
**Status:** Specification — contracts not yet deployed  
**Target chain:** Base (primary) · Polygon (secondary)  
**Architecture:** Safe (Gnosis Safe) 2-of-2 multisig smart account

---

## Overview

`TrustBridgeDeal.sol` is the core smart contract for TrustBridge Protocol v1. It implements a non-custodial escrow mechanism using Safe's 2-of-2 multisig architecture. Every deal creates a Safe smart account. Neither TrustBridge nor any single party holds or controls user funds at any point.

---

## Important: v1 dispute resolution architecture

TrustBridge Protocol v1 does **not** include `openDispute()` or `resolveDispute()` functions. This is a deliberate architectural decision.

**Rationale:** A conventional dispute mechanism where a human reviews evidence and decides who receives funds would make TrustBridge a custodian for the duration of any dispute. This would compromise the non-custodial argument under UAE VARA and introduce discretionary control over user funds.

**v1 approach:** All deal outcomes are governed entirely by smart contract logic:
- Both parties confirm → atomic swap executes
- Timer expires → `claimRefund()` auto-returns both deposits
- One party confirms, other does not respond → single-party auto-release fires after configured window

**"Raise a Concern"** exists in the UI as a timestamped record only. It makes no contract call and does not affect vault state.

**v2 roadmap:** Kleros decentralised arbitration — a randomly selected jury reviews evidence and a smart contract executes the ruling. No human at TrustBridge makes any decision about funds.

---

## Deal state machine (v1)

```
createDeal()     acceptDeal()     confirmTerms()   depositAssets()
PENDING    →    NEGOTIATING    →    AGREED    →   FUNDED/LOCKED

From LOCKED:
  releaseFunds() → COMPLETED  (both parties confirm)
  claimRefund()  → EXPIRED    (timer expires — permissionless)

From any state before LOCKED:
  cancelDeal()   → CANCELLED  (Party A only)
```

---

## Core functions — v1 (7 functions)

### `createDeal()`

**Who calls it:** Party A

| Parameter | Type | Validation |
|-----------|------|-----------|
| `partyB` | `address` | Non-zero. Not equal to msg.sender. Not a contract address. |
| `assetOffered` | `address` | Must be on approved token whitelist. |
| `amountOffered` | `uint256` | Must result in deal value ≥ $10 USD equivalent at current Chainlink price. |
| `assetRequested` | `address` | Must be on approved token whitelist. |
| `amountRequested` | `uint256` | Must result in deal value ≥ $10 USD equivalent. |
| `expiry` | `uint256` | Must be > block.timestamp. Must be ≤ tier maximum (Free: 7d, Pro: 30d, Enterprise: 365d). |
| `priceBandBps` | `uint16` | Basis points. 0 = price band OFF (Pro/Enterprise only). |
| `planTier` | `uint8` | 0=Free, 1=Pro, 2=Enterprise. |

**Key logic:**
- `require(approvedTokens[assetOffered] && approvedTokens[assetRequested])`
- `require(dealValueUSD >= MIN_DEAL_VALUE_USD)` — enforced at creation
- State → PENDING
- Emit `DealCreated(dealId, partyA, partyB, assetOffered, amountOffered, assetRequested, amountRequested, expiry, priceBandBps, timestamp)`

---

### `acceptDeal()`

**Who calls it:** Party B only

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be PENDING. |

**Key logic:**
- `require(msg.sender == deal.partyB)` — exact address match, no substitution possible
- State → NEGOTIATING
- Start timer from this point
- Emit `DealAccepted(dealId, partyB, timestamp)`

---

### `confirmTerms()`

**Who calls it:** Either party

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be NEGOTIATING. |
| `termsHash` | `bytes32` | SHA-256 hash of agreed terms document. |

**Key logic:**
- Store hash against calling party
- When both hashes stored and match: State → AGREED
- Emit `TermsConfirmed(dealId, party, termsHash, timestamp)`

---

### `depositAssets()`

**Who calls it:** Either party

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be AGREED. |

**Key logic:**
- `SafeERC20.safeTransferFrom` — exact agreed amount, no more, no less
- KYC gate: if deal value > AED 55,000 (~$15,000 USD), require KYC verification flag set
- When Party A deposits: State → FUNDED_PARTIAL
- When Party B deposits (both funded): State → LOCKED — vault is sealed
- Emit `AssetDeposited(dealId, party, asset, amount, timestamp)`
- Emit `VaultLocked(dealId, timestamp)` when both funded

---

### `releaseFunds()`

**Who calls it:** Requires both parties' signatures (2-of-2 via Safe)

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be LOCKED. |

**Key logic:**
- `nonReentrant` — mandatory
- Chainlink price band check: if `priceBandBps > 0`, verify current rate is within band
- **UPDATE STATE FIRST:** `deal.state = COMPLETED`
- Calculate fee: `feeAmount = (dealValue * feeBps) / 10000` — round DOWN, precision to wei
- If fee wallet misconfigured → revert, never silent failure
- Transfer Party A's asset to Party B
- Transfer Party B's asset to Party A
- Transfer fee to platform wallet
- Emit `FundsReleased(dealId, partyA, partyB, feeAmount, timestamp)`

---

### `claimRefund()`

**Who calls it:** Anyone — permissionless after expiry

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. `block.timestamp >= deal.expiry`. State must not be COMPLETED. |

**Key logic:**
- `nonReentrant`
- `require(block.timestamp >= deal.expiry)`
- **UPDATE STATE FIRST:** `deal.state = EXPIRED`
- Return Party A's deposit to Party A (if deposited)
- Return Party B's deposit to Party B (if deposited)
- Zero fee always — no exceptions
- Emit `DealExpired(dealId, timestamp)`

---

### `cancelDeal()`

**Who calls it:** Party A only, before LOCKED

| Parameter | Type | Validation |
|-----------|------|-----------|
| `dealId` | `bytes32` | Must exist. State must be PENDING, NEGOTIATING, AGREED, or FUNDED_PARTIAL. |

**Key logic:**
- `require(msg.sender == deal.partyA)`
- `require(deal.state != LOCKED && deal.state != COMPLETED && deal.state != EXPIRED)`
- **UPDATE STATE FIRST:** `deal.state = CANCELLED`
- Return Party A's deposit if already deposited
- Zero fee always
- Emit `DealCancelled(dealId, cancelRef, timestamp)`

---

## Fee configuration

Fee is set in the constructor and is immutable after deployment.

| Plan | Rate | Cap |
|------|------|-----|
| Free | 1% of deal value | $10 maximum |
| Pro | 0.5% of deal value | $25 maximum |
| Enterprise | 0.25% of deal value | Negotiable at deployment |
| White-label split | 70% operator / 30% TrustBridge | Automatic on-chain split |

**Fee rounding:** Always DOWN in favour of user. Precision to wei. Misconfigured fee wallet = revert.

---

## Token whitelist

Enforced at `createDeal()`. Only approved tokens permitted.

**v1 approved tokens:** USDC, USDT, ETH, WBTC, DAI, WETH

---

## Minimum deal value

$10 USD equivalent enforced at `createDeal()`. Checked against Chainlink price at creation. Prevents gas griefing attacks.

---

## Events (10 total)

| Event | Emitted by | Key data |
|-------|-----------|---------|
| `DealCreated` | `createDeal()` | dealId, partyA, partyB, assets, amounts, expiry, priceBand |
| `DealAccepted` | `acceptDeal()` | dealId, partyB, timestamp |
| `TermsConfirmed` | `confirmTerms()` | dealId, party, termsHash, timestamp |
| `AssetDeposited` | `depositAssets()` | dealId, party, asset, amount, timestamp |
| `VaultLocked` | `depositAssets()` | dealId, timestamp |
| `FundsReleased` | `releaseFunds()` | dealId, partyA, partyB, feeAmount, timestamp |
| `DealExpired` | `claimRefund()` | dealId, timestamp |
| `DealCancelled` | `cancelDeal()` | dealId, cancelRef, timestamp |
| `PriceBandBreach` | internal | dealId, currentRate, agreedRate, bandBps |
| `AutoReleaseTriggered` | internal | dealId, confirmingParty, timestamp |

---

## Security requirements

See [SECURITY.md](SECURITY.md) for full threat model.

**Mandatory on every fund-moving function:**
- `nonReentrant` (OpenZeppelin ReentrancyGuard)
- State updated BEFORE every transfer (checks-effects-interactions)
- `SafeERC20` for all token transfers

**Test coverage:**
- 95%+ line coverage
- 100% on fund-moving functions
- All revert conditions, reentrancy, timing attack, and gas griefing scenarios tested

---

## Dependencies

| Library | Usage |
|---------|-------|
| OpenZeppelin 5.x | ReentrancyGuard, SafeERC20, Ownable |
| Safe SDK | 2-of-2 multisig smart account |
| Chainlink | On-chain price band verification |
| Hardhat or Foundry | Development, testing, deployment |

---

## Deployment checklist

- [ ] All 7 v1 functions implemented and tested
- [ ] 95%+ test coverage confirmed
- [ ] Static analysis (Slither) — no critical or high findings
- [ ] Independent security audit — clean report published on this repository
- [ ] Base Sepolia testnet deployment verified
- [ ] 100 testnet deals completed
- [ ] Fee wallet address verified on-chain
- [ ] Token whitelist set and verified
- [ ] VARA legal opinion obtained before mainnet

---

*Specification v1 · May 2026 · TrustBridge Protocol · trustbridgeprotocol.xyz*  
*Responsible disclosure: craig@trustbridgeprotocol.xyz*
