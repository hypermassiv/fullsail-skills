---
name: fullsail-dex
description: Practical skill for integrating Full Sail DEX on Sui — swaps, concentrated liquidity, veSAIL governance, vaults, and prediction voting using @fullsailfinance/sdk.
version: 1.0.0
capabilities:
  - swap-routing
  - concentrated-liquidity
  - position-management
  - reward-claiming
  - governance-voting
  - vault-integration
  - prediction-voting
last-validated: 2026-03-08
---

# Full Sail Skill

**Version:** 1.0.0
**Last validated:** 2026-03-08
**SDK:** `@fullsailfinance/sdk` (verify current version before use)
**Network:** Sui Mainnet

> This skill is self-contained. No external references are required to use it.

## Fast Routing

<!-- Phase 6: Written last after all domain sections exist. Maps common agent tasks to section anchors. -->

## Protocol Fundamentals

### SDK Installation and Initialization

Install the Full Sail SDK and its peer dependency:

```bash
npm i @fullsailfinance/sdk
npm i @mysten/sui
```

> Verify the current `@fullsailfinance/sdk` version at https://www.npmjs.com/package/@fullsailfinance/sdk before publishing. The `@mysten/sui` peer dependency version requirement is declared in the SDK's package.json — check after install.

Initialize the SDK once at application startup:

```typescript
import { initFullSailSDK } from '@fullsailfinance/sdk'

const fullSailSDK = initFullSailSDK({
  network: 'mainnet-production',
  fullNodeUrl: 'https://...',      // optional — uses default Full Sail RPC if omitted
  simulationAccount: '0x...',       // optional
})
```

---

### Two-Phase Sender Setup

The SDK requires a sender address before it can build transactions. Sender setup is a two-phase operation: initialize the SDK at app startup, then set the sender address after wallet connection.

**Rule: Never set the sender address at SDK initialization. Set it after wallet connection.**

```typescript
// WRONG — sender set at init before wallet is connected
const fullSailSDK = initFullSailSDK({ network: 'mainnet-production' })
fullSailSDK.senderAddress = storedAddress // potentially wrong — wallet not connected yet
```

```typescript
// CORRECT — sender set after wallet connection
const fullSailSDK = initFullSailSDK({ network: 'mainnet-production' })
// Phase 1 complete: SDK initialized, no sender yet

// ... later, after wallet connects:
const connectedAddress = await wallet.getAddress() // from your wallet adapter
fullSailSDK.senderAddress = connectedAddress
// Alternative: fullSailSDK.setSenderAddress(connectedAddress)
// Phase 2 complete: sender set, SDK ready to build transactions
```

---

### Transaction Lifecycle Model

**All methods ending in `Transaction` return unsigned `Transaction` objects. They do NOT execute on-chain. Nothing happens until you sign and submit.**

```typescript
// Step 1: Build — returns unsigned Transaction object (nothing happens on-chain yet)
const tx = await pool.swapTransaction({ /* params */ })

// Step 2: Sign and submit — only NOW does the operation execute on-chain
const result = await wallet.signAndExecuteTransaction({ transaction: tx })
```

This pattern applies to every `*Transaction` method in the SDK: `swapTransaction`, `addLiquidityTransaction`, `claimTransaction`, `lockTransaction`, and all others. The build step is safe to call for dry-run inspection. The submit step is irreversible.

## Swaps

<!-- Phase 2 -->

## Liquidity Positions

<!-- Phase 3 -->

## Rewards and oSAIL

<!-- Phase 3 -->

## Locks and veSAIL

<!-- Phase 4 -->

## Governance Voting

<!-- Phase 4 -->

## Vaults

<!-- Phase 5 -->

## Prediction Voting

<!-- Phase 5 -->

## Pitfalls

<!-- Phase 6 -->

## Source of Truth

Reference: https://docs.fullsail.finance

<!-- Phase 6: Complete with contract addresses, SDK npm link, and all canonical URLs. -->
