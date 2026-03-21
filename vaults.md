## Vaults

This section covers Full Sail's automated vault system — delegated liquidity management that deposits assets into a vault-managed position, auto-compounds fees and rewards, and adjusts ranges without manual intervention.

### Vault Purpose

Full Sail vaults provide automated yield strategies on top of concentrated liquidity positions. A vault accepts deposit of one or both pool tokens and manages the underlying position autonomously — rebalancing tick ranges, compounding accrued fees and rewards back into the position, and optimizing capital efficiency without requiring the depositor to monitor or adjust the position manually.

| Property | Value |
|----------|-------|
| What vaults manage | Full Sail concentrated liquidity positions |
| SDK module | `fullSailSDK.PortEntry` (user-facing) + `fullSailSDK.Port` (oracle helpers) |
| Compounding | Automatic — fees and rewards reinvested each epoch |
| Range management | Automatic — vault adjusts ticks based on price movement |
| Vault eligibility | Per-pool — only vault-enabled pools support this; check for compass icon in Full Sail app |

Vaults are not available for every pool. A pool must have vault support enabled (marked with a compass icon in the Full Sail app UI). Agents must confirm vault eligibility for a pool before attempting vault deposit — there is no SDK method to check vault eligibility at runtime; use the pool list from the Full Sail app or a known vault-enabled pool ID.

**Vault-enabled pools are a subset of all Full Sail pools. Verify the pool supports vaults before calling any vault transaction.**

---

### SDK Terminology

The vault feature uses different naming in the SDK than in the UI:

| UI / Docs term | SDK term | Description |
|----------------|----------|-------------|
| Vault | Port (`Port` interface) | The vault-managed strategy for a pool |
| Vault entry / user position | PortEntry (`PortEntry` interface) | A user's position within a vault |
| Deposit | `PortEntry.createPortEntryTransaction()` | Opens a new vault position |
| Add liquidity | `PortEntry.addLiquidityTransaction()` | Increases an existing vault position |
| Withdraw | `PortEntry.removeLiquidityTransaction()` | Partially exits a vault position |
| Close | `PortEntry.closeTransaction()` | Fully exits and destroys a vault position |

---

### Getting Vault IDs

Every vault transaction requires three IDs: `poolId`, `portId`, and `gaugeId`. All three can be retrieved in a single backend API call.

```typescript
// Fetch pool + associated ports in one call
// sdk.api is the backend REST client exposed by FullSailSDK
const { pool, ports } = await sdk.api.pools.byAddressDetail(poolId)
// pool.id       → poolId  (same as the input poolId)
// pool.gauge_id → gaugeId (undefined if pool has no gauge — i.e. vault not supported)
// ports[0].id   → portId  (the vault / Port object ID)
```

**If `pool.gauge_id` is `undefined`, the pool does not support vaults. Do not proceed.**

To retrieve a user's existing PortEntry (needed for add liquidity, withdraw, close):

```typescript
// Returns Strategy { port_entry: PortEntry, port: Port }
const strategy = await fullSailSDK.PortEntry.getById(portEntryId)
// portEntryId is the Sui object ID of the user's PortEntry — found in the user's owned objects
```

---

### Getting Oracle Price IDs

Vault deposit and withdrawal params require oracle price IDs for each token. Fetch them from the Port module before building the transaction:

```typescript
const [pythPriceIdA, pythPriceIdB, aggregatorPriceIdA, aggregatorPriceIdB] = await Promise.all([
  fullSailSDK.Port.getPythPriceFeedId({ coinType: coinTypeA }),
  fullSailSDK.Port.getPythPriceFeedId({ coinType: coinTypeB }),
  fullSailSDK.Port.getAggregatorPriceId({ coinType: coinTypeA }),
  fullSailSDK.Port.getAggregatorPriceId({ coinType: coinTypeB }),
])
// Each returns string | null — pass null if the oracle doesn't exist for that token
```

---

### Vault Deposit

Opens a new vault position by depositing tokens. Uses `PortEntry.createPortEntryTransaction()`.

> **PRECONDITION:** The pool must have vault support enabled (compass icon). Confirm `pool.gauge_id` is defined before proceeding.

| Parameter | Type | Required | Source/Notes |
|-----------|------|----------|--------------|
| `portId` | `string` | Yes | From `ports[0].id` via `sdk.api.pools.byAddressDetail()` |
| `gaugeId` | `string` | Yes | From `pool.gauge_id` via `sdk.api.pools.byAddressDetail()` |
| `pythPriceIdA` | `string \| null \| undefined` | Yes (null ok) | From `Port.getPythPriceFeedId({ coinType: coinTypeA })` |
| `pythPriceIdB` | `string \| null \| undefined` | Yes (null ok) | From `Port.getPythPriceFeedId({ coinType: coinTypeB })` |
| `aggregatorPriceIdA` | `string \| null \| undefined` | Yes (null ok) | From `Port.getAggregatorPriceId({ coinType: coinTypeA })` |
| `aggregatorPriceIdB` | `string \| null \| undefined` | Yes (null ok) | From `Port.getAggregatorPriceId({ coinType: coinTypeB })` |
| `coinTypeA` | `string` | Yes | Pool token A type |
| `coinTypeB` | `string` | Yes | Pool token B type |
| `rewardCoinTypes` | `string[]` | Yes | Reward coin types for this port — from `ports[0].rewards.map(r => r.token.address)`. NOTE: field is `rewardCoinTypes` here, but `portRewardCoinTypes` in add/remove/close (see naming trap below) |
| `amountA` | `bigint` | Yes | Token A deposit amount in base units |
| `amountB` | `bigint` | Yes | Token B deposit amount in base units |
| `currentOSailCoinType` | `string` | Yes | Active oSAIL coin type from `Coin.getCurrentEpochOSail().address` |
| `poolId` | `string` | Yes | Pool object ID from `pool.id` (backend API) |
| `senderAddress` | `string` | Yes | Wallet address |

> **Naming trap:** `createPortEntryTransaction` uses `rewardCoinTypes`. The add/remove/close functions use `portRewardCoinTypes` instead. Do not copy field names between create and add/remove/close — see ERR-15.

Returns unsigned `Transaction` — must be signed and submitted separately.

```typescript
// 1. Fetch vault IDs
const { pool, ports } = await sdk.api.pools.byAddressDetail(poolId)
const portId = ports[0].id
const gaugeId = pool.gauge_id!  // assert non-null after eligibility check

// 2. Fetch oracle price IDs
const [pythPriceIdA, pythPriceIdB, aggregatorPriceIdA, aggregatorPriceIdB] = await Promise.all([
  fullSailSDK.Port.getPythPriceFeedId({ coinType: coinTypeA }),
  fullSailSDK.Port.getPythPriceFeedId({ coinType: coinTypeB }),
  fullSailSDK.Port.getAggregatorPriceId({ coinType: coinTypeA }),
  fullSailSDK.Port.getAggregatorPriceId({ coinType: coinTypeB }),
])

// 3. Build and submit deposit transaction
const transaction = await fullSailSDK.PortEntry.createPortEntryTransaction({
  coinTypeA,
  coinTypeB,
  amountA: 1_000_000n,
  amountB: 1_000_000n,
  poolId: pool.id,
  gaugeId,
  portId,
  pythPriceIdA,
  pythPriceIdB,
  aggregatorPriceIdA,
  aggregatorPriceIdB,
  rewardCoinTypes: ports[0].rewards.map(r => r.token.address),
  currentOSailCoinType,  // active oSAIL coin type from protocol config
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

**`poolId`, `gaugeId`, and `portId` must all come from the same backend API response — do not mix IDs from different sources.**

---

### Add Liquidity to Vault

Increases an existing vault position. Uses `PortEntry.addLiquidityTransaction()`.

> **PRECONDITION:** You need the user's `portEntryId` — the Sui object ID of their existing PortEntry — from their owned objects on-chain.

> **8 reward parameters required.** Unlike `createPortEntryTransaction`, the add/remove/close functions require reward-related parameters (`portRewardCoinTypes`, `poolRewardCoinTypes`, `oSailRewards`, `currentOSailCoinType`, `oSailDecimals`, `sailPrice`, `rewardChoice`, `slippage`) because these operations also harvest pending rewards. An agent omitting these will get transaction failure.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| `coinTypeA` | `SuiCoinType` | Yes | Pool token A type |
| `coinTypeB` | `SuiCoinType` | Yes | Pool token B type |
| `amountA` | `bigint` | Yes | Token A amount to add in base units |
| `amountB` | `bigint` | Yes | Token B amount to add in base units |
| `poolId` | `SuiObjectId` | Yes | From backend API |
| `gaugeId` | `SuiObjectId` | Yes | From backend API |
| `portId` | `SuiObjectId` | Yes | From backend API |
| `portEntryId` | `SuiObjectId` | Yes | User's existing PortEntry object ID |
| `pythPriceIdA` | `string \| null \| undefined` | Yes (null ok) | From `Port.getPythPriceFeedId({ coinType: coinTypeA })` |
| `pythPriceIdB` | `string \| null \| undefined` | Yes (null ok) | From `Port.getPythPriceFeedId({ coinType: coinTypeB })` |
| `aggregatorPriceIdA` | `string \| null \| undefined` | Yes (null ok) | From `Port.getAggregatorPriceId({ coinType: coinTypeA })` |
| `aggregatorPriceIdB` | `string \| null \| undefined` | Yes (null ok) | From `Port.getAggregatorPriceId({ coinType: coinTypeB })` |
| `portRewardCoinTypes` | `SuiCoinType[]` | Yes | Incentive reward types for the port. Pass `[]` if none. NOT `rewardCoinTypes`. |
| `poolRewardCoinTypes` | `SuiCoinType[]` | Yes | Pool trading reward types. Pass `[]` if none. |
| `oSailRewards` | `Array<{ coinType, amount, expired }>` | Yes | Pending oSAIL rewards. Obtain via `PortEntry.getOSailRewards()` then enrich with `expired` field (see pattern below). Pass `[]` if none. |
| `currentOSailCoinType` | `SuiCoinType` | Yes | Active oSAIL coin type from `Coin.getCurrentEpochOSail().address` |
| `oSailDecimals` | `number` | Yes | oSAIL token decimals (typically 6) |
| `sailPrice` | `number` | Yes | Current SAIL token price in USD — NO SDK method, must use external source |
| `rewardChoice` | `'sail' \| 'usd' \| 'vesail'` | Yes | How to exercise oSAIL rewards — `'vesail'` locks into veSAIL, `'sail'` swaps to SAIL, `'usd'` swaps to USDC |
| `slippage` | `Percentage` | Yes | Used for oSAIL exercise swap calculations |

> **`sailPrice` has no SDK method.** You must obtain the current SAIL token price from an external source (CoinGecko API, DeFi Llama, or on-chain price oracle). There is no `getSailPrice()` in the SDK.

> **`rewardChoice` values:** `'sail'` (swap oSAIL to SAIL), `'usd'` (swap oSAIL to USDC), `'vesail'` (lock oSAIL into veSAIL). These are lowercase strings, NOT enum constants.

**`oSailRewards.expired` field population pattern.** `PortEntry.getOSailRewards()` returns `Array<{ coinType, amount }>` — it does NOT include the `expired` field. However, `addLiquidityTransaction` (and removeLiquidity/close) requires `expired: boolean` in each `oSailRewards` entry. To populate `expired`, cross-reference each reward's `coinType` against `Coin.getOSailMap()`:

```typescript
// Step 1: Get raw oSAIL rewards (no expired field)
const rawRewards = await sdk.PortEntry.getOSailRewards({
  portId, portEntryId, coinTypeA, coinTypeB,
  currentOSailCoinType, gaugeId, poolId,
})
// Returns: Array<{ coinType: SuiCoinType, amount: bigint }>

// Step 2: Get oSAIL expiry map
const oSailMap = await sdk.Coin.getOSailMap()
// Returns map of coinType -> { expiration_timestamp, ... }

// Step 3: Enrich with expired field
const now = Date.now()
const oSailRewards = rawRewards.map(r => ({
  coinType: r.coinType,
  amount: r.amount,
  expired: oSailMap[r.coinType]
    ? now > Number(oSailMap[r.coinType].expiration_timestamp)
    : true, // treat unknown as expired for safety
}))
```

> **Low-confidence pattern.** The `expired` field population via `Coin.getOSailMap()` cross-reference is derived from SDK type analysis. Verify against testnet before production use.

---

### Vault Withdrawal

Exits a vault position partially. Use `removeLiquidityTransaction()` to withdraw a portion; use `closeTransaction()` to withdraw everything and destroy the PortEntry object.

> **PRECONDITION:** You need the user's `portEntryId` — the Sui object ID of their PortEntry — from their owned objects on-chain.

`removeLiquidityTransaction` requires all the reward params from `addLiquidityTransaction`, plus five additional position-state fields:

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| `coinTypeA` | `SuiCoinType` | Yes | Pool token A type |
| `coinTypeB` | `SuiCoinType` | Yes | Pool token B type |
| `poolId` | `SuiObjectId` | Yes | From backend API |
| `gaugeId` | `SuiObjectId` | Yes | From backend API |
| `portId` | `SuiObjectId` | Yes | From backend API |
| `portEntryId` | `SuiObjectId` | Yes | User's PortEntry object ID |
| `volume` | `bigint` | Yes | Share volume to withdraw (NOT `liquidityAmount`) |
| `amountA` | `bigint` | Yes | Current position token A amount (for slippage calc) |
| `amountB` | `bigint` | Yes | Current position token B amount (for slippage calc) |
| `tickLower` | `number` | Yes | Current position lower tick |
| `tickUpper` | `number` | Yes | Current position upper tick |
| `liquidity` | `bigint` | Yes | Current position liquidity |
| `currentSqrtPrice` | `bigint` | Yes | Current pool sqrt price |
| `slippage` | `Percentage` | Yes | Used for slippage calc and oSAIL exercise |
| `pythPriceIdA` | `string \| null \| undefined` | Yes (null ok) | From `Port.getPythPriceFeedId({ coinType: coinTypeA })` |
| `pythPriceIdB` | `string \| null \| undefined` | Yes (null ok) | From `Port.getPythPriceFeedId({ coinType: coinTypeB })` |
| `aggregatorPriceIdA` | `string \| null \| undefined` | Yes (null ok) | From `Port.getAggregatorPriceId({ coinType: coinTypeA })` |
| `aggregatorPriceIdB` | `string \| null \| undefined` | Yes (null ok) | From `Port.getAggregatorPriceId({ coinType: coinTypeB })` |
| `portRewardCoinTypes` | `SuiCoinType[]` | Yes | Port incentive reward types. Pass `[]` if none. |
| `poolRewardCoinTypes` | `SuiCoinType[]` | Yes | Pool trading reward types. Pass `[]` if none. |
| `oSailRewards` | `Array<{ coinType, amount, expired }>` | Yes | Pending oSAIL rewards. Pass `[]` if none. |
| `currentOSailCoinType` | `SuiCoinType` | Yes | Active oSAIL coin type |
| `oSailDecimals` | `number` | Yes | oSAIL token decimals (typically 6) |
| `sailPrice` | `number` | Yes | Current SAIL price in USD — NO SDK method, must use external source |
| `rewardChoice` | `'sail' \| 'usd' \| 'vesail'` | Yes | oSAIL exercise choice |

The five fields distinguishing `removeLiquidityTransaction` from `addLiquidityTransaction` are: `volume`, `tickLower`, `tickUpper`, `liquidity`, `currentSqrtPrice`. All five describe the current on-chain position state.

```typescript
// Partial withdrawal — remove liquidity but keep PortEntry alive
const transaction = await fullSailSDK.PortEntry.removeLiquidityTransaction({
  coinTypeA,
  coinTypeB,
  poolId,
  gaugeId,
  portId,
  portEntryId,
  volume,              // bigint — share volume to withdraw
  pythPriceIdA,
  pythPriceIdB,
  aggregatorPriceIdA,
  aggregatorPriceIdB,
  // Position state (current on-chain values):
  amountA, amountB, tickLower, tickUpper, liquidity, currentSqrtPrice,
  slippage,
  // Reward claim params (pass empty arrays if no pending rewards):
  portRewardCoinTypes,
  poolRewardCoinTypes,
  oSailRewards,        // enriched with expired field via Coin.getOSailMap()
  currentOSailCoinType,
  oSailDecimals,
  sailPrice,           // from external source — no SDK method
  rewardChoice,        // 'sail' | 'usd' | 'vesail'
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

**Use `removeLiquidityTransaction()` for partial exits. Use `closeTransaction()` to fully exit and destroy the PortEntry.**

---

### Close Vault Position

Fully exits a vault position and destroys the PortEntry object. Uses `PortEntry.closeTransaction()`.

`closeTransaction` accepts the same parameters as `removeLiquidityTransaction` — identical fields, no additions or removals. It withdraws all liquidity and then destroys the PortEntry NFT on-chain. Use this to fully exit a vault position.

```typescript
// Full exit — withdraws all liquidity and destroys the PortEntry object
const transaction = await fullSailSDK.PortEntry.closeTransaction({
  // Identical params to removeLiquidityTransaction:
  coinTypeA, coinTypeB, poolId, gaugeId, portId, portEntryId,
  volume, amountA, amountB, tickLower, tickUpper, liquidity, currentSqrtPrice,
  slippage,
  pythPriceIdA, pythPriceIdB, aggregatorPriceIdA, aggregatorPriceIdB,
  portRewardCoinTypes, poolRewardCoinTypes, oSailRewards,
  currentOSailCoinType, oSailDecimals, sailPrice, rewardChoice,
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

**`closeTransaction()` destroys the PortEntry NFT on-chain via `vault.port.destoryPortEntry`. After this call the user's PortEntry object no longer exists.**

---

