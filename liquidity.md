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

**No rewards are auto-claimed during `stakeTransaction`. Trading fees require `claimFeeTransaction`; pool rewards require their respective claim methods. This differs from `addLiquidityTransaction`, which auto-claims oSAIL.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Use for positions created without gaugeId — stakes into gauge retroactively
// Returns unsigned Transaction — must be signed and submitted separately
// NOTE: No rewards auto-claimed in this transaction. Fees require claimFeeTransaction; pool rewards require explicit claim.

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

