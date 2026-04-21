---
{"dg-publish":true,"permalink":"/wiki/circuit-breaker/","title":"Circuit Breaker","tags":["security","risk","omnipool"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"Circuit Breaker","tags":["security","risk","omnipool"],"source_count":1,"last_updated":"2026-04-13"}}
---


# Circuit Breaker

**TL;DR:** The circuit breaker (`pallet-circuit-breaker`) is a dedicated [[wiki/hydration\|hydration]] pallet that enforces per-block trade volume limits in the [[omnipool\|omnipool]], protecting against flash-loan style attacks and rapid liquidity drainage.

## Rules

- No more than **50% of an asset's liquidity** can be traded in a single block
- Per-block limits on liquidity additions and removals are also tracked
- Hub asset ([[wiki/lrna\|lrna]]) is excluded from circuit breaker tracking (it is internal)
- Trades exceeding the block limit must be split across multiple blocks

## Storage

The pallet tracks:
- `AllowedTradeVolumeLimitPerAsset`
- `AllowedAddLiquidityLimitPerAsset`
- `AllowedRemoveLiquidityLimitPerAsset`

## Governance

Circuit breaker volume limits are governed on-chain by [[wiki/hdx\|hdx]] holders via [[wiki/opengov\|opengov]].

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
