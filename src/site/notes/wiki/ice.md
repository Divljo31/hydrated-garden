---
{"dg-publish":true,"permalink":"/wiki/ice/","title":"ICE (Intent Composing Engine)","tags":["trading","intents","cross-chain","upcoming"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"ICE (Intent Composing Engine)","tags":["trading","intents","cross-chain","upcoming"],"source_count":1,"last_updated":"2026-08-15"}}
---


# ICE (Intent Composing Engine)

**TL;DR:** ICE (Intent Composing Engine) is [[wiki/hydration\|hydration]]'s intent-based trading model — users express desired outcomes and solvers find the execution. **Still not in the production runtime**, but as of Aug 2026 there is real code: intent tx builders in [[wiki/sdk-next\|sdk-next]] running against a separate `hydrationIce` descriptor.

## Status

The `ICE` / `Intent` pallets are **not** in the Hydration production runtime. [[wiki/sdk-descriptors\|sdk-descriptors]] (v2.6.0) generates the `hydrationIce` chain entry from a checked-in runtime blob (`packages/descriptors/wasm/ice/ice.wasm`, `.papi/metadata/hydrationIce.scale`) instead of a live chain — that package must be built before the builders can be used. `Papi` exposes the second typed API as `apiIce` (`packages/sdk-next/src/api/Papi.ts`).

## SDK surface

[[wiki/sdk-next\|sdk-next]] v1.6.0, `packages/sdk-next/src/tx/` (docs: `packages/sdk-next/docs/INTENT.md`, examples: `test/script/examples/ice/`):

| Entry point | Builder | Call |
|---|---|---|
| `sdk.tx.intentMarket(trade)` | `IntentMarketTxBuilder` | `IntentSwap` |
| `sdk.tx.intentLimit(trade)` | `IntentLimitTxBuilder` | `IntentLimitOrder` |
| `sdk.tx.intentOrder(order)` | `IntentOrderTxBuilder` | `IntentDcaSchedule` / `IntentDcaSchedule.twap` |

`withBeneficiary(address)` is required — it drives an Aave debt check and auto-wraps the call in `Dispatcher.dispatch_with_extra_gas` for accounts with borrow positions. Slippage is encoded on chain as `slippagePct * 10_000`.

Cross-chain intents are a separate stack today: [[wiki/xc-swap\|xc-swap]] (NEAR Intent Routing), not ICE.

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
