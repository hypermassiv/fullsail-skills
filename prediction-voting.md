> SDK version: v9.0.0 | Audit date: 2026-03-21

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
- **veSAIL holders who do NOT submit `batchVoteTransaction` during the voting window receive ZERO accuracy share for that epoch — the 20% active predictor portion requires voting. The 80% passive share flows to all veSAIL holders proportionally regardless of voting status.**

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
| Passive veSAIL distribution | 80% of total fees | All veSAIL holders with active veSAIL balance (proportional to balance) — no voting required |
| Active predictor accuracy rewards | 20% of total fees | veSAIL holders who submitted batchVoteTransaction with volume, weighted by prediction accuracy |

The 80% passive share flows to all veSAIL holders proportionally — no voting required. The 20% accuracy share requires submitting `batchVoteTransaction` with a volume prediction — veSAIL holders who do not vote receive zero accuracy share for that epoch.

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

