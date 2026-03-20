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

**See:** [§Protocol Fundamentals > Transaction Lifecycle Model](protocol-fundamentals.md#transaction-lifecycle-model)

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
// CORRECT — fetch current epoch oSAIL, check expiry, then use its address
const currentOSail = await fullSailSDK.Coin.getCurrentEpochOSail();
if (Date.now() > currentOSail.expiration_timestamp) { // .expiration_timestamp — confirmed field name from SDK type definitions
  throw new Error("oSAIL is expired for this epoch — cannot proceed");
}
const lock = await fullSailSDK.Lock.createLockFromOSailTransaction({
  oSailCoinType: currentOSail.address, // confirmed fresh and unexpired
  // ...
});
```

**See:** [§Protocol Fundamentals > oSAIL Expiry Rule](protocol-fundamentals.md#osail-expiry-rule), [§Rewards and oSAIL > oSAIL Expiry Check](rewards.md#osail-expiry-check)

---

### ERR-03: Assuming All Rewards Auto-Claim

**Failure mode:** Agent assumes all reward types (fees, pool rewards, oSAIL) are claimed automatically during liquidity operations.

**Correct behavior:** Only oSAIL is auto-claimed during staking/unstaking operations. Trading fees and pool token rewards require explicit `claim*` calls.

**Rule:** **Only oSAIL auto-claims during liquidity operations — fees and pool rewards require explicit `claim*` calls.**

Consult the full reward type mapping before any claim operation. Each reward type has a distinct claim method and eligibility path.

**See:** [§Rewards and oSAIL > Reward Claim Matrix](rewards.md#reward-claim-matrix)

---

### ERR-04: Using Wrong Claim Method for Stake Status

**Failure mode:** Agent calls staked-position claim methods on an unstaked position (or vice versa), causing transaction failure or claiming the wrong reward type.

**Correct behavior:** Determine stake status first using `### Staked vs Unstaked Detection`, then follow the appropriate claim path in the decision tree.

**Rule:** **Always determine position stake status before selecting a claim method — staked and unstaked positions use entirely different claim paths.**

Check whether the position has a `stake_info` field populated before selecting any claim method. The decision tree maps each reward type to the correct method for each stake status.

**See:** [§Liquidity Positions > Staked vs Unstaked Detection](liquidity.md#staked-vs-unstaked-detection), [§Rewards and oSAIL > Staked vs Unstaked Claim Path](rewards.md#staked-vs-unstaked-claim-path)

---

### ERR-05: Merge or Split Resets Voting Weight

**Failure mode:** Agent merges or splits locks after voting, expecting existing votes on those locks to persist.

**Correct behavior:** `Lock.mergeTransaction()` and `Lock.splitTransaction()` both irreversibly reset ALL voting weight on the affected locks. After merge or split, the agent must re-cast all governance votes.

**Rule:** **Merge and split are irreversible voting-weight resets — re-cast all votes on affected locks after either operation.**

> **BLOCKING WARNING:** `mergeTransaction()` and `splitTransaction()` reset all voting weight on affected locks. This cannot be undone. Re-vote after any merge or split.

**See:** [§Locks and veSAIL > Lock.mergeTransaction()](locks.md#lockmergetransaction), [§Locks and veSAIL > Lock.splitTransaction()](locks.md#locksplittransaction), [§Governance Voting > Vote Reset Behavior](governance.md#vote-reset-behavior)

---

### ERR-06: Setting Sender Before Wallet Connect

**Failure mode:** Agent calls `setSenderAddress()` during SDK initialization, before wallet connection is confirmed.

**Correct behavior:** `setSenderAddress()` must be called only after wallet connection is confirmed — not during SDK init.

**Rule:** **Call `setSenderAddress()` after wallet connect, never during SDK initialization.**

```typescript
// WRONG — setSenderAddress called at init time, before wallet is connected
const fullSailSDK = initFullSailSDK({ /* config */ });
fullSailSDK.setSenderAddress(walletAddress); // wallet not yet confirmed connected
```

```typescript
// CORRECT — setSenderAddress called in wallet connect callback, after connection confirmed
const fullSailSDK = initFullSailSDK({ /* config */ });

walletClient.on("connect", (wallet) => {
  fullSailSDK.setSenderAddress(wallet.address); // called after confirmed connect
});
```

**See:** [§Protocol Fundamentals > Two-Phase Sender Setup](protocol-fundamentals.md#two-phase-sender-setup)

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

**See:** [§Protocol Fundamentals > Pool Data Sources: Chain Pool vs Backend Pool](protocol-fundamentals.md#pool-data-sources-chain-pool-vs-backend-pool)

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

**See:** [§Liquidity Positions > Position.removeLiquidityTransaction()](liquidity.md#positionremoveliquiditytransaction)

---

### ERR-09: Routing oSAIL Through DEX Swap

**Failure mode:** Agent includes oSAIL as an input or output token in `Swap.getSwapRoute()` or `Swap.swapRouterTransaction()`, resulting in route failure.

**Correct behavior:** oSAIL is not a standard DEX-tradeable token and has no swap routes. To convert oSAIL, use `Lock.createLockFromOSailTransaction()` (lock path) or the redemption options documented in `### oSAIL Redemption Options`.

**Rule:** **oSAIL cannot be routed through the DEX — use the lock or redemption path instead.**

```typescript
// WRONG — oSAIL has no DEX route; this call will fail
const route = await fullSailSDK.Swap.getSwapRoute({
  from: oSailCoinType, // fails — no DEX route for oSAIL
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

**See:** [§Swaps > oSAIL Swap Restriction](swaps.md#osail-swap-restriction), [§Rewards and oSAIL > oSAIL Redemption Options](rewards.md#osail-redemption-options)

---

### ERR-10: Reading State Immediately After Transaction Submit

**Failure mode:** Agent reads on-chain state (position balance, lock amount, reward totals) in the same request immediately after submitting a transaction and acts on stale data.

**Correct behavior:** On-chain state on Sui takes time to finalize after transaction submission. Do not read state in the same logical step as submit. Wait for transaction finality confirmation before querying state.

**Rule:** **Do not read on-chain state immediately after submitting a transaction — wait for finality before querying updated state.**

After calling `signAndExecuteTransaction()`, the returned result confirms submission but on-chain objects may not yet reflect the update. Query state only after the transaction has reached finality on the Sui network.

---

### ERR-11: Passing `0x`-Prefixed Price IDs to Pyth Hermes

**Failure mode:** Agent (or bot) passes Pyth price IDs that include the `0x` prefix directly to `getPriceFeedsUpdateData()` (via `PortModule.updatePrices` / `addLiquidityTransaction`). The Hermes API strips `0x` internally and then hex-decodes the remainder — but Sui stores 32-byte identifiers with leading zeros omitted, so the remainder can be an odd number of hex chars (e.g. 63), triggering: `HTTP 400 — Failed to deserialize query string. Error: Odd number of digits`.

**Correct behavior:** Strip the `0x` prefix from every Pyth price ID before passing it to any Hermes/Pyth call.

**Rule:** **Always strip the `0x` prefix from Pyth price IDs before passing them to Hermes — Sui price IDs are compressed and become odd-length hex strings when the prefix is not removed first.**

```typescript
// WRONG — price ID includes 0x prefix; Hermes will produce "Odd number of digits"
const priceUpdateData = await connection.getPriceFeedsUpdateData([rawPriceId]);
```

```typescript
// CORRECT — strip 0x prefix before passing to Hermes
const priceUpdateData = await connection.getPriceFeedsUpdateData([
  rawPriceId.replace(/^0x/, ''),
]);
```

> **Note:** This applies to any price ID sourced from the Full Sail SDK (e.g., from `PortEntry` oracle fields). The SDK returns them with `0x`; Hermes requires them without.

---

### ERR-12: Not Consuming the Output Coin from `swapRouterTransaction`

**Failure mode:** `swapRouterTransaction` returns `[tx, coinOut]` where `coinOut` is an unconsumed `TransactionObjectArgument`. Signing and submitting `tx` without consuming `coinOut` causes a Sui "dangling object" transaction error — every object touched in a transaction must be consumed.

**Correct behavior:** Either set `isTransferToSender: true` (SDK transfers `coinOut` to the sender internally), or pass `coinOut` into a downstream call within the same transaction (e.g. an `addLiquidity` step in a compound bot).

**Rule:** **Always consume the `coinOut` returned by `swapRouterTransaction`. Use `isTransferToSender: true` for standalone swaps. For composable flows, pass `coinOut` to the next step in the same transaction.**

```typescript
// WRONG — coinOut is dangling; tx will fail
const [tx, coinOut] = await fullSailSDK.Swap.swapRouterTransaction({ router, slippage })
await wallet.signAndExecuteTransaction({ transaction: tx })
```

```typescript
// CORRECT (standalone) — isTransferToSender: true consumes coinOut internally
const [tx, coinOut] = await fullSailSDK.Swap.swapRouterTransaction({
  router,
  slippage,
  isTransferToSender: true,
})
await wallet.signAndExecuteTransaction({ transaction: tx })
```

```typescript
// CORRECT (composable) — coinOut passed downstream in the same tx
const [tx, coinOut] = await fullSailSDK.Swap.swapRouterTransaction({ router, slippage })
tx.transferObjects([coinOut], senderAddress) // or feed into addLiquidityTransaction
await wallet.signAndExecuteTransaction({ transaction: tx })
```

---

### Pre-Flight Checklist

Before executing any Full Sail transaction, verify:

- [ ] SDK sender is set — called after wallet connect, not at SDK init ([§Two-Phase Sender Setup](protocol-fundamentals.md#two-phase-sender-setup))
- [ ] Using Chain Pool from `getByIdFromChain()` — not Backend Pool from `getById()` ([§Pool Data Sources](protocol-fundamentals.md#pool-data-sources-chain-pool-vs-backend-pool))
- [ ] oSAIL expiry checked — fetched via `Coin.getCurrentEpochOSail()` and not past expiry ([§oSAIL Expiry Rule](protocol-fundamentals.md#osail-expiry-rule))
- [ ] Position stake status determined before selecting claim method ([§Staked vs Unstaked Claim Path](rewards.md#staked-vs-unstaked-claim-path))
- [ ] Transaction signed and executed via wallet client — not just constructed ([§Transaction Lifecycle Model](protocol-fundamentals.md#transaction-lifecycle-model))
- [ ] Pyth price IDs have `0x` prefix stripped before passing to Hermes/`getPriceFeedsUpdateData` ([§ERR-11](#err-11-passing-0x-prefixed-price-ids-to-pyth-hermes))
- [ ] `swapRouterTransaction` output coin consumed — either `isTransferToSender: true` or passed downstream in the same tx ([§ERR-12](#err-12-not-consuming-the-output-coin-from-swaproutertransaction))

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
