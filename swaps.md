## Swaps

The Full Sail SDK provides two swap paths — a router path (2 calls) and a direct pool path (4 calls).

### oSAIL Swap Restriction

> **oSAIL Swap Restriction:** oSAIL is an option token — it is not DEX-tradeable. Do not use oSAIL as `from` or `target` in any swap route. To convert oSAIL to SAIL or USDC, use the redemption path — see `## Rewards and oSAIL`.

### Swap.getSwapRoute()

Returns an optimized route across pools via the Aftermath router.

| Parameter | Type | Description |
|-----------|------|-------------|
| `from` | `string` | Coin type address of the input token |
| `target` | `string` | Coin type address of the output token |
| `amount` | `bigint \| string \| number` | Amount of input token in base units |

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

Builds the unsigned swap transaction using a router route.

**Returns a tuple `[Transaction, TransactionObjectArgument]` — `[0]` is the transaction to sign; `[1]` is the output coin object argument.**

> **Output coin must be consumed.** The returned `coinOut` is an unconsumed `TransactionObjectArgument`. Every Sui transaction must consume every object it touches — if `coinOut` is not transferred or passed into another call within the same transaction, the transaction will fail with a "dangling object" error. Use `isTransferToSender: true` for standalone swaps, or pass `coinOut` to a downstream call (e.g. `addLiquidityTransaction`) for composable flows.

| Parameter | Type | Description |
|-----------|------|-------------|
| `router` | `SwapRoute` | Route object from `getSwapRoute()` — pass directly |
| `slippage` | `Percentage` | `Percentage` object — e.g. `Percentage.fromNumber(1)` for 1% |
| `isTransferToSender` | `boolean` | If `true`, the SDK calls `tx.transferObjects([coinOut], senderAddress)` internally, consuming the output coin and sending it to the wallet. Set `true` for standalone swaps. Omit or set `false` when composing — you must then consume `coinOut` yourself. |

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

