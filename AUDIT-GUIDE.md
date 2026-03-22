# Full Sail SDK Audit Guide

**Purpose:** A self-contained, step-by-step checklist for re-auditing any Full Sail skill module when the SDK version bumps. No other document is required.

---

## 1. Prerequisites

**SDK package location:**
```
node_modules/@fullsailfinance/sdk/dist/
```

**Key files:**
- `dist/index.d.ts` — TypeScript type definitions (function signatures, interfaces, parameter types)
- `dist/index.js` — Runtime implementation (actual destructuring, return values, behavior)

**Tools needed:** grep, a text editor (or Read/Edit tools in a Claude session)

**Verify SDK version before starting:**
```bash
grep "version" node_modules/@fullsailfinance/sdk/package.json
```
Expected output: `"version": "9.0.0"` (or whatever version you are auditing against)

---

## 2. Locating SDK Source Files

**Absolute path pattern:**
```
/path/to/project/node_modules/@fullsailfinance/sdk/dist/index.d.ts
/path/to/project/node_modules/@fullsailfinance/sdk/dist/index.js
```

**For the Full Sail Skills project specifically:**
```
/Users/dicksongoh/Desktop/Full Sail Bot/node_modules/@fullsailfinance/sdk/dist/index.d.ts
/Users/dicksongoh/Desktop/Full Sail Bot/node_modules/@fullsailfinance/sdk/dist/index.js
```

**Confirm the file exists and is readable:**
```bash
wc -l /path/to/dist/index.d.ts
wc -l /path/to/dist/index.js
```
A healthy SDK package has thousands of lines in both files. If empty or missing, the package is not installed.

---

## 3. Module-by-Module Audit Checklist

Each skill file maps to one or more SDK namespaces. Use the namespace to scope your grep commands.

| Skill File | SDK Namespace | Key Functions to Audit |
|------------|---------------|------------------------|
| `swaps.md` | `Pool`, `Router` | `swapTransaction`, `swapRouterTransaction`, `getSwapRoute`, `preSwap` |
| `liquidity.md` | `Position` | `openPositionTransaction`, `addLiquidityTransaction`, `removeLiquidityTransaction`, `closeTransaction`, `stakeTransaction` |
| `rewards.md` | `Position`, `Lock` | `claimFeesTransaction`, `claimOSailRewards`, `claimPoolRewards`, `claimStakedPoolRewards` |
| `locks.md` | `Lock` | `createLockTransaction`, `increaseLockAmountTransaction`, `increaseDurationTransaction`, `mergeTransaction`, `splitTransaction`, `transferTransaction`, `disablePermanentTransaction`, `enablePermanentTransaction`, `unlockTransaction` |
| `governance.md` | `Lock` | `batchVoteTransaction`, `isVoted` |
| `vaults.md` | `PortEntry` | `createPortEntryTransaction`, `addLiquidityTransaction`, `removeLiquidityTransaction`, `closeTransaction` |
| `protocol-fundamentals.md` | Root | `initFullSailSDK` |
| `prediction-voting.md` | `Lock` (via `batchVoteTransaction`) | Same as governance.md — no separate prediction namespace |
| `pitfalls.md` | Cross-cutting | Verify ERR entries match current SDK behavior; no single namespace |

---

## 4. Per-Function Verification Procedure

Follow these steps for each function in the table above.

### Step 1: Find the function signature in index.d.ts

```bash
grep -n "functionName" dist/index.d.ts
```

Example:
```bash
grep -n "batchVoteTransaction\|BatchVoteParams" dist/index.d.ts
```

This gives you the line number of the method declaration AND any associated params interface. Note both line numbers.

### Step 2: Find the params interface definition

```bash
grep -n "interface ParamsInterfaceName" dist/index.d.ts
```

Then read the lines starting at that line number to see all fields. Count the closing brace to know how many lines to read:
```bash
sed -n '4635,4650p' dist/index.d.ts
```
(Replace line numbers with what grep returned.)

### Step 3: Trace interface inheritance

If the interface uses `extends`, `Omit<>`, `Pick<>`, or `&` (intersection), you must flatten it.

```bash
grep -n "extends\|Omit<\|Pick<" dist/index.d.ts | grep "TargetInterface"
```

Follow each parent interface recursively. Collect all fields from all ancestors.

### Step 4: Flatten the full field list

Produce a complete flat parameter list. See Section 5 (Interface Flattening Procedure) for the algorithm.

### Step 5: Compare each field against the skill file's parameter table

For each field in your flat list:
- Is it present in the skill file's param table? (If not: add it)
- Is the name correct? (Rename if wrong)
- Is the type correct? (Fix if wrong)
- Is required vs optional marked correctly? (Optional fields have `?` suffix in d.ts)

### Step 6: Verify in index.js

```bash
grep -n "functionName" dist/index.js
```

Read the implementation body (30-40 lines starting from the match):
```bash
sed -n 'LINE,LINE+40p' dist/index.js
```

Check:
- What parameters does the function actually destructure? (The destructure list IS the ground truth)
- What does it return? (d.ts types may not fully capture tuple returns or wrapped values)
- Does the implementation use a field that the d.ts declares but with different behavior?

### Step 7: Document findings

- **Correct parameter:** no change needed
- **Missing parameter:** add to skill file table
- **Wrong name or type:** fix in skill file, add ERR entry to pitfalls.md if runtime-breaking
- **d.ts and index.js disagree:** document the actual runtime behavior; flag in pitfalls.md

---

## 5. Interface Flattening Procedure

The Full Sail SDK uses deep TypeScript interface inheritance. You MUST flatten before comparing against docs.

### Algorithm

```
1. Start with the declared params type for the function
2. Read the interface definition
3. For each field:
   a. If the field is a simple type (string, bigint, boolean, etc.) → add to flat list
   b. If the field is itself an interface → recurse into that interface
4. For "extends ParentInterface":
   → Read ParentInterface, add all its fields to the flat list
   → Then continue reading the child interface for additional fields
5. For "Omit<ParentInterface, 'fieldName'>":
   → Add all fields from ParentInterface to the flat list
   → Remove 'fieldName' from the flat list
6. For "Pick<ParentInterface, 'fieldA' | 'fieldB'>":
   → Add only fieldA and fieldB from ParentInterface to the flat list
7. For "TypeA & TypeB" (intersection):
   → Add all fields from TypeA AND TypeB to the flat list
```

### Concrete Example

```typescript
interface A { x: string }
interface B extends A { y: number }
interface C extends Omit<B, 'x'> { z: boolean }
```

Flattening C:
1. C extends `Omit<B, 'x'>`: start with all fields of B, then remove 'x'
2. B has: x (from A), y (own field) → B flat = { x: string, y: number }
3. After Omit<B, 'x'>: { y: number }
4. C adds its own field: { y: number, z: boolean }
5. Final flat C: `{ y: number, z: boolean }`

### Real SDK Example: BatchVoteParams

```typescript
// From dist/index.d.ts
interface BatchVoteItem {
    poolId: SuiObjectId;
    weight: bigint;
    volume: bigint;
}
interface BatchVoteLock {
    id: SuiObjectId;
    votingPower: bigint;
}
interface BatchVoteParams {
    locks: BatchVoteLock[];
    votes: BatchVoteItem[];
}
```

Flattening `batchVoteTransaction` params:
1. Params type: `BatchVoteParams`
2. `locks: BatchVoteLock[]` → array of `{ id: SuiObjectId, votingPower: bigint }`
3. `votes: BatchVoteItem[]` → array of `{ poolId: SuiObjectId, weight: bigint, volume: bigint }`

Final flat parameter list for `batchVoteTransaction`:

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| locks | BatchVoteLock[] | Yes | Array of lock objects contributing voting power |
| locks[n].id | SuiObjectId | Yes | The lock's Sui object ID |
| locks[n].votingPower | bigint | Yes | The lock's veSAIL voting power (decimals 6) |
| votes | BatchVoteItem[] | Yes | Vote allocations per pool |
| votes[n].poolId | SuiObjectId | Yes | Pool to vote for |
| votes[n].weight | bigint | Yes | Relative weight (e.g., 70n for 70%) |
| votes[n].volume | bigint | Yes | Predicted volume for this pool |

---

## 6. Return Type Verification

### From index.d.ts (declared type)

The declared return type appears in the method signature:
```bash
grep -n "functionName" dist/index.d.ts
```
Look for `=> Promise<Transaction>` or similar.

### From index.js (actual runtime return)

Read the function body and find what it returns:
```bash
grep -n "functionName" dist/index.js
```
Then read the implementation. Look for `return tx`, `return [tx, result]`, `return { tx, ... }` etc.

### When they disagree

Example: d.ts says `Promise<Transaction>` but index.js actually returns `Promise<[Transaction, string]>`. The runtime behavior wins. Document the actual return in the skill file and add an ERR entry to pitfalls.md if the discrepancy would cause a silent failure.

---

## 7. Documenting Findings

### Updating the parameter table in a skill file

Replace the incorrect row or add the missing row. Example format (matches existing skill files):

```markdown
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| locks | BatchVoteLock[] | Yes | Array of lock objects contributing voting power |
```

For nested object arrays, add a sub-table immediately below:

```markdown
**`BatchVoteLock` fields:**
| Field | Type | Description |
|-------|------|-------------|
| id | SuiObjectId | Lock object ID |
| votingPower | bigint | Lock's veSAIL voting power (decimals 6) |
```

### Adding an ERR entry to pitfalls.md

Add ERR entries for runtime-breaking discrepancies. Follow the existing format in pitfalls.md. ERR entries have:
- **ERR-XX:** Title describing the failure mode
- **Failure mode:** What happens when an agent follows the wrong docs
- **Root cause:** Why the SDK behaves this way
- **Correct pattern:** The right way to call the function

Only add ERR entries for errors that would cause a **runtime failure or silent wrong behavior** — not for cosmetic or optional documentation gaps.

---

## 8. Frontmatter Tagging

After auditing a skill file (or confirming it is already correct), add the SDK version tag as the FIRST line of the file:

```
> SDK version: vX.Y.Z | Audit date: YYYY-MM-DD
```

Followed by a blank line before the existing content.

**Example (for v9.0.0 audited 2026-03-21):**
```
> SDK version: v9.0.0 | Audit date: 2026-03-21

## Swaps

The Full Sail SDK provides...
```

**Important:** This is a Markdown blockquote, NOT a YAML frontmatter block. Do NOT use `---` fences. The format must match `locks.md`, `rewards.md`, and `liquidity.md`.

Files that do NOT use this frontmatter format:
- `SKILL.md` — has YAML `---` frontmatter (different format, do not modify)
- `fullsail.md` — has YAML `---` frontmatter (different format, do not modify)

---

## 9. Worked Example: batchVoteTransaction Full Audit

This section walks through a complete end-to-end audit of `batchVoteTransaction` — the function fixed in Phase 15. Use this as a template for auditing any other function.

### Step 1: Identify the SDK path and open the skill file

Skill file: `governance.md`
SDK namespace: `Lock`
Function: `batchVoteTransaction`

### Step 2: Find the function signature in index.d.ts

```bash
grep -n "batchVoteTransaction\|BatchVoteParams" dist/index.d.ts
```

**Output (abridged):**
```
4635: interface BatchVoteItem {
4640: interface BatchVoteLock {
4644: interface BatchVoteParams {
4691: batchVoteTransaction: ({ locks, votes }: BatchVoteParams, transaction?: Transaction) => Promise<Transaction>;
```

Key findings:
- Params type is `BatchVoteParams` (line 4644)
- Method is on `LockModule` (line 4691)
- Second parameter `transaction?` is optional (pass existing tx or omit)

### Step 3: Read the BatchVoteParams interface

```bash
sed -n '4635,4650p' dist/index.d.ts
```

**Output:**
```typescript
interface BatchVoteItem {
    poolId: SuiObjectId;
    weight: bigint;
    volume: bigint;
}
interface BatchVoteLock {
    id: SuiObjectId;
    votingPower: bigint;
}
interface BatchVoteParams {
    locks: BatchVoteLock[];
    votes: BatchVoteItem[];
}
```

No inheritance — all interfaces are flat. The flat parameter list is immediate.

### Step 4: Verify in index.js

```bash
grep -n "batchVoteTransaction" dist/index.js
```

**Output (abridged):**
```
5524: batchVoteTransaction = async ({ locks, votes }, transaction) => {
5525:     const tx = transaction ?? new Transaction10();
5526:     const totalVotingPower = locks.reduce((sum, { votingPower }) => sum + votingPower, 0n);
```

Confirms:
- Destructures `{ locks, votes }` — parameter names are `locks` and `votes`
- Uses `lock.votingPower` and `lock.id` internally (not documented in d.ts directly but inferred from usage)
- Returns a `Transaction` object

### Step 5: Compare against the skill file

**What governance.md originally said:**
```markdown
| lockIds | string[] | All lock IDs contributing voting power |
```

**What SDK says:** `locks: BatchVoteLock[]` where `BatchVoteLock = { id: SuiObjectId, votingPower: bigint }`

Discrepancies found:
1. Parameter name wrong: `lockIds` → should be `locks`
2. Parameter type wrong: `string[]` → should be `BatchVoteLock[]`
3. `locks[n].id` field undocumented
4. `locks[n].votingPower` field undocumented

### Step 6: Apply fixes

Updated parameter table in governance.md:
```markdown
| locks | BatchVoteLock[] | Yes | Array of locks contributing voting power |
```
Plus added BatchVoteLock sub-table.

Updated code example:
```typescript
const transaction = await fullSailSDK.Lock.batchVoteTransaction({
  locks: [
    { id: lockId1, votingPower: lock1VotingPower },
    { id: lockId2, votingPower: lock2VotingPower },
  ],
  votes: [
    { poolId: poolId1, weight: 70n, volume: 5000000000n },
    { poolId: poolId2, weight: 30n, volume: 2000000000n },
  ],
})
```

### Step 7: Check for ERR entry

The `lockIds` → `locks` discrepancy is runtime-breaking: if an agent passes `lockIds: [...]`, the SDK destructures it as `locks: undefined` and crashes at `locks.reduce(...)`. An ERR entry was added to pitfalls.md.

### Step 8: Tag the file

Added as line 1 of governance.md:
```
> SDK version: v9.0.0 | Audit date: 2026-03-21
```

---

## 10. Common Pitfalls When Auditing

### Pitfall A: Using the wrong interface

The SDK has both `IntegrateVoterBatchVoteParams` (internal module) and `BatchVoteParams` (public API for `LockModule.batchVoteTransaction`). They differ significantly:

- `IntegrateVoterBatchVoteParams` uses: `lockIds: SuiObjectId[]`, `poolIds: SuiObjectId[]`, `weights: bigint[]`, `volumes: bigint[]` (parallel arrays)
- `BatchVoteParams` uses: `locks: BatchVoteLock[]`, `votes: BatchVoteItem[]` (array of objects)

**Always grep for the method on the correct module class.** Then follow its declared params type, not a similarly-named internal type.

### Pitfall B: Partial fix — updating the table but not the code example

The code example is what an agent copies verbatim. Updating only the parameter table while leaving the old code example unchanged means the agent still gets the wrong behavior.

**Always update both the parameter table AND the code example.**

### Pitfall C: d.ts says one type, index.js uses another

Sometimes the d.ts type is slightly abstract (e.g., `SuiObjectId` which is just `string`) while index.js uses it as a plain string. This is not an error — document the concrete type from d.ts.

However, if d.ts says `Promise<Transaction>` but index.js returns `[Transaction, string]`, that IS a discrepancy. The runtime behavior from index.js is the ground truth.

### Pitfall D: Overlooking optional parameters

In d.ts, optional parameters have a `?` suffix: `fullNodeUrl?: string`. If the skill file marks an optional field as required (or omits the required/optional column), fix it.

### Pitfall E: Forgetting to check index.js return value

A function may declare `Promise<Transaction>` in d.ts but actually return a tuple in index.js. Always grep index.js for the function body and check what it returns.

---

## 11. Quick Reference: Grep Patterns by Module

### Lock Module

```bash
# Find all Lock module methods
grep -n "batchVoteTransaction\|createLockTransaction\|increaseLockAmount\|increaseDuration\|mergeTransaction\|splitTransaction\|transferTransaction\|disablePermanent\|enablePermanent\|unlockTransaction\|isVoted" dist/index.d.ts

# Find a specific Lock interface
grep -n "interface.*Lock\|interface.*Vote\|interface.*Batch" dist/index.d.ts

# Verify in implementation
grep -n "batchVoteTransaction\|createLockTransaction" dist/index.js
```

### Position Module

```bash
# Find all Position module methods
grep -n "openPositionTransaction\|addLiquidityTransaction\|removeLiquidityTransaction\|closeTransaction\|stakeTransaction\|claimFeesTransaction" dist/index.d.ts

# Find Position interfaces
grep -n "interface.*Position\|interface.*Liquidity\|interface.*OpenPosition" dist/index.d.ts

# Verify in implementation
grep -n "openPositionTransaction\|addLiquidityTransaction" dist/index.js
```

### Pool/Router Module (Swaps)

```bash
# Find swap methods
grep -n "swapTransaction\|swapRouterTransaction\|getSwapRoute\|preSwap" dist/index.d.ts

# Find swap interfaces
grep -n "interface.*Swap\|interface.*Route\|interface.*Pool" dist/index.d.ts

# Verify in implementation
grep -n "swapTransaction\|preSwap" dist/index.js
```

### PortEntry Module (Vaults)

```bash
# Find vault methods
grep -n "createPortEntryTransaction\|PortEntry" dist/index.d.ts

# Find PortEntry interfaces
grep -n "interface.*PortEntry\|interface.*Vault" dist/index.d.ts

# Verify in implementation
grep -n "createPortEntryTransaction" dist/index.js
```

### Root / initFullSailSDK

```bash
# Find init function and options
grep -n "initFullSailSDK\|InitFullSailSDKOptions\|SDKConfigNetwork" dist/index.d.ts

# Verify in implementation
grep -n "initFullSailSDK" dist/index.js
```

---

## 12. Audit Completion Checklist

For each skill file, confirm before marking complete:

- [ ] All functions in the module have been grepped in `dist/index.d.ts`
- [ ] All params interfaces have been fully flattened (no unresolved extends/Omit/Pick)
- [ ] Implementation verified in `dist/index.js` (destructuring matches declared params)
- [ ] Return types verified in `dist/index.js`
- [ ] Skill file parameter table matches flattened SDK truth
- [ ] Code example uses correct parameter names and types
- [ ] ERR entries added to pitfalls.md for any runtime-breaking discrepancies
- [ ] Frontmatter tag added as first line: `> SDK version: vX.Y.Z | Audit date: YYYY-MM-DD`
