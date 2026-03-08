SecureEthUsdcSwapV3
Trustless ETH ↔ USDC atomic swaps on Base
SecureEthUsdcSwapV3 is a peer‑to‑peer smart contract that enables users to swap ETH and USDC directly without intermediaries, custodians, or centralized exchanges.
The protocol is designed with a strict threat model and multiple defensive mechanisms to protect users against frontrunning, griefing, MEV attacks, and collusion.
This contract implements a secure commit‑reveal swap mechanism with symmetric collateral and on‑chain reputation tracking.

Philosophy
Traditional exchanges rely on trusted intermediaries.
SecureEthUsdcSwapV3 follows a different philosophy:
• Trustless execution — the contract enforces the swap
• Symmetric incentives — both parties must behave honestly
• Grief resistance — malicious behavior becomes economically irrational
• MEV resistance — swaps cannot be copied from the mempool
• Self‑custody — users keep control of their assets
The protocol is intentionally minimal:
• no order books
• no price feeds
• no admin control
Only cryptographic commitments and economic incentives enforce honest behavior.

Key Features
Atomic ETH ↔ USDC Swaps
Two users exchange ETH and USDC without trusting each other.
The swap is economically atomic and protocol‑enforced:
• either the exchange completes successfully
• or both parties recover their principal according to predefined rules
Although the swap lifecycle spans multiple transactions, the contract guarantees that dishonest behavior cannot result in unilateral asset capture.

Commit‑Reveal Protection
To prevent mempool copy attacks and MEV frontrunning, swaps use a two‑phase process:

1. Commit swap parameters (hidden)
2. Reveal parameters in a later transaction

Because the commitment includes the initiator’s address, only the original sender can reveal the parameters and create the swap.

Symmetric Collateral (MAD Model)
Both parties lock collateral:
• Initiator locks ETH collateral
• Counterparty locks USDC collateral
If one party violates the protocol, both collaterals are burned.
This mutually assured destruction (MAD) model discourages collusion and malicious griefing by making dishonest behavior economically irrational.

Griefing Protection
The protocol combines multiple mechanisms:
• premium payments
• collateral burning
• reputation tracking
• rate limits
Together, these mechanisms prevent denial‑of‑service and griefing attacks from being profitable.

Reputation System
Each address accumulates on‑chain history:
• completed swaps
• refunded swaps
• griefed swaps
• cancelled swaps
A score is derived from these metrics to help users evaluate counterparties.
Reputation is informational and observational.
It does not automatically block participation in swaps, except in cases where the collusion‑flag heuristic is triggered.
The system provides signaling and transparency rather than acting as a strict enforcement layer.

Pull‑Payment Architecture
ETH transfers are not pushed automatically.
Instead, ETH owed to users is queued in pendingWithdrawals and must be withdrawn manually.
This pull‑payment design prevents denial‑of‑service attacks caused by malicious fallback or receive functions.

Swap Lifecycle
The protocol follows a strict finite state machine:
EMPTY → INITIATED → FUNDED → COMPLETED
Alternative paths:
INITIATED → CANCELLED
INITIATED → REFUNDED
FUNDED → REFUNDED
All invalid state transitions are rejected by the contract.

Swap Steps
1. Commit
The initiator commits swap parameters using a hash.
commitInitiate(commitment)

2. Initiate
Swap parameters are revealed and ETH is deposited.
initiateFromCommit(...)

3. Fund
The counterparty deposits USDC.
fund(swapId)

4. Complete
The initiator reveals the secret and assets are exchanged.
complete(swapId, secret)

Alternative Outcomes
Refund
If the deadline expires without completion:
refund(swapId)

Cancel
If the initiator cancels before the counterparty funds:
cancel(swapId)

Withdraw
To claim ETH owed to an address:
withdraw()

Security Principles
The contract includes protections against:
• MEV frontrunning
• replay attacks
• griefing cycles
• timestamp manipulation
• hash collision attacks
• swap spam attacks
• collusion attempts
The system is designed under adversarial assumptions and aims to make attacks economically irrational.

Network
Designed for the Base network.
Supported assets:
• ETH
• USDC
USDC (Base Mainnet):
0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
Additional deployments may exist on test networks.

License
MIT

SecureEthUsdcSwapV3 is an experiment in economically secure peer‑to‑peer exchange mechanisms built entirely on‑chain.
