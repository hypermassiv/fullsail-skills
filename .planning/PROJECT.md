# Full Sail Skills

## What This Is

A comprehensive skill file (`fullsail.md`) that teaches AI agents how to work with Full Sail — a capital-efficient DEX on Sui blockchain. Modeled after the Aftermath Finance skills format, it gives agents the context, patterns, and gotchas needed to execute swaps, manage liquidity, interact with veSAIL governance, use vaults, and build integrations using the SDK — without having to figure things out from scratch.

## Core Value

An agent that reads this skill can correctly interact with any part of Full Sail on the first try — no guessing, no wrong IDs, no missed reward claims.

## Requirements

### Validated

- ✓ Covers SDK integration (`@fullsailfinance/sdk`) — installation, initialization, sender setup — v1.0
- ✓ Covers swap flows — `getSwapRoute`, `preSwap`, `swapTransaction`, `swapRouterTransaction` — v1.0
- ✓ Covers position management — open, add liquidity, stake, remove liquidity, close — v1.0
- ✓ Covers reward claim patterns — fees, oSAIL, pool rewards, combined claims; staked vs unstaked rules — v1.0
- ✓ Covers lock/veSAIL lifecycle — create from SAIL or oSAIL, increase, merge, split, transfer, permanent lock — v1.0
- ✓ Covers governance — epoch cycle, batch voting with weight distribution, volume predictions — v1.0
- ✓ Covers vaults — deposit/withdraw, yield mechanics — v1.0
- ✓ Covers prediction voting — creating and voting on predictions — v1.0
- ✓ Surfaces gotchas and common mistakes (wrong ID types, oSAIL expiry, reward claim preconditions) — v1.0
- ✓ Production-quality markdown — formatted, structured, ready to publish publicly — v1.0
- ✓ Compatible with Claude, Cursor, Windsurf, Claude Code, and other AI agents — v1.0

### Active

<!-- v1.1: Doc accuracy fixes (sourced from GitHub docs comparison 2026-03-10) -->
- [ ] **FIX-01**: Add 4-year/permanent-only restriction for expired oSAIL locks
- [ ] **FIX-02**: Mark SAIL+USDC redemption (Option B) as deprecated
- [ ] **FIX-03**: Unify SDK initialization pattern (ERR-06 uses `new FullSailSDK()` vs `initFullSailSDK()` elsewhere)
- [ ] **FIX-04**: Fix ERR-02 field names (`currentOSail.coinType` → `.address`, verify `.expiry` field name)
- [ ] **FIX-05**: Remove or verify the 5% insurance fund claim (absent from GitHub docs)
- [ ] **FIX-06**: Fix passive fee voting requirement — passive fees do NOT require voting participation
- [ ] **FIX-07**: Resolve stakeTransaction auto-claim contradiction (section says fees auto-claimed; matrix says NO)

<!-- v1.1: Missing information gaps (from GitHub docs) -->
- [ ] **GAP-01**: Add oSAIL conversion rates table (6-month: 0.5625, 2-year: 0.75, 4-year: 1.0)
- [ ] **GAP-02**: Add voting power formula (`power = locked_amount × duration_ratio`) with examples
- [ ] **GAP-03**: Add 2-week trading fee reward delay to Governance/Rewards sections
- [ ] **GAP-04**: Add exercise fee immediate availability note (contrast with 2-week delay)
- [ ] **GAP-05**: Add managed lock passive fee restriction (`escrow_type.is_locked()` exclusion)
- [ ] **GAP-06**: Add `is_valid_o_sail_type` validation method documentation

<!-- Carried from v1.0 — deferred to v1.2 -->
- [ ] **ADV-01**: API staleness detection script (Python hash comparison, Aftermath-style)
- [ ] **ADV-02**: Separate `fullsail-advanced.md` for power-user patterns
- [ ] **ADV-03**: Cursor/Windsurf-specific integration notes (file size limits, consumption patterns)
- [ ] SDK method signatures live-verified against npm (`@fullsailfinance/sdk`)
- [ ] Code examples testnet-validated
- [ ] Vault SDK namespace confirmed (currently UNVERIFIED in fullsail.md)

### Out of Scope

- Legal/privacy content — not relevant to agent skill
- Brand guide — not actionable for agents
- Tutorial walkthroughs for human users — skill is agent-facing, not human-facing

## Current Milestone: v1.1 Doc Accuracy

**Goal:** Fix all inconsistencies between `fullsail.md` and the GitHub protocol docs, and add the missing protocol mechanics that agents need.

**Target features:**
- 7 accuracy fixes (expired oSAIL restriction, deprecated option B, SDK init form, ERR-02 fields, fee split, passive fee rule, stakeTransaction claim)
- 6 information additions (conversion rates, voting power formula, reward delays, managed lock restriction, oSAIL type validation)

---

## Context

Shipped v1.0 with 1,980 lines of Markdown in `fullsail.md`.
Tech stack: Markdown, `@fullsailfinance/sdk`, `@mysten/sui`, Sui blockchain.
All 53 v1 requirements shipped across 6 phases (12 plans, 5 days).

Known open items for v1.1:
- `@fullsailfinance/sdk` method signatures not live-verified against npm
- Vault SDK namespace absent from public docs — code marked UNVERIFIED
- rewardCoinTypes field name in Pool.getById() return shape not confirmed
- testnet RPC endpoint not confirmed; code examples need validation before publication

## Constraints

- **Format**: Markdown, self-contained, modeled after Aftermath API skill format
- **Scope**: Single file (`fullsail.md`) covering all domains — trading, liquidity, governance, SDK
- **Quality bar**: Production-ready for public publishing, not a draft

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Single file vs multi-file | User wants one comprehensive file — simpler to drop into any agent | ✓ Good — self-contained and complete |
| Model after Aftermath format | Established reference the user explicitly cited | ✓ Good — structure proven |
| Build order: Swaps first as template | Swaps established section format density used by all later phases | ✓ Good — consistent section structure throughout |
| All section anchors locked in Phase 1 | Prevents heading renames breaking downstream cross-references | ✓ Good — no anchor conflicts across phases |
| WRONG/CORRECT labelled code blocks | Agents see failure mode before correct pattern — higher retention | ✓ Good — used consistently across 6+ pitfall entries |
| PRECONDITION + BLOCKING WARNING blockquotes | High-risk actions need visual interruption before code examples | ✓ Good — unstake precondition and merge/split warnings prominent |
| Vault code marked UNVERIFIED | Vault SDK namespace absent from public docs — honesty prevents hallucination | ✓ Good — agents warned of uncertainty |
| Fast Routing deferred to final phase | Can only write accurate anchors after all sections exist | ✓ Good — all 16 links resolve correctly |
| oSAIL coin type epoch-dynamic | Cannot hardcode address — agents must retrieve at runtime | ✓ Good — prevents stale address bugs |

---
*Last updated: 2026-03-10 after v1.1 milestone started — inconsistency analysis vs GitHub docs*
