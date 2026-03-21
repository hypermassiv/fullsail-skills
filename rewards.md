> SDK version: v9.0.0 | Audit date: 2026-03-21

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

**`rewardCoinTypes` must include all reward coin type addresses for this pool. Derive from `pool.rewards?.map((r) => r.token.address)` — `Pool.getById()` returns a `PoolByAddressResponse` wrapper; destructure `{ pool }` from the response.**

**Precondition: position must be unstaked.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately

const { pool } = await fullSailSDK.Pool.getById(poolId)  // PoolByAddressResponse — destructure pool
// rewardCoinTypes: use pool.rewards — confirmed field name from PoolByAddressResponse
const rewardCoinTypes = pool.rewards?.map((r) => r.token.address) ?? []

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

const { pool } = await fullSailSDK.Pool.getById(poolId)  // PoolByAddressResponse — destructure pool
// rewardCoinTypes: use pool.rewards — confirmed field name from PoolByAddressResponse
const rewardCoinTypes = pool.rewards?.map((r) => r.token.address) ?? []

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

**Returns `Promise<[Transaction, oSailCoin]>` — a tuple.** The `oSailCoin` (`TransactionObjectArgument`) is unconsumed and MUST be handled:

> **oSailCoin is not auto-transferred.** The second element of the return tuple is a `TransactionObjectArgument` that must be consumed before submitting. Either:
> - Set `sendToSender: true` in params (SDK auto-transfers oSAIL to sender — simplest option)
> - Or destructure `const [tx, oSailCoin] = await claimOSailTransaction(...)` and use `oSailCoin` in a downstream transaction command

| Parameter | Type | Description |
|-----------|------|-------------|
| `coinTypeA` | `string` | Pool's coinTypeA — from Backend Pool `pool.token_a.address` |
| `coinTypeB` | `string` | Pool's coinTypeB — from Backend Pool `pool.token_b.address` |
| `poolId` | `string` | Pool object ID |
| `positionStakeId` | `string` | Stake object ID — `position.stake_info.id` |
| `oSailCoinType` | `string` | Current epoch oSAIL coin type — `currentEpochOSail.address` from `fullSailSDK.Coin.getCurrentEpochOSail()` |
| `gaugeId` | `string` | From Backend Pool — `pool.gauge_id` |
| `sendToSender?` | `boolean` | When `true`, the SDK automatically transfers the oSailCoin to the sender, avoiding a dangling object. Default: `false`. |

**Precondition: position must be staked (`position.stake_info` exists). `positionStakeId` is `position.stake_info.id` — never hardcode.**

**`oSailCoinType` is `currentEpochOSail.address` from `fullSailSDK.Coin.getCurrentEpochOSail()`. Call this fresh — do not cache.**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns Promise<[Transaction, oSailCoin]> — tuple; oSailCoin must be consumed

const pool = await fullSailSDK.Pool.getById(poolId)
const position = await fullSailSDK.Position.getById(positionId)
// Call getCurrentEpochOSail() fresh — never use a cached value
const currentEpochOSail = await fullSailSDK.Coin.getCurrentEpochOSail()

// Option A: sendToSender: true — SDK auto-transfers oSAIL to sender (simplest)
const [transaction, oSailCoin] = await fullSailSDK.Position.claimOSailTransaction({
  coinTypeA: pool.token_a.address,
  coinTypeB: pool.token_b.address,
  poolId,
  positionStakeId: position.stake_info.id, // never hardcode — always from position.stake_info.id
  oSailCoinType: currentEpochOSail.address, // fresh call required — not cached
  gaugeId: pool.gauge_id,
  sendToSender: true, // auto-transfers oSailCoin to sender — omit if consuming oSailCoin manually
})

// Option B: omit sendToSender and consume oSailCoin manually in a downstream step
// const [transaction, oSailCoin] = await fullSailSDK.Position.claimOSailTransaction({...})
// oSailCoin must be used in a downstream transaction command — DO NOT submit with a dangling oSailCoin

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

**`rewardCoinTypes` must be exhaustive — derive from `pool.rewards?.map((r) => r.token.address)` after destructuring `{ pool }` from `Pool.getById()` (see claimUnstakedPoolRewardsTransaction).**

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately

const { pool } = await fullSailSDK.Pool.getById(poolId)  // PoolByAddressResponse — destructure pool
const position = await fullSailSDK.Position.getById(positionId)
// rewardCoinTypes: use pool.rewards — confirmed field name from PoolByAddressResponse
const rewardCoinTypes = pool.rewards?.map((r) => r.token.address) ?? []

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

Combined call — claims both oSAIL and pool rewards in a single transaction for staked positions. The claimed oSAIL is immediately exercised according to `rewardChoice`. Prefer this over two separate calls.

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
| `rewardChoice` | `'sail' \| 'usd' \| 'vesail'` | How to exercise the claimed oSAIL |
| `positionUpdatedAt` | `number` | Position update timestamp — `position.updated_at` |
| `oSailAmount` | `bigint` | Amount of oSAIL to exercise. Pass `0n` if no oSAIL is pending |
| `slippage` | `Percentage` | For oSAIL exercise swap calculations |
| `oSailExpired?` | `boolean` | Whether the oSAIL is expired |
| `newLockDurationDays?` | `number` | Lock duration in days — only when `rewardChoice: 'vesail'` |
| `newLockIsPermanent?` | `boolean` | Permanent lock flag — only when `rewardChoice: 'vesail'` |
| `permanentLockId?` | `string` | Existing permanent lock to increase — only when `rewardChoice: 'vesail'` |

**Precondition: position must be staked. This combined method replaces both `claimOSailTransaction` and `claimStakedPoolRewardsTransaction` — do not call all three.**

**`rewardChoice` controls oSAIL exercise:** `'sail'` swaps to SAIL, `'usd'` swaps to USDC, `'vesail'` locks as veSAIL. Pass `oSailAmount: 0n` if no oSAIL is pending to skip oSAIL exercise.

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Returns unsigned Transaction — must be signed and submitted separately

const { pool } = await fullSailSDK.Pool.getById(poolId)  // PoolByAddressResponse — destructure pool
const position = await fullSailSDK.Position.getById(positionId)
// Call getCurrentEpochOSail() fresh — never use a cached value
const currentEpochOSail = await fullSailSDK.Coin.getCurrentEpochOSail()
// rewardCoinTypes: use pool.rewards — confirmed field name from PoolByAddressResponse
const rewardCoinTypes = pool.rewards?.map((r) => r.token.address) ?? []

const transaction = await fullSailSDK.Position.claimOSailAndStakedPoolRewardsTransaction({
  coinTypeA: pool.token_a.address,
  coinTypeB: pool.token_b.address,
  poolId,
  positionStakeId: position.stake_info.id,
  gaugeId: pool.gauge_id,
  oSailCoinType: currentEpochOSail.address, // fresh call required — not cached
  rewardCoinTypes,
  rewardChoice: 'sail',              // 'sail' | 'usd' | 'vesail'
  positionUpdatedAt: position.updated_at,
  oSailAmount: 0n,                   // pass 0n if no oSAIL is pending
  slippage: 0.005,                   // 0.5% slippage tolerance
  // oSailExpired: false,            // optional — set if oSAIL is expired
  // newLockDurationDays: 365,       // optional — only when rewardChoice: 'vesail'
  // newLockIsPermanent: false,      // optional — only when rewardChoice: 'vesail'
  // permanentLockId: '0x...',       // optional — existing permanent lock to increase
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
// currentEpochOSail.expiration_timestamp — confirmed expiry field name
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

**oSAIL Conversion Rates**

The SAIL received when exercising oSAIL depends on the chosen lock duration:

| Lock Duration | SAIL per oSAIL | veSAIL per oSAIL |
|---------------|----------------|------------------|
| 6 months | 0.5625 | 0.0703 |
| 2 years | 0.75 | 0.375 |
| 4 years | 1.0 | 1.0 |

**Formula:** `SAIL = oSAIL × (0.5 + duration_ratio × 0.5)` where `duration_ratio = lock_duration / max_duration` (max = 4 years = 1461 days). Longer locks receive more SAIL and more veSAIL per oSAIL redeemed.

**Option B: DEPRECATED — SAIL+USDC exercise (`exercise_o_sail`) is deprecated**

> **DEPRECATED:** The paid exercise path (redeeming oSAIL for SAIL+USDC) has been deprecated in the protocol. Do not use this path. Use Option A (lock as veSAIL) or free exercise instead.

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

### oSAIL Type Validation (is_valid_o_sail_type)

`is_valid_o_sail_type` is a contract-level method for validating whether a historical oSAIL coin type is recognized by the protocol. Each epoch issues a unique oSAIL coin type — historical oSAIL coin types from prior epochs may need validation before being submitted to protocol operations.

**When to use:** When handling oSAIL tokens that may have been collected across multiple epochs (for example, from a wallet holding oSAIL from past emissions), call `is_valid_o_sail_type` to confirm the coin type is a recognized Full Sail oSAIL type before attempting any protocol operation with it.

**Note:** This is a contract-level function, not an SDK namespace method. Invocation requires direct contract interaction via the Sui RPC or a Sui PTB (programmable transaction block) targeting the Full Sail protocol package. Verify the exact call pattern from the Full Sail contract ABI or protocol documentation before use.

---

