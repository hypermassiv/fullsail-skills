> SDK version: v9.0.0 | Audit date: 2026-03-21

## Swaps

The Full Sail SDK provides two swap paths — a router path (2 calls) and a direct pool path (4 calls).

### oSAIL Swap Restriction

> **oSAIL Swap Restriction:** oSAIL is an option token — it is not DEX-tradeable. Do not use oSAIL as `from` or `target` in any swap route. To convert oSAIL to SAIL or USDC, use the redemption path — see `## Rewards and oSAIL`.

### Swap.getSwapRoute()

Returns an optimized route across pools via the Aftermath router.

_Verified against `dist/index.js` and `dist/index.d.ts`._

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `from` | `string` | YES | Coin type address of the input token |
| `target` | `string` | YES | Coin type address of the output token |
| `amount` | `BN \| string \| number \| bigint` | YES | Amount of input token in base units |
| `providers` | `string[]` | NO | Subset of available router providers; defaults to all |
| `excludeOracleProviders` | `boolean` | NO | Removes oracle-dependent providers (HAEDAL, STEAMM_OMM, etc.) |
| `depth` | `number` | NO | Routing depth |
| `splitAlgorithm` | `string` | NO | Split algorithm selection |
| `splitFactor` | `number` | NO | Split factor |
| `splitCount` | `number` | NO | Split count |
| `liquidityChanges` | `PreSwapLpChangeParams[]` | NO | Simulated LP changes to apply before routing |

Return value: Returns a `SwapRoute` object — pass it as `router` to `swapRouterTransaction`. The route structure is opaque; do not enumerate or manipulate its fields.

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
const from = '0x...'    // input token type address
const target = '0x...'  // output token type address
const amount = 1000000n // input amount in base units

const router = await fullSailSDK.Swap.getSwapRoute({
  from,
  target,
  amount,
})
// router is opaque — pass directly to swapRouterTransaction
```

---

### Swap.swapRouterTransaction()

Builds the unsigned swap transaction. Has two call signatures depending on the swap path used.

**Returns a tuple `[Transaction, TransactionObjectArgument]` — `[0]` is the transaction to sign; `[1]` is the output coin object argument.**

> **Output coin must be consumed.** The returned `coinOut` is an unconsumed `TransactionObjectArgument`. Every Sui transaction must consume every object it touches — if `coinOut` is not transferred or passed into another call within the same transaction, the transaction will fail with a "dangling object" error. Use `isTransferToSender: true` for standalone swaps, or pass `coinOut` to a downstream call (e.g. `addLiquidityTransaction`) for composable flows.

#### Router-path signature (use with `getSwapRoute`)

_Verified against `dist/index.d.ts`._

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `router` | `SwapRoute` | YES | Route object from `getSwapRoute()` — pass directly |
| `slippage` | `Percentage` | YES | `Percentage` object — e.g. `Percentage.fromNumber(1)` for 1% |
| `inputCoin` | `TransactionObjectArgument` | NO | Pre-built coin object for composable flows. If omitted, SDK fetches coin assets from sender and builds internally. Pass when chaining transactions to avoid double-spend. |
| `isTransferToSender` | `boolean` | NO | If `true`, the SDK calls `tx.transferObjects([coinOut], senderAddress)` internally, consuming the output coin and sending it to the wallet. Set `true` for standalone swaps. Omit or set `false` when composing — you must then consume `coinOut` yourself. |

#### Direct-pool-path signature (use without `getSwapRoute`)

`swapRouterTransaction` also accepts a direct pool call signature that bypasses the router. Use this when you already know the pool and want to skip route discovery.

_Verified against `dist/index.js`._

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `pool` | `SuiObjectId` | YES | Pool object ID |
| `coinTypeA` | `string` | YES | Pool's token A type |
| `coinTypeB` | `string` | YES | Pool's token B type |
| `a2b` | `boolean` | YES | Swap direction (A-to-B) |
| `byAmountIn` | `boolean` | YES | `true`: amount is input; `false`: amount is desired output |
| `amount` | `bigint` | YES | Swap amount in base units |
| `amountLimit` | `bigint` | YES | Min output or max input (slippage guard) |
| `inputCoin` | `TransactionObjectArgument` | NO | Pre-built coin object for composable flows. If omitted, SDK auto-splits from wallet. Pass when chaining transactions to avoid double-spend. |
| `isTransferToSender` | `boolean` | NO | If `true`, SDK transfers `coinOut` to sender internally. Set `true` for standalone swaps. |
| `senderAddress` | `string` | YES | Sender wallet address |

`Percentage` is exported from `@fullsailfinance/sdk`.

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Router path — standalone swap (isTransferToSender: true)

import { Percentage } from '@fullsailfinance/sdk'

const router = await fullSailSDK.Swap.getSwapRoute({ from, target, amount })

// isTransferToSender: true — SDK transfers coinOut to sender internally
const [transaction, coinOut] = await fullSailSDK.Swap.swapRouterTransaction({
  router,
  slippage: Percentage.fromNumber(1),
  isTransferToSender: true,
})
const result = await wallet.signAndExecuteTransaction({ transaction })
```

```typescript
// Composable swap — coinOut passed downstream (e.g. into addLiquidityTransaction)
// isTransferToSender omitted (false) — caller is responsible for consuming coinOut

const [tx, coinOut] = await fullSailSDK.Swap.swapRouterTransaction({
  router,
  slippage: Percentage.fromNumber(1),
  // isTransferToSender not set — coinOut is unconsumed
})

// coinOut MUST be consumed in the same tx — pass it to the next step
// e.g. tx.transferObjects([coinOut], senderAddress), or feed into addLiquidityTransaction
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
| `currentSqrtPrice` | `bigint` | From Chain Pool (`getByIdFromChain`) — must be real-time. Type is `bigint`, not `number` (ERR-16). |

Return value: Returns `PreSwapResult` containing `estimatedAmountOut: bigint`, `estimatedAmountIn: bigint`, `estimatedEndSqrtPrice: bigint`, `estimatedFeeAmount: bigint`, `isExceed: boolean`, `currentSqrtPrice: bigint`, `amount: bigint`, and `swapParams` (see `swapTransaction` below). The `swapParams` field is `Pick<SwapParams, 'coinTypeA' | 'coinTypeB' | 'byAmountIn' | 'poolId' | 'isAtoB'>` — spread it directly into `swapTransaction`.

**Always derive `isAtoB` from the pool's `coinTypeA` field: `const isAtoB = chainPool.coinTypeA === coinInType`. Never hardcode `isAtoB`.**

**Use Chain Pool (`Pool.getByIdFromChain`) for `currentSqrtPrice`. Backend Pool data is stale and produces incorrect estimates.**

---

### Swap.swapTransaction()

Builds the unsigned swap transaction for a direct pool swap. Requires a prior `preSwap` call — its output provides 5 of the parameters.

**Returns unsigned transaction — must be signed and submitted via `wallet.signAndExecuteTransaction()`.**

_Verified against `dist/index.js`._

| Parameter | Type | Required | Source | Description |
|-----------|------|----------|--------|-------------|
| `amount` | `bigint` | YES | Caller | Same amount passed to `preSwap` |
| `amountLimit` | `bigint` | YES | Caller | Min output if `byAmountIn=true` (`presSwap.estimatedAmountOut`); max input if `false` (`presSwap.estimatedAmountIn`) |
| `slippage` | `Percentage` | YES | Caller | `Percentage` object (e.g., `Percentage.fromNumber(1)` for 1%) |
| `coinTypeA` | `string` | YES | From `presSwap.swapParams` | Pool's token A type |
| `coinTypeB` | `string` | YES | From `presSwap.swapParams` | Pool's token B type |
| `poolId` | `SuiObjectId` | YES | From `presSwap.swapParams` | Pool object ID |
| `isAtoB` | `boolean` | YES | From `presSwap.swapParams` | Swap direction |
| `byAmountIn` | `boolean` | YES | From `presSwap.swapParams` | `true`: amount is input; `false`: amount is desired output |
| `partner` | `string` | NO | Caller | Optional partner address |

The 5 `presSwap.swapParams` fields (`coinTypeA`, `coinTypeB`, `poolId`, `isAtoB`, `byAmountIn`) are typically supplied via spread (`...presSwap.swapParams`) in code, but they are explicit named parameters — not an opaque blob. Do NOT add `inputCoin` or `isTransferToSender` here; those belong to `swapRouterTransaction`.

---

### Direct Pool Swap: Complete 4-Step Sequence

```typescript
// Source: https://docs.fullsail.finance/developer/SDK
// Direct pool path — 4 steps: Chain Pool → coin metadata → preSwap → swapTransaction

const poolId = '0x...'         // pool object ID — replace with real pool ID
const coinInType = '0x...'     // input token type address
const coinOutType = '0x...'    // output token type address
import { Percentage } from '@fullsailfinance/sdk'
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

