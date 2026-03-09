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

Use this table to jump directly to the relevant section for your current task.

| Task | Go to |
|------|-------|
| Install and initialize the SDK | [§Protocol Fundamentals > SDK Installation and Initialization](#sdk-installation-and-initialization) |
| Set up sender address after wallet connect | [§Protocol Fundamentals > Two-Phase Sender Setup](#two-phase-sender-setup) |
| Understand the transaction lifecycle (unsigned return) | [§Protocol Fundamentals > Transaction Lifecycle Model](#transaction-lifecycle-model) |
| Choose between Chain Pool and Backend Pool | [§Protocol Fundamentals > Pool Data Sources](#pool-data-sources-chain-pool-vs-backend-pool) |
| Execute a token swap (router path) | [§Swaps > Swap.swapRouterTransaction()](#swapswaproutertransaction) |
| Execute a direct pool swap | [§Swaps > Direct Pool Swap: Complete 4-Step Sequence](#direct-pool-swap-complete-4-step-sequence) |
| Open a liquidity position | [§Liquidity Positions > Position.openTransaction()](#positionopentransaction) |
| Add liquidity to an existing position | [§Liquidity Positions > Position.addLiquidityTransaction()](#positionaddliquiditytransaction) |
| Remove liquidity (staked or unstaked) | [§Liquidity Positions > Position.removeLiquidityTransaction()](#positionremoveliquiditytransaction) |
| Claim all rewards from a staked position | [§Rewards and oSAIL > Staked Position: claimOSailAndStakedPoolRewardsTransaction()](#staked-position-claimosailandstakedpoolrewardstransaction) |
| Check oSAIL expiry and redeem | [§Rewards and oSAIL > oSAIL Expiry Check](#osail-expiry-check) |
| Create a veSAIL lock | [§Locks and veSAIL > Lock.createLockTransaction()](#lockcreatelocktransaction) |
| Vote on governance pools | [§Governance Voting > Lock.batchVoteTransaction()](#lockbatchvotetransaction) |
| Deposit into a vault | [§Vaults > Vault Deposit](#vault-deposit) |
| Submit a prediction vote | [§Prediction Voting > Creating and Submitting a Prediction Vote](#creating-and-submitting-a-prediction-vote) |
| Debug a transaction failure | [§Pitfalls](#pitfalls) |

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

### Position.removeLiquidityTransaction()

> **PRECONDITION (Staked Positions):** For staked positions, supply `positionStakeId`, `gaugeId`, and `oSailCoinType` directly to `removeLiquidityTransaction`. No separate unstake call is required or documented in the SDK. Omitting these params on a staked position will fail.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Chain Pool |
| `coinTypeB` | `string` | Pool's coinTypeB — from Chain Pool |
| `poolId` | `string` | Pool object ID |
| `positionId` | `string` | Position object ID |
| `tickLower` | `number` | `position.tick_lower` |
| `tickUpper` | `number` | `position.tick_upper` |
| `currentSqrtPrice` | `number` | From Chain Pool — must be real-time |
| `liquidity` | `bigint` | Liquidity units to remove — derive from position object |
| `slippage` | `Percentage` | Slippage tolerance |
| `gaugeId` | `string` | (staked only) From Backend Pool (`Pool.getById()`) |
| `oSailCoinType` | `string` | (staked only) From `Coin.getCurrentEpochOSail().address` |
| `positionStakeId` | `string` | (staked only) From `position.stake_info.id` |

**Only oSAIL is auto-claimed during this transaction. Trading fees and pool rewards require separate claim calls.**

**`liquidity` is the number of liquidity units to remove — derive from the position object, not from token amounts.**

```typescript
// WRONG — no separate unstakeTransaction exists in the SDK
const unstakeTx = await fullSailSDK.Position.unstakeTransaction({ positionId }) // ERROR: method does not exist
const removeTx = await fullSailSDK.Position.removeLiquidityTransaction({ positionId, liquidity, ... })
```

```typescript
// CORRECT — pass staked params directly to removeLiquidityTransaction
const removeTx = await fullSailSDK.Position.removeLiquidityTransaction({
  ...,
  positionStakeId: position.stake_info.id, // handles staked case; no separate unstake call
})
```

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately
// NOTE: Only oSAIL auto-claimed in this transaction. Fees and pool rewards are not.

const chainPool = await fullSailSDK.Pool.getByIdFromChain(poolId)
const pool = await fullSailSDK.Pool.getById(poolId)               // needed for gauge_id (staked only)
const currentEpochOSail = await fullSailSDK.Coin.getCurrentEpochOSail()

const transaction = await fullSailSDK.Position.removeLiquidityTransaction({
  coinTypeA: chainPool.coinTypeA,
  coinTypeB: chainPool.coinTypeB,
  poolId,
  positionId,
  tickLower: position.tick_lower,
  tickUpper: position.tick_upper,
  currentSqrtPrice: chainPool.currentSqrtPrice,
  liquidity, // derive from position object — not from token amounts
  slippage,
  // Required only if position is staked — no separate unstake call needed:
  gaugeId: pool.gauge_id,
  oSailCoinType: currentEpochOSail.address,
  positionStakeId: position.stake_info?.id,
})
```

---

### Position.closeTransaction()

Closes the position, removes all liquidity, and claims all rewards (oSAIL, fees, and pool rewards) in a single atomic transaction.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Chain Pool |
| `coinTypeB` | `string` | Pool's coinTypeB — from Chain Pool |
| `poolId` | `string` | Pool object ID |
| `positionId` | `string` | Position object ID |
| `tickLower` | `number` | `position.tick_lower` |
| `tickUpper` | `number` | `position.tick_upper` |
| `currentSqrtPrice` | `number` | From Chain Pool — must be real-time |
| `liquidity` | `bigint` | `position.liquidity` — full position liquidity |
| `slippage` | `Percentage` | Slippage tolerance |
| `rewardCoinTypes` | `string[]` | Complete list of all reward coin type addresses for this pool — must be exhaustive |
| `gaugeId` | `string` | (staked only) From Backend Pool (`Pool.getById()`) |
| `oSailCoinType` | `string` | (staked only) From `Coin.getCurrentEpochOSail().address` |
| `positionStakeId` | `string` | (staked only) From `position.stake_info.id` |

**`rewardCoinTypes` must be the complete list of all reward coin type addresses for this pool. An incomplete array causes partial reward loss — rewards not listed are not claimed.**

**All rewards (oSAIL, fees, pool rewards) are claimed atomically within `closeTransaction`. No separate claim calls are needed after close.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately
// All rewards claimed atomically — no follow-up claim calls needed

const chainPool = await fullSailSDK.Pool.getByIdFromChain(poolId)
const pool = await fullSailSDK.Pool.getById(poolId)               // needed for gauge_id (staked only)
const currentEpochOSail = await fullSailSDK.Coin.getCurrentEpochOSail()

const transaction = await fullSailSDK.Position.closeTransaction({
  coinTypeA: chainPool.coinTypeA,
  coinTypeB: chainPool.coinTypeB,
  poolId,
  positionId,
  tickLower: position.tick_lower,
  tickUpper: position.tick_upper,
  currentSqrtPrice: chainPool.currentSqrtPrice,
  liquidity: position.liquidity,
  slippage,
  rewardCoinTypes: [/* all reward coin type addresses — must be exhaustive; partial = partial claim */],
  // Required only if position is staked:
  gaugeId: pool.gauge_id,
  oSailCoinType: currentEpochOSail.address,
  positionStakeId: position.stake_info?.id,
})
```

---

### Ticks and Price Range

Concentrated liquidity positions are defined by a price range expressed as tick indices (`tick_lower`, `tick_upper`). Ticks must be exact multiples of the pool's `tickSpacing`. Use `TickMath` helpers to compute valid tick boundaries from the current pool state — never hardcode tick values.

| Field | Source | Description |
|-------|--------|-------------|
| `tick_lower` | `position.tick_lower` | Lower boundary tick index |
| `tick_upper` | `position.tick_upper` | Upper boundary tick index |
| `tickSpacing` | `chainPool.tickSpacing` | Pool's minimum tick step — from Chain Pool |
| `currentTickIndex` | `chainPool.currentTickIndex` | Pool's current active tick — from Chain Pool |

**Never hardcode tick values. Always derive tick boundaries using TickMath helpers with `chainPool.currentTickIndex` and `chainPool.tickSpacing`. Hardcoded ticks that are not multiples of `tickSpacing` will create un-initializable positions.**

**`TickMath` is imported from `@fullsailfinance/sdk` — verify import path before use.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// TickMath: verify import from @fullsailfinance/sdk
// Always derive tick boundaries from pool state — never hardcode tick values.
// Ticks must be multiples of tickSpacing; TickMath enforces this automatically.

const chainPool = await fullSailSDK.Pool.getByIdFromChain(poolId)

const tickLower = TickMath.getPrevInitializableTickIndex(
  chainPool.currentTickIndex,
  chainPool.tickSpacing
)
const tickUpper = TickMath.getNextInitializableTickIndex(
  chainPool.currentTickIndex,
  chainPool.tickSpacing
)
```

---

## Rewards and oSAIL

This section covers reward claim patterns split by stake state. Before selecting a claim method, determine whether a position is staked or unstaked using the detection pattern in ## Liquidity Positions.

### Reward Claim Matrix

| Reward Type | When Triggered | Auto-claimed? | Explicit Call (unstaked) | Explicit Call (staked) |
|-------------|----------------|---------------|--------------------------|------------------------|
| oSAIL emissions | Liquidity add, remove, close | YES — during liquidity ops | — | `claimOSailTransaction` (standalone) |
| Trading fees | Continuous | NO | `claimFeeTransaction` | Not applicable (unstake required) |
| Pool staking rewards | Continuous | NO | `claimUnstakedPoolRewardsTransaction` | `claimStakedPoolRewardsTransaction` |
| All (combined) | On close | YES via `closeTransaction` | `claimFeeAndUnstakedPoolRewardsTransaction` | `claimOSailAndStakedPoolRewardsTransaction` |

**oSAIL is the only reward type auto-claimed during liquidity operations. Trading fees and pool staking rewards always require an explicit claim call.**

**Staked and unstaked positions use different claim method sets — never mix them. Check `position.stake_info` before selecting a method.**

---

### Staked vs Unstaked Claim Path

```
1. Fetch position: await fullSailSDK.Position.getById(positionId)
2. Check stake status: const isStaked = !!position.stake_info
3. If NOT staked → use unstaked methods:
   - Fees only:            claimFeeTransaction
   - Pool rewards only:    claimUnstakedPoolRewardsTransaction
   - Fees + pool rewards:  claimFeeAndUnstakedPoolRewardsTransaction
4. If STAKED → use staked methods (requires positionStakeId, gaugeId, oSailCoinType):
   - oSAIL only:                          claimOSailTransaction
   - Pool rewards only:                   claimStakedPoolRewardsTransaction
   - oSAIL + pool rewards:                claimOSailAndStakedPoolRewardsTransaction
```

**`getCurrentEpochOSail()` must be called fresh before every staked claim operation. Never reuse a cached oSailCoinType value across sessions or epoch boundaries.**

---

### Unstaked Position: claimFeeTransaction()

Claims accrued trading fees for an unstaked position.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Backend Pool `pool.token_a.address` |
| `coinTypeB` | `string` | Pool's coinTypeB — from Backend Pool `pool.token_b.address` |
| `poolId` | `string` | Pool object ID |
| `positionId` | `string` | Position object ID |

**Precondition: position must be unstaked (`position.stake_info` is undefined). Calling this on a staked position will fail.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately

const pool = await fullSailSDK.Pool.getById(poolId)

const transaction = await fullSailSDK.Position.claimFeeTransaction({
  coinTypeA: pool.token_a.address,
  coinTypeB: pool.token_b.address,
  poolId,
  positionId,
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Unstaked Position: claimUnstakedPoolRewardsTransaction()

Claims accrued pool staking rewards for an unstaked position.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Backend Pool `pool.token_a.address` |
| `coinTypeB` | `string` | Pool's coinTypeB — from Backend Pool `pool.token_b.address` |
| `poolId` | `string` | Pool object ID |
| `positionId` | `string` | Position object ID |
| `rewardCoinTypes` | `string[]` | Array of all reward coin type addresses for this pool |

**`rewardCoinTypes` must include all reward coin type addresses for this pool. Derive from the pool object's reward token list — verify the field name from `Pool.getById()` return shape.**

**Precondition: position must be unstaked.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately

const pool = await fullSailSDK.Pool.getById(poolId)
// rewardCoinTypes: derive from pool object reward token list — verify field name from Pool.getById() return shape
const rewardCoinTypes = pool.reward_tokens?.map((t: any) => t.address) ?? []

const transaction = await fullSailSDK.Position.claimUnstakedPoolRewardsTransaction({
  coinTypeA: pool.token_a.address,
  coinTypeB: pool.token_b.address,
  poolId,
  positionId,
  rewardCoinTypes,
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Unstaked Position: claimFeeAndUnstakedPoolRewardsTransaction()

Combined call — claims both trading fees and pool rewards in a single transaction. Prefer this over two separate calls.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Backend Pool `pool.token_a.address` |
| `coinTypeB` | `string` | Pool's coinTypeB — from Backend Pool `pool.token_b.address` |
| `poolId` | `string` | Pool object ID |
| `positionId` | `string` | Position object ID |
| `rewardCoinTypes` | `string[]` | Array of all reward coin type addresses for this pool |

**Precondition: position must be unstaked. This combined method replaces both `claimFeeTransaction` and `claimUnstakedPoolRewardsTransaction` — do not call all three.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately

const pool = await fullSailSDK.Pool.getById(poolId)
// rewardCoinTypes: derive from pool object reward token list — verify field name from Pool.getById() return shape
const rewardCoinTypes = pool.reward_tokens?.map((t: any) => t.address) ?? []

const transaction = await fullSailSDK.Position.claimFeeAndUnstakedPoolRewardsTransaction({
  coinTypeA: pool.token_a.address,
  coinTypeB: pool.token_b.address,
  poolId,
  positionId,
  rewardCoinTypes,
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Staked Position: claimOSailTransaction()

Claims accrued oSAIL emissions for a staked position.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Backend Pool `pool.token_a.address` |
| `coinTypeB` | `string` | Pool's coinTypeB — from Backend Pool `pool.token_b.address` |
| `poolId` | `string` | Pool object ID |
| `positionStakeId` | `string` | Stake object ID — `position.stake_info.id` |
| `oSailCoinType` | `string` | Current epoch oSAIL coin type — `currentEpochOSail.address` from `fullSailSDK.Coin.getCurrentEpochOSail()` |
| `gaugeId` | `string` | From Backend Pool — `pool.gauge_id` |

**Precondition: position must be staked (`position.stake_info` exists). `positionStakeId` is `position.stake_info.id` — never hardcode.**

**`oSailCoinType` is `currentEpochOSail.address` from `fullSailSDK.Coin.getCurrentEpochOSail()`. Call this fresh — do not cache.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately

const pool = await fullSailSDK.Pool.getById(poolId)
const position = await fullSailSDK.Position.getById(positionId)
// Call getCurrentEpochOSail() fresh — never use a cached value
const currentEpochOSail = await fullSailSDK.Coin.getCurrentEpochOSail()

const transaction = await fullSailSDK.Position.claimOSailTransaction({
  coinTypeA: pool.token_a.address,
  coinTypeB: pool.token_b.address,
  poolId,
  positionStakeId: position.stake_info.id, // never hardcode — always from position.stake_info.id
  oSailCoinType: currentEpochOSail.address, // fresh call required — not cached
  gaugeId: pool.gauge_id,
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Staked Position: claimStakedPoolRewardsTransaction()

Claims accrued pool staking rewards for a staked position.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Backend Pool `pool.token_a.address` |
| `coinTypeB` | `string` | Pool's coinTypeB — from Backend Pool `pool.token_b.address` |
| `poolId` | `string` | Pool object ID |
| `positionStakeId` | `string` | Stake object ID — `position.stake_info.id` |
| `gaugeId` | `string` | From Backend Pool — `pool.gauge_id` |
| `rewardCoinTypes` | `string[]` | Array of all reward coin type addresses for this pool |

**Precondition: position must be staked.**

**`rewardCoinTypes` must be exhaustive — see note in claimUnstakedPoolRewardsTransaction for derivation.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately

const pool = await fullSailSDK.Pool.getById(poolId)
const position = await fullSailSDK.Position.getById(positionId)
// rewardCoinTypes: derive from pool object reward token list — verify field name from Pool.getById() return shape
const rewardCoinTypes = pool.reward_tokens?.map((t: any) => t.address) ?? []

const transaction = await fullSailSDK.Position.claimStakedPoolRewardsTransaction({
  coinTypeA: pool.token_a.address,
  coinTypeB: pool.token_b.address,
  poolId,
  positionStakeId: position.stake_info.id,
  gaugeId: pool.gauge_id,
  rewardCoinTypes,
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Staked Position: claimOSailAndStakedPoolRewardsTransaction()

Combined call — claims both oSAIL and pool rewards in a single transaction for staked positions. Prefer this over two separate calls.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Backend Pool `pool.token_a.address` |
| `coinTypeB` | `string` | Pool's coinTypeB — from Backend Pool `pool.token_b.address` |
| `poolId` | `string` | Pool object ID |
| `positionStakeId` | `string` | Stake object ID — `position.stake_info.id` |
| `gaugeId` | `string` | From Backend Pool — `pool.gauge_id` |
| `oSailCoinType` | `string` | Current epoch oSAIL coin type — `currentEpochOSail.address` from `fullSailSDK.Coin.getCurrentEpochOSail()` |
| `rewardCoinTypes` | `string[]` | Array of all reward coin type addresses for this pool |

**Precondition: position must be staked. This combined method replaces both `claimOSailTransaction` and `claimStakedPoolRewardsTransaction` — do not call all three.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately

const pool = await fullSailSDK.Pool.getById(poolId)
const position = await fullSailSDK.Position.getById(positionId)
// Call getCurrentEpochOSail() fresh — never use a cached value
const currentEpochOSail = await fullSailSDK.Coin.getCurrentEpochOSail()
// rewardCoinTypes: derive from pool object reward token list — verify field name from Pool.getById() return shape
const rewardCoinTypes = pool.reward_tokens?.map((t: any) => t.address) ?? []

const transaction = await fullSailSDK.Position.claimOSailAndStakedPoolRewardsTransaction({
  coinTypeA: pool.token_a.address,
  coinTypeB: pool.token_b.address,
  poolId,
  positionStakeId: position.stake_info.id,
  gaugeId: pool.gauge_id,
  oSailCoinType: currentEpochOSail.address, // fresh call required — not cached
  rewardCoinTypes,
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### oSAIL Expiry Check

oSAIL expires 5 weeks after the epoch start in which it was issued. Each epoch has its own oSAIL coin type with a unique address. `getCurrentEpochOSail()` returns the CURRENT epoch's oSAIL — using a prior epoch's coin type will fail or produce no result.

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Call fresh before every oSAIL operation — never cache across sessions.
const currentEpochOSail = await fullSailSDK.Coin.getCurrentEpochOSail()
// currentEpochOSail.address — current epoch oSAIL coin type address
// Expiry field name on returned object: verify from SDK source
```

**Call `getCurrentEpochOSail()` fresh before every oSAIL operation. A cached value from a previous session or epoch will be wrong.**

**If a position holds oSAIL from a prior epoch, the current epoch's oSAIL coin type will differ — the prior epoch oSAIL is expired and cannot be claimed via the current epoch type.**

**The oSAIL Expiry Rule is also documented in ## Protocol Fundamentals — see that section for the 5-week expiry window rule.**

---

### oSAIL Redemption Options

**Option A: Lock as veSAIL (available even after oSAIL has expired)**

- Use `Lock.createLockFromOSailTransaction()` for standalone oSAIL-to-veSAIL conversion
- Use `Lock.claimOSailAndCreateLockTransaction()` to claim oSAIL from a position and lock it in one transaction
- Full lock parameters documented in ## Locks and veSAIL (Phase 4)

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Lock oSAIL as veSAIL — even expired oSAIL can be locked
const lockTx = await fullSailSDK.Lock.createLockFromOSailTransaction({
  // exact params — see ## Locks and veSAIL for full signature
})
```

**Option B: Redeem for SAIL + USDC (50% of SAIL spot price at redemption, paid in USDC)**

- No SDK method confirmed for this path — may be protocol UI only
- Verify from SDK docs before attempting programmatic redemption. If no SDK method exists, use the Full Sail app UI.

**The lock path (Option A) has confirmed SDK methods. The SAIL+USDC redemption path (Option B) has no confirmed SDK method — verify before implementing.**

---

### Epoch Cycle

Full Sail uses a 7-day voting epoch cycle. Each epoch starts a new oSAIL emission token with a unique coin type. oSAIL expires 5 weeks after the epoch start in which it was issued. Voting windows open at epoch start; governance votes affect gauge weight allocations that determine pool reward distribution.

| Property | Value |
|----------|-------|
| Epoch duration | 7 days |
| oSAIL expiry window | 5 weeks after issuing epoch start |
| Impact on claims | `claimOSailTransaction` must use current epoch's `oSailCoinType` |

**Epoch timing affects oSAIL expiry calculations. If operating near an epoch boundary, oSAIL may expire between when it was earned and when the claim transaction is submitted.**

---

## Locks and veSAIL

The veSAIL lock lifecycle — create, increase, merge, split, transfer, and toggle permanent status — is managed via the `Lock` namespace. All `*Transaction` methods return unsigned `Transaction` objects and must be signed and submitted separately. Governance voting (`batchVoteTransaction`) is documented in ## Governance Voting.

### Lock.createLockTransaction()

Creates a new veSAIL lock by depositing SAIL tokens.

| Parameter | Type | Description |
|-----------|------|-------------|
| amount | bigint | SAIL amount to lock (9 decimal places) |
| isPermanent | boolean | true = permanent lock (no expiry, max voting power) |
| durationDays | number | Lock duration in days — ignored if isPermanent is true |

**Returns unsigned Transaction — must be signed and submitted separately.**

**`createLockTransaction` accepts exactly three parameters: `amount`, `isPermanent`, `durationDays`. There is no `sailCoinType` parameter — SAIL is implied. Passing an unknown parameter will cause the call to fail silently or error.**

**If `isPermanent` is true, `durationDays` is ignored. Permanent locks hold max voting power with no expiry.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-09)
// Returns unsigned Transaction — must be signed and submitted separately
// No sailCoinType parameter — SAIL is implied

const transaction = await fullSailSDK.Lock.createLockTransaction({
  amount: 1000000n,        // SAIL amount in base units (9 decimals)
  isPermanent: false,      // true = permanent lock (max voting power, no expiry)
  durationDays: 365 * 4,  // lock duration in days
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Lock.createLockFromOSailTransaction()

Creates a new veSAIL lock by converting oSAIL tokens. Requires a fresh oSAIL coin type from the current epoch.

| Parameter | Type | Description |
|-----------|------|-------------|
| oSailCoinType | string | Current epoch oSAIL coin type — from `getCurrentEpochOSail().address` |
| amount | bigint | oSAIL amount to lock (9 decimal places) |
| isPermanent | boolean | true = permanent lock (no expiry, max voting power) |
| durationDays | number | Lock duration in days — ignored if isPermanent is true |

**Returns unsigned Transaction — must be signed and submitted separately.**

**`oSailCoinType` must come from `await fullSailSDK.Coin.getCurrentEpochOSail()` called fresh before this transaction. Never cache the coin type across sessions or epoch boundaries — the oSAIL coin type changes every epoch.**

**This method converts oSAIL to veSAIL in a single transaction. For a combined oSAIL claim + lock creation from a staked position, see `Lock.claimOSailAndCreateLockTransaction()`.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-09)
// Returns unsigned Transaction — must be signed and submitted separately
// Call getCurrentEpochOSail() fresh — never use a cached value

const currentEpochOSail = await fullSailSDK.Coin.getCurrentEpochOSail()

const transaction = await fullSailSDK.Lock.createLockFromOSailTransaction({
  amount: 1000000n,
  oSailCoinType: currentEpochOSail.address, // fresh call required — not cached
  durationDays: 365 * 4,
  isPermanent: false,
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Lock.increaseAmountTransaction()

Adds more SAIL to an existing lock without changing its duration or permanence status.

| Parameter | Type | Description |
|-----------|------|-------------|
| lockId | string | ID of the existing lock to increase |
| amount | bigint | Additional SAIL to add (9 decimal places) |

**Returns unsigned Transaction — must be signed and submitted separately.**

**`increaseAmountTransaction` adds SAIL to the lock — it does not change duration or permanence status. To extend duration, use `Lock.increaseDurationTransaction()`.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-09)
// Returns unsigned Transaction — must be signed and submitted separately

const transaction = await fullSailSDK.Lock.increaseAmountTransaction({
  lockId,
  amount: 500000n, // additional SAIL to add (9 decimals)
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Lock.increaseDurationTransaction()

Extends the duration of an existing time-limited lock. Not applicable to permanent locks.

| Parameter | Type | Description |
|-----------|------|-------------|
| lockId | string | ID of the existing lock to extend |
| durationDays | number | New duration in days — see CAUTION below |

**Returns unsigned Transaction — must be signed and submitted separately.**

**CAUTION (LOW CONFIDENCE): Whether `durationDays` adds to the lock's existing remaining duration or sets an absolute new total duration is NOT confirmed in the SDK documentation. Test with a non-permanent lock on a testnet before using in production.**

**Not applicable to permanent locks — permanent locks have no expiry and cannot have their duration modified via this method.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-09)
// Returns unsigned Transaction — must be signed and submitted separately
// CAUTION: Whether durationDays adds to existing duration or sets absolute duration
// is NOT confirmed in SDK docs — verify behavior before use in production.

const transaction = await fullSailSDK.Lock.increaseDurationTransaction({
  lockId,
  durationDays: 365 * 4, // days — verify add-vs-replace semantics before use
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Lock.mergeTransaction()

> **WARNING — IRREVERSIBLE:** Merging resets votes from the affected locks. If either lock has active governance votes, those votes will be lost when this transaction executes. You must recast votes after the merge. This cannot be undone — confirm both locks have no active votes, or accept that votes from both locks will be reset.
>
> **Additionally:** If either lock has Auto-Max Lock (permanent) enabled, disable it using `Lock.disablePermanentTransaction()` before merging.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| fromLockId | string | Lock to merge from — this lock is destroyed after merge |
| toLockId | string | Lock to merge into — receives all SAIL from fromLock |
| isFromLockVoted | boolean | true if fromLock has active governance votes |
| isToLockVoted | boolean | true if toLock has active governance votes |

**`fromLockId` is destroyed after the merge. All SAIL is consolidated into `toLockId`. This is irreversible.**

**Vote reset is scoped to the affected locks only — other locks owned by the same wallet are not affected.**

**Always pass the correct `isFromLockVoted` and `isToLockVoted` values. These signal the SDK to handle vote cleanup for the affected locks.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-09)
// Returns unsigned Transaction — must be signed and submitted separately
// WARNING: Votes from affected locks are reset — recast after merge
// fromLockId is destroyed; toLockId receives all SAIL

const transaction = await fullSailSDK.Lock.mergeTransaction({
  fromLockId,    // lock to merge from — destroyed after merge
  toLockId,      // lock to merge into — receives combined SAIL
  isFromLockVoted: false, // set true if fromLock has active governance votes
  isToLockVoted: false,   // set true if toLock has active governance votes
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Lock.splitTransaction()

> **WARNING — IRREVERSIBLE:** Splitting resets votes from the affected lock. If the lock has active governance votes, those votes will be lost when this transaction executes. You must recast votes after the split. This cannot be undone — confirm the lock has no active votes, or accept that its votes will be reset.

**Returns unsigned Transaction — must be signed and submitted separately.**

| Parameter | Type | Description |
|-----------|------|-------------|
| lockId | string | Lock to split |
| amount | bigint | SAIL amount to move into the newly created lock (9 decimal places) |
| isVoted | boolean | true if the lock currently has active governance votes |

**`amount` determines how much SAIL moves into the new lock. The original lock retains the remainder. Both locks persist after the split.**

**Vote reset is scoped to the affected lock only — the original lock's votes are reset; other locks owned by the same wallet are not affected.**

**The newly created lock starts with no votes. Recast votes on both locks after splitting if governance participation is required.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-09)
// Returns unsigned Transaction — must be signed and submitted separately
// WARNING: Lock votes are reset — recast after split

const transaction = await fullSailSDK.Lock.splitTransaction({
  lockId,
  amount: 500000n, // SAIL amount to split into a new lock (9 decimals)
  isVoted: false,  // set true if lock has active governance votes
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Lock.transferTransaction()

Transfers ownership of a lock to another Sui wallet address. After transfer, the recipient controls the lock and the sender has no further access.

| Parameter | Type | Description |
|-----------|------|-------------|
| lockId | string | Lock to transfer |
| receiver | string | Recipient Sui wallet address (0x...) |

**Returns unsigned Transaction — must be signed and submitted separately.**

**After transfer, the original wallet loses all control of the lock. This is irreversible — verify the recipient address before submitting.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-09)
// Returns unsigned Transaction — must be signed and submitted separately

const transaction = await fullSailSDK.Lock.transferTransaction({
  lockId,
  receiver: '0x...', // recipient wallet address — verify before submitting
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Lock.enablePermanentTransaction() and disablePermanentTransaction()

Toggles the permanent status of a lock. Permanent locks have no expiry and provide maximum voting power proportional to SAIL amount.

**enablePermanentTransaction:**

| Parameter | Type | Description |
|-----------|------|-------------|
| lockId | string | Lock to make permanent |

**disablePermanentTransaction:**

| Parameter | Type | Description |
|-----------|------|-------------|
| lockId | string | Permanent lock to convert back to time-limited |

**Both methods return unsigned Transaction — must be signed and submitted separately.**

**Disable permanent status using `disablePermanentTransaction()` before calling `mergeTransaction()` or `splitTransaction()` if the lock has Auto-Max Lock (permanent) enabled.**

**After `disablePermanentTransaction()`, the lock becomes time-limited. Duration behavior after disable is not explicitly documented in SDK — verify from SDK source if precision is needed.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-09)
// Returns unsigned Transaction — must be signed and submitted separately
// Permanent = max voting power with no expiry; disable returns to time-weighted lock

// Enable permanent lock
const enableTx = await fullSailSDK.Lock.enablePermanentTransaction({ lockId })
const result1 = await wallet.signAndExecuteTransaction({ transaction: enableTx })

// Disable permanent lock (call before mergeTransaction or splitTransaction if lock is permanent)
const disableTx = await fullSailSDK.Lock.disablePermanentTransaction({ lockId })
const result2 = await wallet.signAndExecuteTransaction({ transaction: disableTx })
```

---

### Lock.claimOSailAndCreateLockTransaction()

Combines oSAIL claim from a staked position and veSAIL lock creation in a single atomic transaction. Equivalent to calling `claimOSailTransaction` then `createLockFromOSailTransaction` but in one submission.

| Parameter | Type | Description |
|-----------|------|-------------|
| coinTypeA | string | Pool token A coin type — from Backend Pool `token_a.address` |
| coinTypeB | string | Pool token B coin type — from Backend Pool `token_b.address` |
| poolId | string | Pool ID |
| gaugeId | string | Gauge ID — from Backend Pool `gauge_id` |
| positionStakeId | string | Stake ID — from `position.stake_info.id`; position must be staked |
| oSailCoinType | string | Current epoch oSAIL coin type — from `getCurrentEpochOSail().address` |
| durationDays | number | Lock duration in days — ignored if isPermanent is true |
| isPermanent | boolean | true = permanent lock (no expiry, max voting power) |

**Returns unsigned Transaction — must be signed and submitted separately.**

**Precondition: position must be staked (`position.stake_info` exists). `positionStakeId` is `position.stake_info.id` — verify stake status before calling.**

**`oSailCoinType` must come from `await fullSailSDK.Coin.getCurrentEpochOSail()` called fresh before this transaction. Never cache.**

**This method requires both a Backend Pool fetch (for coinTypeA, coinTypeB, gaugeId) and a Position fetch (for positionStakeId). Perform both before calling.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-09)
// Returns unsigned Transaction — must be signed and submitted separately
// Call getCurrentEpochOSail() fresh — never use a cached value

const pool = await fullSailSDK.Pool.getById(poolId)
const position = await fullSailSDK.Position.getById(positionId)
const currentEpochOSail = await fullSailSDK.Coin.getCurrentEpochOSail()

const transaction = await fullSailSDK.Lock.claimOSailAndCreateLockTransaction({
  coinTypeA: pool.token_a.address,
  coinTypeB: pool.token_b.address,
  poolId,
  gaugeId: pool.gauge_id,
  positionStakeId: position.stake_info.id, // position must be staked
  oSailCoinType: currentEpochOSail.address, // fresh call required — not cached
  durationDays: 365 * 4,
  isPermanent: false,
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

## Governance Voting

Governance voting allocates veSAIL voting power across liquidity pools via the `Lock` namespace. Vote weight determines each pool's share of protocol trading fees for the epoch. All voting is done through `Lock.batchVoteTransaction()` — there is no separate Governance namespace.

### Epoch Cycle

| Property | Value |
|----------|-------|
| Epoch duration | 7 days |
| Epoch boundary | 00:00 UTC every Thursday (epoch starts and ends) |
| Voting window opens | 01:00 UTC Thursday (1 hour after epoch start) |
| Voting window closes | 23:00 UTC the following Wednesday (1 hour before epoch end) |
| Effect of no vote | No fee share for that epoch — lock earns no governance rewards |

Votes cast during the voting window allocate veSAIL voting power across liquidity pools. The pool weight determines what share of protocol trading fees that pool's liquidity providers receive for the epoch. Votes reset at the start of each new epoch — recast votes each epoch to maintain fee share participation.

**If no vote is cast in an epoch, the lock earns no governance fee share for that epoch. There is no carry-forward from prior epochs.**

**Votes can be adjusted at any point during the voting window by calling `Lock.batchVoteTransaction()` again. A new call replaces the prior vote allocation from the specified locks (MEDIUM confidence — functionally implied by "adjust votes" language; verify from SDK source if exact replace-vs-accumulate behavior is critical).**

---

### Lock.batchVoteTransaction()

Allocates veSAIL voting power from one or more locks across liquidity pools. Voting determines fee distribution for the epoch.

| Parameter | Type | Description |
|-----------|------|-------------|
| lockIds | string[] | All lock IDs contributing voting power in this batch |
| votes | Array<{ poolId: string, weight: bigint, volume: bigint }> | Vote allocations per pool |

**votes array members:**

| Field | Type | Description |
|-------|------|-------------|
| poolId | string | Pool to allocate voting weight to |
| weight | bigint | Abstract ratio units — any values; SDK normalizes total to 100% |
| volume | bigint | Predicted pool volume for next epoch, in USD with 6 decimal places (e.g., `5000000000n` = $5000.00) |

**Returns unsigned Transaction — must be signed and submitted separately.**

**`weight` values are abstract ratio units — NOT percentages required to sum to 100. The SDK normalizes any values to 100%. Pass values that make the intended split clear (e.g., `70n` and `30n` for a 70%/30% allocation).**

**`batchVoteTransaction` lives on the `Lock` namespace — `fullSailSDK.Lock.batchVoteTransaction()`. There is no separate Governance namespace.**

**Votes must be cast during the voting window (01:00 UTC Thursday to 23:00 UTC Wednesday). Votes submitted outside the window may fail or be ignored.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-09)
// Returns unsigned Transaction — must be signed and submitted separately
// weight: abstract units — any values normalize to 100%
//   [70n, 30n] = 70%/30% split
//   [1n, 1n]   = 50%/50% split (equal weight)
//   [20n, 80n] = 20%/80% split
// volume: predicted pool volume for next epoch, in USD with 6 decimal places (1000000n = $1.00)

const transaction = await fullSailSDK.Lock.batchVoteTransaction({
  lockIds: [lockId1, lockId2], // all locks contributing voting power
  votes: [
    { poolId: poolId1, weight: 70n, volume: 5000000000n }, // 70% weight, $5000 predicted volume
    { poolId: poolId2, weight: 30n, volume: 2000000000n }, // 30% weight, $2000 predicted volume
  ],
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

---

### Batch Vote Weight Mechanics

The `weight` field in each `votes` array member is an abstract ratio unit. The SDK sums all weight values across the votes array and uses each individual value as a fraction of that sum. This means any combination of values produces a valid allocation — there is no requirement to sum to exactly 100.

| weights array | Total sum | Pool A share | Pool B share |
|---------------|-----------|--------------|--------------|
| [70n, 30n] | 100 | 70% | 30% |
| [7n, 3n] | 10 | 70% | 30% |
| [1n, 1n] | 2 | 50% | 50% |
| [20n, 80n] | 100 | 20% | 80% |
| [1n, 3n] | 4 | 25% | 75% |

The first two rows produce identical allocations despite different values. The SDK normalizes any values to 100%.

**Do NOT validate that weights sum to exactly 100 before calling `batchVoteTransaction`. The SDK accepts any values and normalizes them. A sum-to-100 check will incorrectly reject valid inputs like `[1n, 1n]` (50%/50%) or `[7n, 3n]` (70%/30%).**

**Use weight values that make the intended split visually unambiguous. Prefer `[70n, 30n]` over `[7n, 3n]` for a 70%/30% split — both are correct, but the former is clearer when reading the code.**

---

### Vote Reset Behavior

When `Lock.mergeTransaction()` or `Lock.splitTransaction()` executes, the governance votes associated with the affected lock(s) are reset. Other locks owned by the same wallet are not affected — their vote allocations remain active for the current epoch.

| Operation | Locks affected | Votes reset | Unaffected |
|-----------|---------------|-------------|------------|
| mergeTransaction | fromLockId and toLockId | Votes from both affected locks | All other wallet locks |
| splitTransaction | lockId (original) | Votes from the split lock | New lock (starts with no votes), all other wallet locks |

**After `mergeTransaction`, recast votes from the surviving lock (`toLockId`) to resume governance participation for the epoch.**

**After `splitTransaction`, recast votes on both the original lock and the newly created lock to resume governance participation. The new lock starts with no votes.**

**The `isFromLockVoted`, `isToLockVoted` (merge) and `isVoted` (split) parameters signal the SDK to handle vote cleanup for the affected lock. Always pass these accurately — check whether each lock has active governance votes before calling.**

**Vote reset is not global — only the affected lock(s)' votes are cleared. An agent managing multiple locks does not need to recast votes for locks that were not part of the merge or split.**

See `### Lock.mergeTransaction()` and `### Lock.splitTransaction()` in ## Locks and veSAIL for the BLOCKING WARNINGs that appear before those method signatures.

---

### Reading Epoch State

No SDK method is documented for querying current epoch state or confirming whether the voting window is currently open. The only epoch-related SDK method is `fullSailSDK.Coin.getCurrentEpochOSail()`, which returns the current epoch's oSAIL coin type address — not epoch metadata, not epoch number, not window open/close timestamps.

To determine voting window status, use protocol timing rules:

```
Epoch boundary:      00:00 UTC every Thursday
Voting window open:  01:00 UTC Thursday
Voting window close: 23:00 UTC the following Wednesday
```

```typescript
// Source: https://docs.fullsail.finance/tutorials/prediction-voting.md (fetched 2026-03-09)
// No SDK method exists for epoch state — use protocol timing rules
// No getCurrentEpoch() method is documented in the SDK — do not call it

const now = new Date()
const dayOfWeek = now.getUTCDay()         // 0=Sun, 1=Mon, ..., 4=Thu, 3=Wed
const hourUTC = now.getUTCHours()
const minuteUTC = now.getUTCMinutes()
const totalMinutesUTC = hourUTC * 60 + minuteUTC
// Voting is OPEN if: day is Thu (4) and time >= 01:00, OR day is Fri/Sat/Sun/Mon/Tue/Wed and time < 23:00
// This is protocol-derived timing — no SDK call available to confirm current window status.
```

**Do NOT call `fullSailSDK.Lock.getCurrentEpoch()`, `fullSailSDK.Gauge.getEpoch()`, or any similar method — no such method is documented in the SDK. Attempting to call it will throw a runtime error.**

**`fullSailSDK.Coin.getCurrentEpochOSail()` returns the oSAIL coin type for the current epoch — use this for oSAIL lock creation and claim operations, not for determining epoch timing.**

---

## Vaults

This section covers Full Sail's automated vault system — delegated liquidity management that deposits assets into a vault-managed position, auto-compounds fees and rewards, and adjusts ranges without manual intervention.

### Vault Purpose

Full Sail vaults provide automated yield strategies on top of concentrated liquidity positions. A vault accepts deposit of one or both pool tokens, issues vault shares to the depositor, and manages the underlying position autonomously — rebalancing tick ranges, compounding accrued fees and rewards back into the position, and optimizing capital efficiency without requiring the depositor to monitor or adjust the position manually.

| Property | Value |
|----------|-------|
| What vaults manage | Full Sail concentrated liquidity positions |
| Deposit output | Vault shares (redeemable for underlying position value) |
| Compounding | Automatic — fees and rewards reinvested each epoch |
| Range management | Automatic — vault adjusts ticks based on price movement |
| Vault eligibility | Per-pool — only vault-enabled pools support this; check for compass icon in Full Sail app |

Vaults are not available for every pool. A pool must have vault support enabled (marked with a compass icon in the Full Sail app UI). Agents must confirm vault eligibility for a pool before attempting vault deposit — there is no SDK method to check vault eligibility at runtime; use the pool list from the Full Sail app or a known vault-enabled pool ID.

**Vault shares represent a proportional claim on the underlying vault position — their value changes as the vault compounds and rebalances. Do not treat share amounts as stable token balances.**

**Vault-enabled pools are a subset of all Full Sail pools. Verify the pool supports vaults before calling any vault transaction.**

---

### Chain Pool Requirement

All vault `*Transaction` calls require a Chain Pool — a real-time pool object fetched via `Pool.getByIdFromChain(poolId)`. Do not use a Backend Pool (`Pool.getById()`). This is the universal rule established in ## Protocol Fundamentals and applies equally to vault operations.

```typescript
// WRONG — stale state; use getByIdFromChain for vault transactions
const backendPool = await fullSailSDK.Pool.getById(poolId)
// await fullSailSDK.Vault.depositTransaction({ poolId: backendPool.id, ... }) // DO NOT USE
```

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-10)
// CORRECT — real-time tick state required for vault transactions
const chainPool = await fullSailSDK.Pool.getByIdFromChain(poolId)
// proceed to vault deposit or withdrawal using chainPool.id
```

**Call `Pool.getByIdFromChain(poolId)` before every vault `*Transaction` call. Backend Pool (`getById`) is for display calculations only — never pass it to vault transactions.**

**This is the same Chain Pool rule that applies to all Full Sail `*Transaction` calls. Vaults are not an exception.**

### Vault Deposit

Deposits tokens into a vault-managed position and returns vault shares.

> **PRECONDITION:** The pool must have vault support enabled (compass icon). Fetch a Chain Pool before calling: `const chainPool = await fullSailSDK.Pool.getByIdFromChain(poolId)`.

> **NOTE:** Vault deposit SDK method signatures are not published in the current public SDK documentation (`@fullsailfinance/sdk` as of 2026-03-10 lists no `Vault` namespace). The method and parameters below follow the project naming convention (`*Transaction` returns unsigned) but are INFERRED from UI tutorial behavior. **Verify exact method name and parameters at `https://docs.fullsail.finance/developer/SDK.md` before use.**

| Parameter | Type | Description |
|-----------|------|-------------|
| poolId | string | Vault-enabled pool ID — use `chainPool.id` from `Pool.getByIdFromChain()` |
| amountA | bigint | Token A deposit amount in base units |
| amountB | bigint | Token B deposit amount in base units |

Returns unsigned Transaction — must be signed and submitted separately.

```typescript
// UNVERIFIED — Vault SDK method signatures are NOT confirmed in public docs as of 2026-03-10
// Verify exact parameters at https://docs.fullsail.finance/developer/SDK.md before use
// Pattern follows project convention: *Transaction returns unsigned Transaction

// VAULT-04: Chain Pool required for vault transactions
const chainPool = await fullSailSDK.Pool.getByIdFromChain(poolId)

const transaction = await fullSailSDK.Vault.depositTransaction({
  poolId: chainPool.id,
  amountA: 1000000n,  // token A amount in base units
  amountB: 1000000n,  // token B amount in base units
})

const result = await wallet.signAndExecuteTransaction({ transaction })
// Returns vault shares — store for later withdrawal
```

**`// UNVERIFIED` — The `Vault` namespace is not documented in the public `@fullsailfinance/sdk` as of 2026-03-10. Verify `fullSailSDK.Vault.depositTransaction` exists and confirm parameter names before calling in production.**

**The `poolId` parameter must be `chainPool.id` from `Pool.getByIdFromChain()` — not from `Pool.getById()`. Backend Pool causes positioning errors.**

---

### Vault Withdrawal

Redeems vault shares for the underlying position value at current vault NAV.

> **NOTE:** Vault withdrawal SDK method signatures are not published in the current public SDK documentation (`@fullsailfinance/sdk` as of 2026-03-10 lists no `Vault` namespace). The method and parameters below follow the project naming convention but are INFERRED. **Verify exact method name and parameters at `https://docs.fullsail.finance/developer/SDK.md` before use.**

| Parameter | Type | Description |
|-----------|------|-------------|
| poolId | string | Vault-enabled pool ID — use `chainPool.id` from `Pool.getByIdFromChain()` |
| shares | bigint | Vault share amount to redeem (received from prior deposit) |

Returns unsigned Transaction — must be signed and submitted separately.

```typescript
// UNVERIFIED — Vault SDK method signatures are NOT confirmed in public docs as of 2026-03-10
// Verify exact parameters at https://docs.fullsail.finance/developer/SDK.md before use

// VAULT-04: Chain Pool required for vault transactions
const chainPool = await fullSailSDK.Pool.getByIdFromChain(poolId)

const transaction = await fullSailSDK.Vault.withdrawTransaction({
  poolId: chainPool.id,
  shares: shareAmount,  // vault share amount to redeem
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

**`// UNVERIFIED` — The `Vault` namespace is not documented in the public `@fullsailfinance/sdk` as of 2026-03-10. Verify `fullSailSDK.Vault.withdrawTransaction` exists and confirm parameter names before calling in production.**

**Pass the `shares` value received from the deposit receipt — do not derive share amounts independently. Share amounts are minted by the vault at deposit time.**

---

## Prediction Voting

This section covers Full Sail's prediction voting system, where veSAIL holders submit volume predictions for liquidity pools each epoch alongside their governance weight allocation. Prediction voting uses `Lock.batchVoteTransaction()` — the same method documented in `## Governance Voting` — the `volume` field in each vote entry IS the prediction; there is no separate prediction namespace or prediction-creation method.

### Prediction Voting Mechanism

Prediction voting allows veSAIL holders to forecast trading volume for each liquidity pool over the next epoch. These forecasts are submitted as part of `batchVoteTransaction` — the same call that allocates governance weight. A single transaction simultaneously casts a governance vote (directing emissions via `weight`) and a prediction vote (forecasting volume via `volume`). The two are inseparable: there is no governance vote without a volume prediction, and no prediction without a governance vote.

| Field | Purpose | Effect |
|-------|---------|--------|
| `weight` | Governance dimension | Directs veSAIL voting power to pools; determines SAIL emission allocation |
| `volume` | Prediction dimension | Forecasts pool trading volume for next epoch; scored for accuracy at epoch close |

The protocol aggregates all submitted volume predictions from veSAIL holders. At epoch close, actual pool trading volumes are compared against each holder's prediction. Holders with more accurate predictions receive a larger share of the 20% active-voter fee pool. Holders with less accurate predictions receive a smaller share. A `volume` of `0n` submits a null prediction and scores near-zero accuracy against any positive actual volume.

- **Do NOT look for a `Prediction` namespace, `createPrediction()`, or `submitPrediction()` method in `@fullsailfinance/sdk`. No such method exists. Prediction voting is `Lock.batchVoteTransaction()` with `volume` populated.**
- **`volume: 0n` is a valid call (will not revert) but submits a null prediction. It will score near-zero accuracy against any positive actual trading volume, earning minimal accuracy reward. Always pass your best volume estimate.**
- **veSAIL holders who do NOT submit `batchVoteTransaction` during the voting window receive ZERO fee share for that epoch — neither the 80% passive portion nor the 20% accuracy portion. Missing an epoch forfeits all fee share for that epoch.**

---

### Creating and Submitting a Prediction Vote

Prediction votes and governance votes are submitted in a single `Lock.batchVoteTransaction()` call. The method is fully documented in `## Governance Voting` — this subsection focuses on the `volume` field as the prediction component.

| Field | Type | Unit | Example |
|-------|------|------|---------|
| `volume` | bigint | USD with 6 decimal places | `5000000000n` = $5,000.00 |

| USD Amount | bigint Value |
|------------|-------------|
| $1.00 | `1000000n` |
| $100.00 | `100000000n` |
| $1,000.00 | `1000000000n` |
| $5,000.00 | `5000000000n` |
| $100,000.00 | `100000000000n` |

Set `volume` to your best estimate of the pool's actual trading volume over the next epoch. The prediction accuracy determines your share of the 20% active-voter fee pool at epoch close. Prediction accuracy is scored against the actual epoch trading volume — the closer your prediction, the higher your share.

```typescript
// Source: https://docs.fullsail.finance/developer/SDK.md (fetched 2026-03-10)
// Prediction voting and governance voting are the SAME operation.
// weight: relative units allocating veSAIL power (see ## Governance Voting for mechanics)
// volume: your volume prediction for this pool in the next epoch (USD with 6 decimal places)
//   1000000n    = $1.00
//   1000000000n = $1,000.00
//   5000000000n = $5,000.00
// Rewards auto-distributed at epoch end based on prediction accuracy — no claim needed

const transaction = await fullSailSDK.Lock.batchVoteTransaction({
  lockIds: [lockId],
  votes: [
    {
      poolId: poolId1,
      weight: 70n,
      volume: 5000000000n, // predict $5,000 volume for pool 1 next epoch
    },
    {
      poolId: poolId2,
      weight: 30n,
      volume: 2000000000n, // predict $2,000 volume for pool 2 next epoch
    },
  ],
})

const result = await wallet.signAndExecuteTransaction({ transaction })
// Prediction rewards auto-distributed at epoch close — no separate claim transaction required
```

- **`batchVoteTransaction` lives on the `Lock` namespace — `fullSailSDK.Lock.batchVoteTransaction()`. See `## Governance Voting → Lock.batchVoteTransaction()` for the complete parameter reference including the `lockIds` array and full `votes` member type.**
- **`volume` is NOT optional in a prediction voting context. Omitting it or passing `0n` submits a null prediction that earns no accuracy reward. Pass your best estimate of USD trading volume with 6 decimal places.**
- **Votes must be cast during the voting window (01:00 UTC Thursday to 23:00 UTC Wednesday). See `## Governance Voting → Epoch Cycle` for the complete timing reference.**

---

### Prediction Resolution and Rewards

At the close of each epoch, the protocol automatically distributes governance fee rewards. No transaction or SDK call is required — rewards flow directly to participating wallets. The distribution uses the pool's collected trading fees for the epoch, split as follows:

| Allocation | Share | Recipients |
|------------|-------|------------|
| Insurance fund | 5% of total fees | Protocol insurance reserve |
| Passive veSAIL distribution | 80% of remaining 95% | All veSAIL holders (proportional to veSAIL balance) — requires voting participation |
| Active predictor accuracy rewards | 20% of remaining 95% | veSAIL holders who submitted batchVoteTransaction with volume, weighted by prediction accuracy |

Both the 80% passive share and the 20% accuracy share require voting participation during the epoch — veSAIL holders who did not submit `batchVoteTransaction` receive neither share.

The protocol computes reward distribution on-chain using the following formula:

```
payout_i = (Σ vj) × [ vi × f(|T - Gi|) ] / [ Σ vk × f(|T - Gk|) ]

Where:
  T    = actual pool trading volume for the epoch
  Gi   = voter i's predicted volume (the volume field submitted via batchVoteTransaction)
  vi   = voting power allocated by voter i (proportional to veSAIL balance)
  f(x) = accuracy scoring function — f decreases as |T - G| increases (closer prediction = higher score)
```

Larger voting power (`vi`) amplifies reward proportionally. More accurate predictions (smaller `|T - Gi|`) increase the reward share fraction. A prediction of `0n` sets `Gi = 0`, so `|T - 0| = T` — scored against the full actual volume, yielding near-zero reward when actual volume is positive.

- **Do NOT call `fullSailSDK.Lock.claimPredictionReward()`, `fullSailSDK.Voter.claimFees()`, or any similar method after epoch close. No such method is documented in the SDK. Attempting to call it will throw a runtime error.**
- **Prediction rewards are distributed automatically at epoch boundary — no `*Transaction` claim call is needed. The 20% active-voter fee share flows to participating wallets by the protocol without agent action.**
- **If prediction rewards are not appearing in your wallet after epoch close, verify protocol behavior at the Full Sail app dashboard (`https://app.fullsail.finance`). No SDK diagnostic method is confirmed as of 2026-03-10.**

---

### Epoch Timing

Prediction votes are submitted within the governance voting window — the same epoch cycle documented in `## Governance Voting → Epoch Cycle`.

| Event | Time |
|-------|------|
| Epoch boundary (start/end) | 00:00 UTC every Thursday |
| Voting window opens | 01:00 UTC Thursday |
| Voting window closes | 23:00 UTC the following Wednesday |
| Predictions lock | At voting window close (23:00 UTC Wednesday) |
| Rewards distributed | At epoch boundary (00:00 UTC following Thursday) |

Predictions submitted after the voting window closes are not accepted for that epoch. To update a prediction mid-epoch, call `batchVoteTransaction` again during the window — the new call replaces the prior vote and prediction allocation.

- **Predictions cannot be submitted or updated after 23:00 UTC Wednesday. Any `batchVoteTransaction` call outside the voting window may fail or be ignored by the protocol.**

---

## Pitfalls

### ERR-01: Treating `*Transaction()` Return as Submitted

**Failure mode:** Agent calls any `*Transaction()` method (e.g., `Swap.swapTransaction()`, `Position.openTransaction()`) and assumes the transaction has been submitted to the network.

**Correct behavior:** All `*Transaction()` methods return an unsigned `Transaction` object. The agent must sign and execute it separately via the wallet client.

**Rule:** **Every `*Transaction()` call returns an unsigned `Transaction` — always sign and execute it before assuming any state change occurred.**

```typescript
// WRONG — tx is NOT submitted; it is only a constructed transaction object
const tx = await fullSailSDK.Swap.swapTransaction({ ... });
// State has NOT changed. Nothing was sent to the network.
```

```typescript
// CORRECT — sign and execute via the wallet client after constructing
const tx = await fullSailSDK.Swap.swapTransaction({ ... });
const result = await suiClient.signAndExecuteTransaction({
  transaction: tx,
  signer: keypair,
});
```

**See:** [§Protocol Fundamentals > Transaction Lifecycle Model](#transaction-lifecycle-model)

---

### ERR-02: Using oSAIL Without Checking Expiry

**Failure mode:** Agent uses an oSAIL coin object in a lock, claim, or redemption operation without verifying it is within the current epoch's 5-week expiry window.

**Correct behavior:** Before any oSAIL operation, retrieve the current epoch oSAIL via `Coin.getCurrentEpochOSail()` and check the expiry timestamp. Abort if expired.

**Rule:** **Check oSAIL expiry before every oSAIL operation — expired oSAIL cannot be used.**

```typescript
// WRONG — using a static or cached oSAIL coin type without expiry check
const lock = await fullSailSDK.Lock.createLockFromOSailTransaction({
  oSailCoinType: staticOSailType, // may be expired — no check performed
  // ...
});
```

```typescript
// CORRECT — fetch current epoch oSAIL, check expiry, then use its coinType
const currentOSail = await fullSailSDK.Coin.getCurrentEpochOSail();
if (Date.now() > currentOSail.expiry) {
  throw new Error("oSAIL is expired for this epoch — cannot proceed");
}
const lock = await fullSailSDK.Lock.createLockFromOSailTransaction({
  oSailCoinType: currentOSail.coinType, // confirmed fresh and unexpired
  // ...
});
```

**See:** [§Protocol Fundamentals > oSAIL Expiry Rule](#osail-expiry-rule), [§Rewards and oSAIL > oSAIL Expiry Check](#osail-expiry-check)

---

### ERR-03: Assuming All Rewards Auto-Claim

**Failure mode:** Agent assumes all reward types (fees, pool rewards, oSAIL) are claimed automatically during liquidity operations.

**Correct behavior:** Only oSAIL is auto-claimed during staking/unstaking operations. Trading fees and pool token rewards require explicit `claim*` calls.

**Rule:** **Only oSAIL auto-claims during liquidity operations — fees and pool rewards require explicit `claim*` calls.**

Consult the full reward type mapping before any claim operation. Each reward type has a distinct claim method and eligibility path.

**See:** [§Rewards and oSAIL > Reward Claim Matrix](#reward-claim-matrix)

---

### ERR-04: Using Wrong Claim Method for Stake Status

**Failure mode:** Agent calls staked-position claim methods on an unstaked position (or vice versa), causing transaction failure or claiming the wrong reward type.

**Correct behavior:** Determine stake status first using `### Staked vs Unstaked Detection`, then follow the appropriate claim path in the decision tree.

**Rule:** **Always determine position stake status before selecting a claim method — staked and unstaked positions use entirely different claim paths.**

Check whether the position has a `stake_info` field populated before selecting any claim method. The decision tree maps each reward type to the correct method for each stake status.

**See:** [§Liquidity Positions > Staked vs Unstaked Detection](#staked-vs-unstaked-detection), [§Rewards and oSAIL > Staked vs Unstaked Claim Path](#staked-vs-unstaked-claim-path)

---

### ERR-05: Merge or Split Resets Voting Weight

**Failure mode:** Agent merges or splits locks after voting, expecting existing votes on those locks to persist.

**Correct behavior:** `Lock.mergeTransaction()` and `Lock.splitTransaction()` both irreversibly reset ALL voting weight on the affected locks. After merge or split, the agent must re-cast all governance votes.

**Rule:** **Merge and split are irreversible voting-weight resets — re-cast all votes on affected locks after either operation.**

> **BLOCKING WARNING:** `mergeTransaction()` and `splitTransaction()` reset all voting weight on affected locks. This cannot be undone. Re-vote after any merge or split.

**See:** [§Locks and veSAIL > Lock.mergeTransaction()](#lockmergetransaction), [§Locks and veSAIL > Lock.splitTransaction()](#locksplittransaction), [§Governance Voting > Vote Reset Behavior](#vote-reset-behavior)

---

### ERR-06: Setting Sender Before Wallet Connect

**Failure mode:** Agent calls `setSenderAddress()` during SDK initialization, before wallet connection is confirmed.

**Correct behavior:** `setSenderAddress()` must be called only after wallet connection is confirmed — not during SDK init.

**Rule:** **Call `setSenderAddress()` after wallet connect, never during SDK initialization.**

```typescript
// WRONG — setSenderAddress called at init time, before wallet is connected
const fullSailSDK = new FullSailSDK({ /* config */ });
fullSailSDK.setSenderAddress(walletAddress); // wallet not yet confirmed connected
```

```typescript
// CORRECT — setSenderAddress called in wallet connect callback, after connection confirmed
const fullSailSDK = new FullSailSDK({ /* config */ });

walletClient.on("connect", (wallet) => {
  fullSailSDK.setSenderAddress(wallet.address); // called after confirmed connect
});
```

**See:** [§Protocol Fundamentals > Two-Phase Sender Setup](#two-phase-sender-setup)

---

### ERR-07: Using Backend Pool Object for Transactions

**Failure mode:** Agent fetches a pool via `Pool.getById()` (Backend Pool) and passes it into transaction methods, causing incorrect calculation or failure.

**Correct behavior:** Transaction methods require a Chain Pool fetched via `Pool.getByIdFromChain()`. Backend Pool is for display and metadata only.

**Rule:** **Use `Pool.getByIdFromChain()` for all transaction calls — `Pool.getById()` returns a Backend Pool valid only for display.**

```typescript
// WRONG — backend pool used for transaction; will produce incorrect results
const pool = await fullSailSDK.Pool.getById({ poolId });
// passing pool to transaction methods — incorrect
```

```typescript
// CORRECT — chain pool fetched for transaction use
const chainPool = await fullSailSDK.Pool.getByIdFromChain({ poolId });
// chainPool is the correct object for all transaction methods
```

**See:** [§Protocol Fundamentals > Pool Data Sources: Chain Pool vs Backend Pool](#pool-data-sources-chain-pool-vs-backend-pool)

---

### ERR-08: Calling `removeLiquidityTransaction()` Without Staked Params

**Failure mode:** Agent calls `Position.removeLiquidityTransaction()` on a staked position without passing the staked-position parameters, causing transaction failure.

**Correct behavior:** There is no separate `unstakeTransaction()` in the SDK. To remove liquidity from a staked position, pass `positionStakeId`, `gaugeId`, and `oSailCoinType` directly to `removeLiquidityTransaction()`. See the full parameter set in the domain section.

**Rule:** **There is no `unstakeTransaction()` — pass staked params directly to `removeLiquidityTransaction()` when the position is staked.**

```typescript
// WRONG — unstakeTransaction does not exist in the SDK
await fullSailSDK.Position.unstakeTransaction({ positionId }); // ERROR: method does not exist
```

```typescript
// CORRECT — pass staked params directly to removeLiquidityTransaction
await fullSailSDK.Position.removeLiquidityTransaction({
  // ... standard remove params ...
  positionStakeId: position.stake_info.id, // required when position is staked
  gaugeId,
  oSailCoinType,
});
```

**See:** [§Liquidity Positions > Position.removeLiquidityTransaction()](#positionremoveliquiditytransaction)

---

### ERR-09: Routing oSAIL Through DEX Swap

**Failure mode:** Agent includes oSAIL as an input or output token in `Swap.getSwapRoute()` or `Swap.swapRouterTransaction()`, resulting in route failure.

**Correct behavior:** oSAIL is not a standard DEX-tradeable token and has no swap routes. To convert oSAIL, use `Lock.createLockFromOSailTransaction()` (lock path) or the redemption options documented in `### oSAIL Redemption Options`.

**Rule:** **oSAIL cannot be routed through the DEX — use the lock or redemption path instead.**

```typescript
// WRONG — oSAIL has no DEX route; this call will fail
const route = await fullSailSDK.Swap.getSwapRoute({
  coinInType: oSailCoinType, // fails — no DEX route for oSAIL
  // ...
});
```

```typescript
// CORRECT — use the lock path to convert oSAIL
await fullSailSDK.Lock.createLockFromOSailTransaction({
  oSailCoinType,
  // ...
});
```

**See:** [§Swaps > oSAIL Swap Restriction](#osail-swap-restriction), [§Rewards and oSAIL > oSAIL Redemption Options](#osail-redemption-options)

---

### ERR-10: Reading State Immediately After Transaction Submit

**Failure mode:** Agent reads on-chain state (position balance, lock amount, reward totals) in the same request immediately after submitting a transaction and acts on stale data.

**Correct behavior:** On-chain state on Sui takes time to finalize after transaction submission. Do not read state in the same logical step as submit. Wait for transaction finality confirmation before querying state.

**Rule:** **Do not read on-chain state immediately after submitting a transaction — wait for finality before querying updated state.**

After calling `signAndExecuteTransaction()`, the returned result confirms submission but on-chain objects may not yet reflect the update. Query state only after the transaction has reached finality on the Sui network.

---

### Pre-Flight Checklist

Before executing any Full Sail transaction, verify:

- [ ] SDK sender is set — called after wallet connect, not at SDK init ([§Two-Phase Sender Setup](#two-phase-sender-setup))
- [ ] Using Chain Pool from `getByIdFromChain()` — not Backend Pool from `getById()` ([§Pool Data Sources](#pool-data-sources-chain-pool-vs-backend-pool))
- [ ] oSAIL expiry checked — fetched via `Coin.getCurrentEpochOSail()` and not past expiry ([§oSAIL Expiry Rule](#osail-expiry-rule))
- [ ] Position stake status determined before selecting claim method ([§Staked vs Unstaked Claim Path](#staked-vs-unstaked-claim-path))
- [ ] Transaction signed and executed via wallet client — not just constructed ([§Transaction Lifecycle Model](#transaction-lifecycle-model))

---

## Source of Truth

### Documentation

| Resource | URL |
|----------|-----|
| Full Sail Docs | https://docs.fullsail.finance |
| SDK Reference | https://docs.fullsail.finance/developer/SDK.md |
| Contract Addresses | https://docs.fullsail.finance/developer/contracts.md |
| Token Contracts | https://docs.fullsail.finance/developer/token-contracts.md |
| Full Sail App | https://app.fullsail.finance/ |
| SDK npm | https://www.npmjs.com/package/@fullsailfinance/sdk |

### Contract Addresses (Mainnet)

Fetched from https://docs.fullsail.finance/developer/contracts.md (2026-03-10). Verify against live docs before production use.

**Packages:**

| Package | Address |
|---------|---------|
| Integrate Package | `0x3d5f22b95e48ba187397e1f573c6931fac5bbb0023e9a753da36da6c93c8f151` |
| clmm_pool | `0xf7ca99f9fd82da76083a52ab56d88aff15d039b76499b85db8b8bc4d4804584a` |
| voting_escrow | `0xfc410c145e4a9ba8f4aa3cb266bf3e467c35ea39dc90788e9a34f85338b734b7` |
| governance | `0x1cde2f0d4a50700960a8062f4ed7b19258f2a8c5eb4dc798fbda5e8b8d8c0658` |

**Global Objects:**

| Object | Address |
|--------|---------|
| GlobalConfig | `0xe93baa80cb570b3a494cbf0621b2ba96bc993926d34dc92508c9446f9a05d615` |
| RewarderGlobalVault | `0xfb971d3a2fb98bde74e1c30ba15a3d8bef60a02789e59ae0b91660aeed3e64e1` |
| Pools | `0x0efb954710df6648d090bdfa4a5e274843212d6eb3efe157ee465300086e3650` |
| VotingEscrow | `0xe36c353bf09559253306fcec8ccdd6414ef01c20684f1d31f00ed25034718189` |
| Voter | `0x266ff531d300f00ed725e801ba2898d926cad17b9406ea2150e1085de255898f` |
| Minter | `0x58f1b1c1c3b996ffc6e100131cddd0c9999d10a2744db37b5d2422ae52db97f4` |

### Token Contracts (Mainnet)

Fetched from https://docs.fullsail.finance/developer/token-contracts.md (2026-03-10).

| Token | Coin Type |
|-------|-----------|
| SAIL | `0x1d4a2bdbc1602a0adaa98194942c220202dcc56bb0a205838dfaa63db0d5497e::SAIL::SAIL` |
| veSAIL | `0xe616397e503278d406e184d2258bcbe7a263d0192cc0848de2b54b518165f832::voting_escrow::Lock` |
| oSAIL | epoch-dynamic — retrieve at runtime via `Coin.getCurrentEpochOSail()`. Do not hardcode. |
