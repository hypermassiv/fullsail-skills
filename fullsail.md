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
- After the 5-week window: the only valid operation is locking as veSAIL
- Attempting to redeem expired oSAIL will fail

**Check oSAIL expiry before every oSAIL operation.** Do not assume oSAIL is valid. Verify the token's epoch against the current epoch before calling any oSAIL transaction method.

## Swaps

The Full Sail SDK provides two swap paths — a router path (2 calls) and a direct pool path (4 calls).

### oSAIL Swap Restriction

> **oSAIL Swap Restriction:** oSAIL is an option token — it is not DEX-tradeable. Do not use oSAIL as `coinInType` or `coinOutType` in any swap route. To convert oSAIL to SAIL or USDC, use the redemption path — see `## Rewards and oSAIL`.

### Swap.getSwapRoute()

Returns an optimized route across pools via the Aftermath router.

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinInType` | `string` | Coin type address of the input token |
| `coinOutType` | `string` | Coin type address of the output token |
| `coinInAmount` | `bigint` | Amount of input token in base units |

Return value: Returns a `completeRoute` object — pass it directly to `swapRouterTransaction`. The route structure is opaque; do not enumerate or manipulate its fields.

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
const coinInType = '0x...'   // input token type address
const coinOutType = '0x...'  // output token type address
const coinInAmount = 1000000n // input amount in base units

const completeRoute = await fullSailSDK.Swap.getSwapRoute({
  coinInType,
  coinOutType,
  coinInAmount,
})
// completeRoute is opaque — pass directly to swapRouterTransaction
```

---

### Swap.swapRouterTransaction()

Builds the unsigned swap transaction using an Aftermath router route.

**Returns unsigned transaction — must be signed and submitted via `wallet.signAndExecuteTransaction()`.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `completeRoute` | `object` | Route object from `getSwapRoute()` — pass directly |
| `slippage` | `number` | Slippage as coefficient: use `Percentage.fromNumber(n).toCoefficient()` |

The `Percentage` class is used for slippage. It is likely exported from `@fullsailfinance/sdk` — verify the import if a NameError occurs.

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Router path — 2 calls: getSwapRoute → swapRouterTransaction

const coinInType = '0x...'   // input token type address
const coinOutType = '0x...'  // output token type address
const coinInAmount = 1000000n
// Percentage: likely exported from @fullsailfinance/sdk — verify import
const slippage = Percentage.fromNumber(1) // 1% slippage

const completeRoute = await fullSailSDK.Swap.getSwapRoute({
  coinInType,
  coinOutType,
  coinInAmount,
})

// Returns unsigned Transaction — must be signed and submitted separately
const transaction = await fullSailSDK.Swap.swapRouterTransaction({
  completeRoute,
  slippage: slippage.toCoefficient(),
})
const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Swap.preSwap()

Computes swap estimates for a direct pool swap. Must be called before `swapTransaction` — its output provides required parameters.

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA (assign using `isAtoB` logic — see below) |
| `coinTypeB` | `string` | Pool's coinTypeB |
| `decimalsA` | `number` | Decimals for coinTypeA |
| `decimalsB` | `number` | Decimals for coinTypeB |
| `poolId` | `string` | Pool object ID |
| `amount` | `bigint` | Swap amount (input if `byAmountIn=true`; desired output if `false`) |
| `byAmountIn` | `boolean` | `true`: amount is input; `false`: amount is desired output |
| `isAtoB` | `boolean` | Swap direction — always derive: `chainPool.coinTypeA === coinInType`. Never hardcode. |
| `currentSqrtPrice` | `number` | From Chain Pool (`getByIdFromChain`) — must be real-time |

Return value: Returns `{ estimatedAmountOut, estimatedAmountIn, swapParams }`. The `swapParams` field is an opaque object — spread it directly into `swapTransaction`: `...presSwap.swapParams`.

**Always derive `isAtoB` from the pool's `coinTypeA` field: `const isAtoB = chainPool.coinTypeA === coinInType`. Never hardcode `isAtoB`.**

**Use Chain Pool (`Pool.getByIdFromChain`) for `currentSqrtPrice`. Backend Pool data is stale and produces incorrect estimates.**

---

### Swap.swapTransaction()

Builds the unsigned swap transaction for a direct pool swap.

**Returns unsigned transaction — must be signed and submitted via `wallet.signAndExecuteTransaction()`.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `amount` | `bigint` | Same amount passed to `preSwap` |
| `amountLimit` | `bigint` | Min output if `byAmountIn=true` (`presSwap.estimatedAmountOut`); max input if `false` (`presSwap.estimatedAmountIn`) |
| `slippage` | `Percentage` | `Percentage` object (e.g., `Percentage.fromNumber(1)` for 1%) |
| `...presSwap.swapParams` | spread | Remaining parameters from `preSwap` output — spread directly |

---

### Direct Pool Swap: Complete 4-Step Sequence

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Direct pool path — 4 steps: Chain Pool → coin metadata → preSwap → swapTransaction

const poolId = '0x...'         // pool object ID — replace with real pool ID
const coinInType = '0x...'     // input token type address
const coinOutType = '0x...'    // output token type address
// Percentage: likely exported from @fullsailfinance/sdk — verify import
const slippage = Percentage.fromNumber(1) // 1% slippage
const amount = 1000000n        // input amount in base units
const byAmountIn = true        // true: amount is input; false: amount is desired output

// Step 1: Fetch Chain Pool — currentSqrtPrice must be real-time (NOT Backend Pool)
const chainPool = await fullSailSDK.Pool.getByIdFromChain(poolId)

// Step 2: Fetch coin metadata for decimals
const coinIn = await fullSailSDK.Coin.getByType(coinInType)
const coinOut = await fullSailSDK.Coin.getByType(coinOutType)

// Step 3: Compute estimates — derive isAtoB from pool ordering, never hardcode
const isAtoB = chainPool.coinTypeA === coinInType
const presSwap = await fullSailSDK.Swap.preSwap({
  coinTypeA: isAtoB ? coinInType : coinOutType,
  coinTypeB: isAtoB ? coinOutType : coinInType,
  decimalsA: isAtoB ? coinIn.decimals : coinOut.decimals,
  decimalsB: isAtoB ? coinOut.decimals : coinIn.decimals,
  poolId,
  amount,
  byAmountIn,
  isAtoB,
  currentSqrtPrice: chainPool.currentSqrtPrice,
})

// Step 4: Build unsigned transaction
// Returns unsigned Transaction — must be signed and submitted separately
const transaction = await fullSailSDK.Swap.swapTransaction({
  amount,
  amountLimit: byAmountIn ? presSwap.estimatedAmountOut : presSwap.estimatedAmountIn,
  slippage,
  ...presSwap.swapParams, // spreads required params from preSwap output — do not enumerate
})

// Step 5: Sign and submit
const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

## Liquidity Positions

This section covers the five position lifecycle operations (open, add, stake, remove, close) and tick range computation — all `*Transaction` calls return unsigned transactions that must be signed and submitted separately.

### Staked vs Unstaked Detection

Before calling any claim or liquidity method, fetch the position and inspect `stake_info` to determine the correct parameter set.

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
const position = await fullSailSDK.Position.getById(positionId)

const isStaked = !!position.stake_info
const positionStakeId = position.stake_info?.id // undefined if not staked

if (isStaked) {
  // Staked path: supply gaugeId, oSailCoinType, positionStakeId to staked-path methods
} else {
  // Unstaked path: omit gaugeId, oSailCoinType, positionStakeId entirely
}
```

| Field | Source | Value When Unstaked |
|-------|--------|---------------------|
| `position.stake_info` | `Position.getById(positionId)` | `undefined` — falsy |
| `positionStakeId` | `position.stake_info?.id` | `undefined` |

**Always fetch the position and check `position.stake_info` before any claim or liquidity operation. Staked and unstaked paths use different method sets.**

---

### Position.openTransaction()

Opens a new concentrated liquidity position. Supply `gaugeId` to auto-stake on creation and earn gauge rewards; omit `gaugeId` to leave the position unstaked.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Chain Pool |
| `coinTypeB` | `string` | Pool's coinTypeB — from Chain Pool |
| `poolId` | `string` | Pool object ID |
| `tickLower` | `number` | Lower tick boundary — derive with TickMath (see Ticks and Price Range) |
| `tickUpper` | `number` | Upper tick boundary — derive with TickMath (see Ticks and Price Range) |
| `amountA` | `bigint` | Desired amount of token A in base units |
| `amountB` | `bigint` | Max amount of token B in base units |
| `slippage` | `Percentage` | Slippage tolerance |
| `fixedAmountA` | `boolean` | `true`: fix token A amount; `false`: fix token B amount |
| `currentSqrtPrice` | `number` | From Chain Pool (`Pool.getByIdFromChain()`) — must be real-time |
| `gaugeId` | `string` | (optional) From Backend Pool (`Pool.getById()`) — omit to leave position unstaked |

**`gaugeId` comes from Backend Pool (`Pool.getById()`). `currentSqrtPrice` comes from Chain Pool (`Pool.getByIdFromChain()`). Both fetches are required.**

**If `gaugeId` is omitted, the position is created unstaked and earns no gauge rewards even if the pool has an active gauge.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Open path — dual pool fetch required: Chain Pool for tx params, Backend Pool for gauge_id

// Step 1: Fetch both pool sources
const chainPool = await fullSailSDK.Pool.getByIdFromChain(poolId) // currentSqrtPrice, tick state
const pool = await fullSailSDK.Pool.getById(poolId)               // gauge_id only

// Step 2: Compute tick boundaries — never hardcode tick values
const tickLower = TickMath.getPrevInitializableTickIndex(
  chainPool.currentTickIndex,
  chainPool.tickSpacing
)
const tickUpper = TickMath.getNextInitializableTickIndex(
  chainPool.currentTickIndex,
  chainPool.tickSpacing
)

// Step 3: Build unsigned transaction
// Returns unsigned Transaction — must be signed and submitted separately
const transaction = await fullSailSDK.Position.openTransaction({
  coinTypeA: chainPool.coinTypeA,
  coinTypeB: chainPool.coinTypeB,
  poolId,
  tickLower,
  tickUpper,
  amountA,
  amountB: maxAmountB,
  slippage,
  fixedAmountA,
  currentSqrtPrice: chainPool.currentSqrtPrice,
  gaugeId: pool.gauge_id, // omit to leave position unstaked; auto-stakes when provided
})

// Step 4: Sign and submit
const result = await wallet.signAndExecuteTransaction({ transaction })
// positionId is in result — extract from transaction effects
```

---

### Position.addLiquidityTransaction()

Adds liquidity to an existing position. Staked positions require three extra parameters; omitting them on a staked position will fail.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Chain Pool |
| `coinTypeB` | `string` | Pool's coinTypeB — from Chain Pool |
| `poolId` | `string` | Pool object ID |
| `positionId` | `string` | Position object ID |
| `tickLower` | `number` | `position.tick_lower` |
| `tickUpper` | `number` | `position.tick_upper` |
| `amountA` | `bigint` | Desired amount of token A in base units |
| `amountB` | `bigint` | Max amount of token B in base units |
| `slippage` | `Percentage` | Slippage tolerance |
| `fixedAmountA` | `boolean` | `true`: fix token A amount; `false`: fix token B amount |
| `currentSqrtPrice` | `number` | From Chain Pool — must be real-time |
| `gaugeId` | `string` | (staked only) From Backend Pool (`Pool.getById()`) |
| `oSailCoinType` | `string` | (staked only) From `Coin.getCurrentEpochOSail().address` |
| `positionStakeId` | `string` | (staked only) From `position.stake_info.id` |

**Only oSAIL is auto-claimed during this transaction. Trading fees and pool rewards are NOT auto-claimed — call the appropriate claim method separately.**

**For staked positions, all three extra params (`gaugeId`, `oSailCoinType`, `positionStakeId`) must be supplied together or not at all.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately
// NOTE: Only oSAIL auto-claimed in this transaction. Fees and pool rewards are not.

const chainPool = await fullSailSDK.Pool.getByIdFromChain(poolId)
const pool = await fullSailSDK.Pool.getById(poolId)               // needed for gauge_id (staked only)
const currentEpochOSail = await fullSailSDK.Coin.getCurrentEpochOSail()

const transaction = await fullSailSDK.Position.addLiquidityTransaction({
  coinTypeA: chainPool.coinTypeA,
  coinTypeB: chainPool.coinTypeB,
  poolId,
  positionId,
  tickLower: position.tick_lower,
  tickUpper: position.tick_upper,
  amountA,
  amountB: maxAmountB,
  slippage,
  fixedAmountA,
  currentSqrtPrice: chainPool.currentSqrtPrice,
  // Required only if position is staked — supply all three or none:
  gaugeId: pool.gauge_id,
  oSailCoinType: currentEpochOSail.address,
  positionStakeId: position.stake_info?.id,
})
```

---

### Position.stakeTransaction()

Stakes an existing unstaked position into a gauge. Use when the position was created without `gaugeId`, or after a gauge was added to the pool.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | From Backend Pool — `pool.token_a.address` |
| `coinTypeB` | `string` | From Backend Pool — `pool.token_b.address` |
| `poolId` | `string` | Pool object ID — `pool.address` |
| `positionId` | `string` | Position object ID |
| `gaugeId` | `string` | From Backend Pool — `pool.gauge_id` |

**Only trading fees are auto-claimed during `stakeTransaction` — pool rewards are not. This differs from `addLiquidityTransaction`, which auto-claims oSAIL.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Use for positions created without gaugeId — stakes into gauge retroactively
// Returns unsigned Transaction — must be signed and submitted separately
// NOTE: Only fees auto-claimed in this transaction. Pool rewards are not.

const pool = await fullSailSDK.Pool.getById(poolId)

const transaction = await fullSailSDK.Position.stakeTransaction({
  coinTypeA: pool.token_a.address,
  coinTypeB: pool.token_b.address,
  poolId: pool.address,
  positionId: position.id,
  gaugeId: pool.gauge_id,
})
```

---

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
