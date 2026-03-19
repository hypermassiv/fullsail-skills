---
name: fullsail-dex
description: Practical skill for integrating Full Sail DEX on Sui — swaps, concentrated liquidity, veSAIL governance, vaults, and prediction voting using @fullsailfinance/sdk.
version: 1.4.0
capabilities:
  - swap-routing
  - concentrated-liquidity
  - position-management
  - reward-claiming
  - governance-voting
  - vault-integration
  - prediction-voting
last-validated: 2026-03-20
---

# Full Sail Skill

**Version:** 1.3.0
**Last validated:** 2026-03-20
**SDK:** `@fullsailfinance/sdk@9.0.0`
**Network:** Sui Mainnet

> This skill directory contains 10 files. Start here for routing; each domain file is self-contained.

## Fast Routing

Use this table to jump directly to the relevant section for your current task.

| Task | Go to |
|------|-------|
| Install and initialize the SDK | [§Protocol Fundamentals > SDK Installation and Initialization](protocol-fundamentals.md#sdk-installation-and-initialization) |
| Set up sender address after wallet connect | [§Protocol Fundamentals > Two-Phase Sender Setup](protocol-fundamentals.md#two-phase-sender-setup) |
| Understand the transaction lifecycle (unsigned return) | [§Protocol Fundamentals > Transaction Lifecycle Model](protocol-fundamentals.md#transaction-lifecycle-model) |
| Choose between Chain Pool and Backend Pool | [§Protocol Fundamentals > Pool Data Sources](protocol-fundamentals.md#pool-data-sources-chain-pool-vs-backend-pool) |
| Execute a token swap (router path) | [§Swaps > Swap.swapRouterTransaction()](swaps.md#swapswaproutertransaction) |
| Execute a direct pool swap | [§Swaps > Direct Pool Swap: Complete 4-Step Sequence](swaps.md#direct-pool-swap-complete-4-step-sequence) |
| Open a liquidity position | [§Liquidity Positions > Position.openTransaction()](liquidity.md#positionopentransaction) |
| Add liquidity to an existing position | [§Liquidity Positions > Position.addLiquidityTransaction()](liquidity.md#positionaddliquiditytransaction) |
| Remove liquidity (staked or unstaked) | [§Liquidity Positions > Position.removeLiquidityTransaction()](liquidity.md#positionremoveliquiditytransaction) |
| Claim all rewards from a staked position | [§Rewards and oSAIL > Staked Position: claimOSailAndStakedPoolRewardsTransaction()](rewards.md#staked-position-claimosailandstakedpoolrewardstransaction) |
| Check oSAIL expiry and redeem | [§Rewards and oSAIL > oSAIL Expiry Check](rewards.md#osail-expiry-check) |
| Create a veSAIL lock | [§Locks and veSAIL > Lock.createLockTransaction()](locks.md#lockcreatelocktransaction) |
| Vote on governance pools | [§Governance Voting > Lock.batchVoteTransaction()](governance.md#lockbatchvotetransaction) |
| Deposit into a vault | [§Vaults > Vault Deposit](vaults.md#vault-deposit) (`PortEntry.createPortEntryTransaction()`) |
| Submit a prediction vote | [§Prediction Voting > Creating and Submitting a Prediction Vote](prediction-voting.md#creating-and-submitting-a-prediction-vote) |
| Debug a transaction failure | [§Pitfalls](pitfalls.md#pitfalls) |
