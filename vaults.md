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

| Parameter | Type | Source |
|-----------|------|--------|
| `coinTypeA` | `SuiCoinType` | Pool token A type |
| `coinTypeB` | `SuiCoinType` | Pool token B type |
| `amountA` | `bigint` | Token A deposit amount in base units |
| `amountB` | `bigint` | Token B deposit amount in base units |
| `poolId` | `SuiObjectId` | From `pool.id` (backend API) |
| `gaugeId` | `SuiObjectId` | From `pool.gauge_id` (backend API) |
| `portId` | `SuiObjectId` | From `ports[0].id` (backend API) |
| `pythPriceIdA/B` | `string \| null` | From `Port.getPythPriceFeedId()` |
| `aggregatorPriceIdA/B` | `string \| null` | From `Port.getAggregatorPriceId()` |
| `rewardCoinTypes` | `SuiCoinType[]` | Reward token types for the port (from `port.rewards`) |
| `currentOSailCoinType` | `SuiCoinType` | Active oSAIL coin type |

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
  rewardCoinTypes: ports[0].rewards.map(r => r.token.coin_type),
  currentOSailCoinType,  // active oSAIL coin type from protocol config
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

**`poolId`, `gaugeId`, and `portId` must all come from the same backend API response — do not mix IDs from different sources.**

---

### Vault Withdrawal

Exits a vault position partially or fully. Use `removeLiquidityTransaction()` to withdraw a portion; use `closeTransaction()` to withdraw everything and destroy the PortEntry object.

> **PRECONDITION:** You need the user's `portEntryId` — the Sui object ID of their PortEntry — from their owned objects on-chain.

`PortEntryRemoveLiquidityParams` extends the deposit params with additional fields:

| Additional parameter | Type | Description |
|----------------------|------|-------------|
| `portEntryId` | `SuiObjectId` | User's PortEntry object ID |
| `volume` | `bigint` | Share volume to withdraw |
| `amountA` / `amountB` | `bigint` | Current position token amounts (for slippage calc) |
| `tickLower` / `tickUpper` | `number` | Current position tick range |
| `liquidity` | `bigint` | Current position liquidity |
| `currentSqrtPrice` | `bigint` | Current pool sqrt price |
| `slippage` | `Percentage` | Maximum acceptable slippage |
| `portRewardCoinTypes` | `SuiCoinType[]` | Port incentive reward types |
| `poolRewardCoinTypes` | `SuiCoinType[]` | Pool trading reward types |
| `oSailRewards` | `Array<{ coinType, amount, expired }>` | Pending oSAIL rewards |

```typescript
// Partial withdrawal — remove liquidity but keep PortEntry alive
const transaction = await fullSailSDK.PortEntry.removeLiquidityTransaction({
  coinTypeA,
  coinTypeB,
  poolId,
  gaugeId,
  portId,
  portEntryId,
  volume,            // bigint — share volume to withdraw
  pythPriceIdA,
  pythPriceIdB,
  aggregatorPriceIdA,
  aggregatorPriceIdB,
  // Position state for slippage calculation:
  amountA, amountB, tickLower, tickUpper, liquidity, currentSqrtPrice,
  slippage,
  // Reward claim params (pass empty arrays if no pending rewards):
  portRewardCoinTypes,
  poolRewardCoinTypes,
  oSailRewards,
  // ... additional reward claim fields (see PortEntryClaimAllRewardsParam)
})

// Full exit — withdraws all liquidity and destroys the PortEntry object
const transaction = await fullSailSDK.PortEntry.closeTransaction({
  // Same params as removeLiquidityTransaction, plus:
  // portEntryId is also used for the destroy call (included via PortEntryCloseParams)
  ...removeLiquidityParams,
})

const result = await wallet.signAndExecuteTransaction({ transaction })
```

**Use `closeTransaction()` when fully exiting — it destroys the PortEntry object and reclaims the deposit. Use `removeLiquidityTransaction()` for partial exits.**

---

