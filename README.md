**Trustless, non-custodial atomic swap between ETH and USDC on EVM networks.**

Built with symmetric MAD collateral, commit-reveal MEV protection, reputation scoring, and bidirectional collusion detection.

**Version:** v9.0 FINAL  
**Solidity:** 0.8.28  
**License:** MIT

---

## Overview

SecureEthUsdcSwap enables two parties to exchange ETH and USDC without a trusted intermediary. The protocol uses Hash Time-Locked Contract (HTLC) mechanics combined with a collateral system that punishes griefing and rewards honest participation.

The contract ensures both parties are equally incentivized to complete the swap through:

- **Symmetric Collateral**: Both parties lock collateral equal to 5% of their asset value
- **Premium Bonds**: Initiators post a 2% (minimum 0.0005 ETH) anti-griefing premium
- **On-Chain Reputation**: Tracks completed swaps, griefing incidents, and cancellations
- **Collusion Detection**: Bidirectional flags prevent repeated griefing between the same parties
- **Pull-Payment Withdrawals**: USDC blacklisting cannot block ETH recovery

---

## How It Works

### Lifecycle

commitInitiate()                        →  COMMIT
initiateFromCommit()                    →  INITIATED
fund()                                  →  FUNDED
complete() / claimAsCounterparty()      →  COMPLETED
or refund()                             →  REFUNDED
or cancel()                             →  CANCELLED

### Timing Diagram

All windows are relative to `duration` chosen at initiation. Example with `duration = 2h`:


T+0                T+1h30m             T+2h
|                  |                   |
[──── fund window ─────────────────────]
[──── reveal window────────────────────]
+15s ← refund opens (FUNDED)
+15s ← refund opens (INITIATED)

Where:
- `revealDeadline = deadline - 30min` (always 30 minutes before deadline)
- `deadline = T + duration` (chosen by initiator, between 1h and 7d)

**Key Properties:**
- **No dead zone**: `complete()` closes and `refund()` opens at the same boundary (`revealDeadline + 15s`)
- **No grey zone**: `cancel()` closes and `refund()` opens at the same boundary (`deadline + 15s`)

---

## Key Features

| Feature | Description |
|---|---|
| **Commit-Reveal** | MEV protection: parameters hidden until `initiateFromCommit()` |
| **MAD Collateral** | Both parties post 5% collateral; griefer's collateral is burned |
| **Premium** | Initiator posts 2% premium; returned on success, burned on cancel, paid to counterparty if initiator griefs |
| **Pull-Payment** | ETH and USDC withdrawn separately — USDC blacklist cannot block ETH |
| **Reputation** | On-chain score tracks completed, griefed, victimized, cancelled swaps |
| **Collusion Detection** | Canonical pair counter; A→B and B→A grief share one slot |
| **Rate Limiting** | Per-address cooldowns and max 5 concurrent active swaps |

---

## Collateral Model

### Initiator Deposits (ETH side)


ethAmount + 5% collateral + 2% premium (min 0.0005 ETH)

### Counterparty Deposits (USDC side)


usdcAmount + 5% collateral

### On Success
- Initiator receives: USDC principal + ETH collateral + ETH premium back
- Counterparty receives: ETH principal + USDC collateral back

### On Grief (initiator fails to reveal secret in time)
- Initiator's ETH collateral → burned to 0xdEaD
- Counterparty's USDC collateral → logically burned (stays in contract)
- Premium → counterparty as compensation
- Principals returned to each party

### On Cancel (initiator cancels before deadline)
- Initiator receives: ETH principal + ETH collateral back
- Premium → burned to 0xdEaD (anti-griefing penalty for wasting counterparty's time)
- Reputation: initiator's `cancelled` counter incremented

> Note: `cancel()` is only available **before** the deadline. After the deadline, use `refund()` instead — it returns the premium with no reputation penalty.

---

## Installation

```bash
npm install @openzeppelin/contracts
```

**Dependencies:**
- `@openzeppelin/contracts` — `IERC20`, `SafeERC20`, `ReentrancyGuard`

---

## Deployment

```solidity
constructor(address _usdc)
```

| Parameter | Description |
|---|---|
| `_usdc` | Address of the USDC ERC-20 token contract |

**Example (Hardhat):**
```javascript
const swap = await ethers.deployContract("SecureEthUsdcSwap", [
  "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48" // USDC mainnet
]);
```

> ⚠️ **CRITICAL**: Verify that `0xdEaD` has no deployed code on your target network before deploying. On Ethereum mainnet it is a pure EOA.

---

## Usage

### Step 0 — Commit

```javascript
const salt       = ethers.randomBytes(32);
const secret     = ethers.randomBytes(32);
const nonce      = ethers.randomBytes(32);
const secretHash = ethers.keccak256(ethers.AbiCoder.defaultAbiCoder().encode(
  ["bytes32","bytes32"], [secret, nonce]
));

const commitment = ethers.keccak256(ethers.AbiCoder.defaultAbiCoder().encode(
  ["bytes32","bytes32","address","uint256","uint256","uint256","address","bytes32"],
  [secretHash, nonce, counterparty, ethAmount, usdcAmount, duration, initiator, salt]
));

await swap.commitInitiate(commitment);
```

### Step 1 — Initiate

```javascript
const deposit = await swap.requiredEthDeposit(ethAmount);

await swap.initiateFromCommit(
  commitment, salt, secretHash, nonce,
  counterparty, ethAmount, usdcAmount, duration,
  { value: deposit }
);
```

### Step 2 — Fund (counterparty)

```javascript
const usdcDeposit = await swap.requiredUsdcDeposit(usdcAmount);
await usdc.approve(swap.address, usdcDeposit);
await swap.fund(swapId);
```

### Step 3 — Complete

```javascript
// Initiator reveals secret
await swap.complete(swapId, secret);

// OR counterparty claims if they learned the secret cross-chain
await swap.claimAsCounterparty(swapId, secret);
```

### Withdraw

```javascript
// Always use separate functions — never assume USDC won't revert
await swap.withdrawEth();
await swap.withdrawUsdc();
```

---

## View Functions

| Function | Returns | Purpose |
|---|---|---|
| `getSwap(swapId)` | Swap struct | Full swap state |
| `getReputation(user)` | (completed, griefed, victimized, cancelled, volume, score) | User reputation details |
| `getPairGriefCount(a, b)` | uint32 | Canonical symmetric grief count |
| `getPendingWithdrawals(user)` | (pendingEth, pendingUsdc) | Claimable balances |
| `isCollusionFlagActive(a, b)` | (active, expiryTime) | Flag status and expiry |
| `revealWindowOpen(swapId)` | bool | Whether secret can still be revealed |
| `timeLeft(swapId)` | uint256 | Seconds until deadline |
| `canInitiate(user)` | (ok, reason) | Rate limit check with reason string |
| `buildCommitment(...)` | bytes32 | Off-chain commitment hash helper |
| `buildSecretHash(secret, nonce)` | bytes32 | Off-chain secret hash helper |

---

## Constants

| Constant | Value | Purpose |
|---|---|---|
| `MIN_ETH` | 0.001 ETH | Minimum swap size |
| `MIN_USDC` | 1 USDC (1e6) | Minimum USDC amount |
| `MIN_DURATION` | 1 hour | Shortest swap window |
| `MAX_DURATION` | 7 days | Longest swap window |
| `SECRET_REVEAL_BUFFER` | 30 minutes | Reveal window size |
| `COLLATERAL_BPS` | 500 (5%) | Collateral as % of amount |
| `PREMIUM_BPS` | 200 (2%) | Premium as % of ETH |
| `MIN_PREMIUM` | 0.0005 ETH | Minimum premium |
| `MAX_ACTIVE_PER_USER` | 5 | Max concurrent swaps |
| `INITIATE_COOLDOWN` | 2 minutes | Min time between initiations |
| `FUND_COOLDOWN` | 1 minute | Min time between funding operations |
| `COMMIT_EXPIRY` | 10 minutes | Commitment validity |
| `COLLUSION_THRESHOLD` | 2 | Grief events to flag collusion |
| `COLLUSION_FLAG_TIMEOUT` | 30 days | Collusion flag duration |
| `TIMESTAMP_DRIFT_BUFFER` | 15 seconds | Validator timestamp tolerance |

---

## Error Reference

| Error | Trigger | Mitigation |
|---|---|---|
| `EmptyCommitment` | Zero-value commitment hash | Use non-zero hash |
| `AlreadyCommitted` | Commitment hash already registered | Use different salt |
| `CommitmentUsed` | Commitment already consumed | Generate new secret/nonce |
| `CommitmentExpired` | Commitment older than 10 minutes | Initiate within 10min of commit |
| `EthTooSmall` | ETH amount below MIN_ETH | Increase ETH amount to ≥0.001 |
| `UsdcTooSmall` | USDC amount below MIN_USDC | Increase USDC amount to ≥1 |
| `DurationTooShort` / `TooLong` | Duration outside [1h, 7d] | Set duration between 1h-7d |
| `SelfSwap` | Counterparty is caller | Specify different counterparty |
| `SelfFund` | Initiator tries to fund own swap | Use different address to fund |
| `CollusionFlagActive` | Pair flagged for repeated griefing | Wait 30 days for flag to expire |
| `SecretHashReused` | secretHash previously used | Generate new secret/nonce |
| `WrongEthDeposit` | msg.value does not match required | Send exact `requiredEthDeposit()` amount |
| `RevealWindowClosed` | Secret submitted after reveal deadline | Complete before `revealDeadline + 15s` |
| `WrongSecret` | Secret does not match secretHash | Use correct secret |
| `NotInitiator` | Caller is not the swap initiator | Use initiator's address |
| `NotCounterparty` | Caller is not the swap counterparty | Use counterparty's address |
| `NotExpired` | Refund called before effective deadline | Wait until deadline + 15s |
| `CannotRefund` | Swap not in INITIATED or FUNDED state | Refund only in INITIATED/FUNDED states |
| `TooManyActiveSwaps` | activeCount reached MAX_ACTIVE_PER_USER | Wait for existing swap to complete |
| `NothingToWithdraw` | No pending balance for caller | Check `getPendingWithdrawals()` first |
| `EthWithdrawFailed` | Low-level ETH transfer reverted | Check account is not blacklisted |
| `VolumeOverflow` | Volume counter would overflow uint128 | Extremely unlikely; split into smaller swaps |

---

## Intentional Design Decisions

### `usedSecretHashes` is Permanent
Not cleared on cancel or refund. Each `secretHash` is globally single-use to prevent replay attacks. **Always generate a fresh secret for each swap attempt.**

### USDC Burn is Logical
`bCollateral` is burned by incrementing `burnedUsdcTotal`; tokens remain in the contract permanently. Physical transfer to `0xdEaD` is skipped because Circle may blacklist that address, which would cause `refund()` to permanently revert.

### ETH Burn is Physical
`aCollateral` and `aPremium` (on cancel) are sent to `0xdEaD` via low-level `call`. This removes them from the contract balance and eliminates the storage leak of the old `pendingWithdrawals[0xdEaD]` approach.

### `withdrawBoth()` is Omitted
A combined withdrawal reintroduces the USDC-blacklist-blocks-ETH atomicity bug. Use `withdrawEth()` and `withdrawUsdc()` independently.

---

## Gas Estimates

| Operation | Gas |
|---|---|
| `commitInitiate()` | ~24,000 |
| `initiateFromCommit()` | ~85,000 |
| `fund()` | ~95,000 |
| `complete()` | ~65,000 |
| `refund()` | ~70,000 |
| `cancel()` | ~45,000 |
| `withdrawEth()` | ~25,000 |
| `withdrawUsdc()` | ~60,000 |

---

## License

MIT — see [LICENSE](./LICENSE)
apV3 is an experiment in economically secure peer‑to‑peer exchange mechanisms built entirely on‑chain.
