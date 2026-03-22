> SDK version: v9.0.0 | Audit date: 2026-03-21

## Protocol Fundamentals

### SDK Installation and Initialization

Install the Full Sail SDK and its peer dependency:

```bash
npm i @fullsailfinance/sdk@9.0.0
npm i @mysten/sui
```

> Current version: `9.0.0` (as of 2026-03-10). Check https://www.npmjs.com/package/@fullsailfinance/sdk for a newer release before publishing. The `@mysten/sui` peer dependency version requirement is declared in the SDK's package.json — check after install.

Initialize the SDK once at application startup:

```typescript
// Source: dist/index.d.ts verified against dist/index.js (2026-03-21)
import { initFullSailSDK } from '@fullsailfinance/sdk'

const fullSailSDK = initFullSailSDK({
  network: 'mainnet-production', // required — see valid values below
  fullNodeUrl: 'https://...',    // optional — uses default Full Sail RPC if omitted
  simulationAccount: '0x...',    // optional — required for preswap/simulation calls
})
```

**Valid `network` values (`SDKConfigNetwork`):**

| Value | Environment |
|-------|-------------|
| `'mainnet-dev'` | Development / staging |
| `'mainnet-production'` | Production mainnet |
| `'mainnet-prediction'` | Prediction market variant |

**The `network` parameter accepts exactly these three string literals. Passing any other value will cause the SDK to throw `SDKError('Unknown network: ...', 'InvalidNetwork')` at initialization.**

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

---

### Pool Data Sources: Chain Pool vs Backend Pool

Full Sail has two pool data sources. Using the wrong one for a transaction causes stale-state failures.

| Use Case | Method | Returns | Notes |
|----------|--------|---------|-------|
| Display only (TVL, APR, name, metadata) | `Pool.getById(poolId)` | Backend Pool | Pre-calculated; may lag by minutes |
| Building any transaction | `Pool.getByIdFromChain(poolId)` | Chain Pool | Real-time tick state; always current |
| Computing swap route | `Pool.getByIdFromChain(poolId)` | Chain Pool | Current tick index required for routing |
| Fetching `gauge_id` for staking | `Pool.getById(poolId)` | Backend Pool | `gauge_id` is a backend-calculated field |

**Hard rule: If the pool object will be passed to any `*Transaction` method, it MUST come from `Pool.getByIdFromChain(poolId)`.** Backend Pool data is acceptable only for read-only display.

---

### Token Types

Full Sail has three token types. Understanding each prevents routing and redemption errors.

| Token | Type | Purpose | Key Rule |
|-------|------|---------|----------|
| SAIL | Native governance token | Base liquid token; locked into veSAIL for governance | Never emitted directly; obtained by unlocking oSAIL (50% fee) or buying on market |
| oSAIL | Option token (weekly emission) | Earned by staked liquidity positions; redeemable for SAIL/USDC or lockable as veSAIL | Expires 5 weeks after epoch start in which issued; check before every operation |
| veSAIL | Vote-escrowed | Voting power derived from locked SAIL or oSAIL | Transferable and mergeable; voting weight resets on merge or split |

> **oSAIL is an option token — it is not DEX-tradeable. Do not use oSAIL in swap routes.**

---

### oSAIL Expiry Rule

> **Warning: oSAIL expires 5 weeks after the epoch start in which it was issued. Expired oSAIL cannot be redeemed for SAIL or USDC — it can only be locked as veSAIL.**

Rules:
- oSAIL is valid for redemption and locking for 5 weeks from the epoch it was issued
- After the 5-week window: the only valid operation is locking as veSAIL — only a 4-year (1461-day) or permanent lock is accepted; shorter durations fail
- Attempting to redeem expired oSAIL will fail

**Check oSAIL expiry before every oSAIL operation.** Do not assume oSAIL is valid. Verify the token's epoch against the current epoch before calling any oSAIL transaction method.

