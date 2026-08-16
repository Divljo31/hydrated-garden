---
{"dg-publish":true,"permalink":"/wiki/circuit-breaker/","title":"Circuit Breaker","tags":["security","risk","omnipool","cross-chain"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"Circuit Breaker","tags":["security","risk","omnipool","cross-chain"],"source_count":1,"last_updated":"2026-08-15"}}
---


# Circuit Breaker

**TL;DR:** The circuit breaker (`pallet-circuit-breaker`) is a dedicated [[wiki/hydration\|hydration]] pallet that enforces per-block trade volume limits in the [[omnipool\|omnipool]], plus cross-chain deposit / global-withdraw limits, protecting against flash-loan style attacks and rapid liquidity drainage.

## Rules

- No more than **50% of an asset's liquidity** can be traded in a single block
- Per-block limits on liquidity additions and removals are also tracked
- Hub asset ([[wiki/lrna\|lrna]]) is excluded from circuit breaker tracking (it is internal)
- Trades exceeding the block limit must be split across multiple blocks

## Storage

The pallet tracks:
- `AllowedTradeVolumeLimitPerAsset`
- `AllowedAddLiquidityAmountPerAsset`
- `AllowedRemoveLiquidityAmountPerAsset`
- `AssetLockdownState`, `WithdrawLimitAccumulator`, `WithdrawLockdownUntil` (cross-chain deposit / withdraw side)

## Trade context (Aug 2026)

`Config::InTradeContext: Get<bool>` gates the `DepositLockWhitelist` exemption: a whitelisted account (the router account) **errors instead of locking** the deposit, which is only safe while a trade is in flight and the error unwinds it (`do_lock_deposit`, `pallets/circuit-breaker/src/lib.rs`).

The runtime wires it to `pallet_broadcast::get_swapper().is_some()` (`runtime/hydradx/src/assets.rs`), and [[wiki/pallet-route-executor\|pallet-route-executor]] now sets the swapper **before** the first transfer of user funds — so the trade-context window spans the whole router execution, not just the individual pool swap.

## Surfaced in the UI

Cross-chain limits are shown at transfer time by [[wiki/hydration-ui-modules\|hydration-ui-modules]] (`CBreakerInbound/OutboundLimitInfo`, alert hooks) via `api/xcm.ts` → `useCrossChainDepositLimit`, `useCrossChainGlobalWithdrawLimit` ([[wiki/hydration-ui-api\|hydration-ui-api]]).

## Governance

Circuit breaker volume limits are governed on-chain by [[wiki/hdx\|hdx]] holders via [[wiki/opengov\|opengov]].

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]]
