# Full Sail Skills

## What This Is

A comprehensive skill file (`fullsail.md`) that teaches AI agents how to work with Full Sail — a capital-efficient DEX on Sui blockchain. Modeled after the Aftermath Finance skills format, it gives agents the context, patterns, and gotchas needed to execute swaps, manage liquidity, interact with veSAIL governance, use vaults, and build integrations using the SDK — without having to figure things out from scratch.

## Core Value

An agent that reads this skill can correctly interact with any part of Full Sail on the first try — no guessing, no wrong IDs, no missed reward claims.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Covers SDK integration (`@fullsailfinance/sdk`) — installation, initialization, sender setup
- [ ] Covers swap flows — `getSwapRoute`, `preSwap`, `swapTransaction`, `swapRouterTransaction`
- [ ] Covers position management — open, add liquidity, stake, remove liquidity, close
- [ ] Covers reward claim patterns — fees, oSAIL, pool rewards, combined claims; staked vs unstaked rules
- [ ] Covers lock/veSAIL lifecycle — create from SAIL or oSAIL, increase, merge, split, transfer, permanent lock
- [ ] Covers governance — epoch cycle, batch voting with weight distribution, volume predictions
- [ ] Covers vaults — deposit/withdraw, yield mechanics
- [ ] Covers prediction voting — creating and voting on predictions
- [ ] Surfaces gotchas and common mistakes (wrong ID types, oSAIL expiry, reward claim preconditions)
- [ ] Production-quality markdown — formatted, structured, ready to publish publicly
- [ ] Compatible with Claude, Cursor, Windsurf, Claude Code, and other AI agents

### Out of Scope

- Legal/privacy content — not relevant to agent skill
- Brand guide — not actionable for agents
- Tutorial walkthroughs for human users — skill is agent-facing, not human-facing

## Context

- Full Sail is a DEX on Sui blockchain targeting capital efficiency over token emission cycles
- Key tokens: SAIL (native), veSAIL (vote-escrowed governance), oSAIL (weekly emission, expires 5 weeks after epoch, redeemable for SAIL/USDC or lockable as veSAIL)
- SDK: `@fullsailfinance/sdk` — methods ending in "Transaction" return unsigned transactions
- Pool types: Chain Pool (real-time, from `getByIdFromChain`) vs Backend Pool (calculated, from `getById`)
- oSAIL is auto-claimed in liquidity operations; fees and pool rewards require separate claims
- Lock voting resets on merge/split operations
- Reference format: Aftermath Finance skills (https://github.com/AftermathFinance/skills)
- Aftermath's structure: SKILL.md overview + specialized sub-documents; Full Sail will be a single comprehensive file

## Constraints

- **Format**: Markdown, self-contained, modeled after Aftermath API skill format
- **Scope**: Single file (`fullsail.md`) covering all domains — trading, liquidity, governance, SDK
- **Quality bar**: Production-ready for public publishing, not a draft

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Single file vs multi-file | User wants one comprehensive file — simpler to drop into any agent | — Pending |
| Model after Aftermath format | Established reference the user explicitly cited | — Pending |

---
*Last updated: 2026-03-05 after initialization*
