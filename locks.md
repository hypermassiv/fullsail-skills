## Locks and veSAIL

The veSAIL lock lifecycle — create, increase, merge, split, transfer, and toggle permanent status — is managed via the `Lock` namespace. All `*Transaction` methods return unsigned `Transaction` objects and must be signed and submitted separately. Governance voting (`batchVoteTransaction`) is documented in ## Governance Voting.

### Voting Power

veSAIL voting power is proportional to both the locked SAIL amount and the lock duration relative to the maximum duration (4 years = 1461 days):

**Formula:** `voting_power = locked_amount × (lock_duration / max_duration)`

| Example | Locked SAIL | Lock Duration | Voting Power (veSAIL) |
|---------|-------------|---------------|----------------------|
| Max lock | 1 SAIL | 4 years | 1 veSAIL |
| Half duration | 1 SAIL | 2 years | 0.5 veSAIL |
| Minimum useful | 1 SAIL | 6 months | 0.125 veSAIL |

**Permanent locks** (enabled via `Lock.enablePermanentTransaction()`) hold the equivalent of a max-duration lock — full voting power proportional to the locked SAIL amount, with no expiry.

---

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
| durationDays | number | Lock duration in days — likely additive (adds to remaining duration) based on the underlying `increaseUnlockTime` contract method name, but not yet confirmed on testnet (MEDIUM confidence) |

**Returns unsigned Transaction — must be signed and submitted separately.**

**`durationDays` is likely additive — passing `365` probably extends the lock by 1 year from its current expiry, not from now. Evidence: the underlying contract method is `increaseUnlockTime`, which implies adding time. However, this has not been confirmed on testnet (MEDIUM confidence — deferred to v1.3 VER-07).**

**Not applicable to permanent locks — permanent locks have no expiry and cannot have their duration modified via this method.**

```typescript
// Returns unsigned Transaction — must be signed and submitted separately
// durationDays is likely additive (extends from current expiry), not absolute — MEDIUM confidence, see note above

const transaction = await fullSailSDK.Lock.increaseDurationTransaction({
  lockId,
  durationDays: 365 * 4, // new expiry = 4 years from now
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

