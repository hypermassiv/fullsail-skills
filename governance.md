> SDK version: v9.0.0 | Audit date: 2026-03-21

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

**Votes can be adjusted at any point during the voting window by calling `Lock.batchVoteTransaction()` again. A new call replaces the prior vote allocation for the specified locks — no reset call is needed beforehand.**

---

### Fee Reward Timing

Fee rewards are not distributed in the same epoch they are collected. The timing differs by reward type:

| Reward Type | When Available |
|-------------|----------------|
| Trading fee rewards (80% passive) | Epoch N+2 — two-week delay after collection epoch |
| oSAIL exercise fee rewards | End of epoch N — available immediately at epoch close |

**Trading fee rewards collected in epoch N are distributed in epoch N+2 — there is a two-week delay.** An agent expecting trading fee rewards from the current epoch will not receive them until two epochs later.

**oSAIL redemption (exercise) fee rewards are available immediately at epoch end** — unlike trading fee rewards, there is no multi-epoch delay.

---

### Managed Lock Fee Restriction

A lock can be deposited into a managed escrow position (for example, by a third-party protocol or automated vault). Such locks are detectable by checking `escrow_type.is_locked() == true` on the lock object.

**Locks with `escrow_type.is_locked() == true` cannot claim passive fees.** The managed escrow state blocks passive fee distribution for that lock.

**Rule:** Before claiming passive governance fees for a lock, check `escrow_type.is_locked()`. If `true`, passive fee claims will fail or return zero — the lock is in managed escrow and passive fees are not claimable directly by the lock owner.

---

### Lock.batchVoteTransaction()

Allocates veSAIL voting power from one or more locks across liquidity pools. Voting determines fee distribution for the epoch.

| Parameter | Type | Description |
|-----------|------|-------------|
| locks | BatchVoteLock[] | Locks contributing voting power — each entry needs `id` and `votingPower` |
| votes | BatchVoteItem[] | Vote allocations per pool |

**locks array members (BatchVoteLock):**

| Field | Type | Description |
|-------|------|-------------|
| id | string (SuiObjectId) | Lock object ID |
| votingPower | bigint | The lock's current voting power — obtain from the lock object (e.g., via `Lock.getById()`), not hardcoded |

**votes array members (BatchVoteItem):**

| Field | Type | Description |
|-------|------|-------------|
| poolId | string (SuiObjectId) | Pool to allocate voting weight to |
| weight | bigint | Abstract ratio units — any values; SDK normalizes total to 100% |
| volume | bigint | Predicted pool volume for next epoch, in USD with 6 decimal places (e.g., `5000000000n` = $5000.00) |

**Returns unsigned Transaction — must be signed and submitted separately.**

**`weight` values are abstract ratio units — NOT percentages required to sum to 100. The SDK normalizes any values to 100%. Pass values that make the intended split clear (e.g., `70n` and `30n` for a 70%/30% allocation).**

**`batchVoteTransaction` lives on the `Lock` namespace — `fullSailSDK.Lock.batchVoteTransaction()`. There is no separate Governance namespace.**

**Votes must be cast during the voting window (01:00 UTC Thursday to 23:00 UTC Wednesday). Votes submitted outside the window may fail or be ignored.**

**Passing the old `lockIds: string[]` parameter will cause `locks` to be `undefined` inside the SDK, crashing at runtime. Always use `locks: BatchVoteLock[]`.**

```typescript
// Source: dist/index.d.ts verified against dist/index.js (2026-03-21)
// Returns unsigned Transaction — must be signed and submitted separately
// locks: each lock that contributes voting power in this batch
//   id: the lock object ID
//   votingPower: the lock's current veSAIL power — obtain from the lock object
//     e.g., const lock = await fullSailSDK.Lock.getById(lockId); votingPower = lock.locked_amount
// weight: abstract units — any values normalize to 100%
//   [70n, 30n] = 70%/30% split
//   [1n, 1n]   = 50%/50% split (equal weight)
//   [20n, 80n] = 20%/80% split
// volume: predicted pool volume for next epoch, in USD with 6 decimal places (1000000n = $1.00)

const transaction = await fullSailSDK.Lock.batchVoteTransaction({
  locks: [
    { id: lockId1, votingPower: lock1VotingPower }, // votingPower: bigint from lock object, not hardcoded
    { id: lockId2, votingPower: lock2VotingPower },
  ],
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

### isVoted Parameter

`isVoted` is a boolean pre-flight parameter on two `Lock` methods: `splitTransaction` and `disablePermanentTransaction`. It is NOT a standalone query function — it is a flag that tells the SDK whether to reset active governance votes on the lock before completing the operation.

| Method | Parameter | Type | Description |
|--------|-----------|------|-------------|
| `Lock.splitTransaction` | `isVoted` | `boolean` (optional) | Pass `true` if the lock being split currently has active votes — SDK resets votes before splitting |
| `Lock.disablePermanentTransaction` | `isVoted` | `boolean` (optional) | Pass `true` if the permanent lock currently has active votes — SDK resets votes before unlocking |

**How to determine the correct value:** Check whether the lock has cast votes in the current epoch before calling `splitTransaction` or `disablePermanentTransaction`. Pass `true` if it has voted, `false` (or omit) if it has not. Passing `false` when the lock has active votes will leave stale vote records — the split or permanent-unlock will succeed but the lock's vote state may be inconsistent.

```typescript
// Source: dist/index.d.ts verified against dist/index.js (2026-03-21)
// Check whether the lock has active votes before splitting
const hasVoted = /* determine from your application state or on-chain query */

const transaction = await fullSailSDK.Lock.splitTransaction({
  lockId: lockId,
  splitAmount: amountToSplit,
  isVoted: hasVoted, // true if lock has active votes — SDK resets them before splitting
})
```

**This is not a query method.** There is no `Lock.isVoted(epoch, address)` function in the SDK. The `isVoted` flag must be determined by the caller from application state or a separate on-chain query.

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

