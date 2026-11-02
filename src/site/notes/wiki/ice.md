---
{"dg-publish":true,"permalink":"/wiki/ice/","title":"ICE (Intent Composing Engine)","tags":["trading","intents","solver","dca","ice"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"ICE (Intent Composing Engine)","tags":["trading","intents","solver","dca","ice"],"source_count":2,"last_updated":"2026-11-02"}}
---


# ICE (Intent Composing Engine)

**TL;DR:** [[wiki/hydration\|hydration]]'s intent-based trading, **live in the production runtime** (spec 458): users post intents (swap, partial swap, match-first swap, DCA) to [[wiki/pallet-intent\|pallet-intent]]; a solver run natively by block authors matches opposite intents and routes the remainder across Omnipool / Stableswap / XYK / Aave / Uniswap v3; [[wiki/pallet-ice\|pallet-ice]] settles the batch; [[wiki/pallet-lazy-executor\|pallet-lazy-executor]] runs optional EVM post-resolution forwards. Legacy [[wiki/pallet-dca\|pallet-dca]] schedules migrate into ICE DCA intents.

## Pieces

| Layer | Where | Page |
|---|---|---|
| Intent storage / lifecycle | `pallets/intent` (index 98) | [[wiki/pallet-intent\|pallet-intent]] |
| Settlement, fee, solver mode, venue routing | `pallets/ice` (index 99) | [[wiki/pallet-ice\|pallet-ice]] |
| EVM forwards after resolution | `pallets/lazy-executor` (index 97) | [[wiki/pallet-lazy-executor\|pallet-lazy-executor]] |
| Shared types (`IntentData`, `Solution`, `SolverMode`, `RoutingTarget`) | `pallets/ice/support` | [[wiki/pallet-ice\|pallet-ice]] |
| AMM simulators (Omnipool, Stableswap, Aave, Uniswap v3, XYK) | `pallets/ice/amm-simulator`, `runtime/hydradx/src/ice_simulator_provider.rs` | [[wiki/pallet-ice\|pallet-ice]] |
| Solver (v4 + passthrough), route discovery, bench/fuzz | `ice/ice-solver`, `ice/route-findr`, `ice/solver-bench` | [[wiki/pallet-ice\|pallet-ice]] |
| Node worker (solve per best block, submit unsigned) | `node/src/ice_solver_worker.rs` | [[wiki/pallet-ice\|pallet-ice]] |
| SDK builders | `sdk-next/src/tx/Intent{Market,Limit,Order}TxBuilder.ts`, docs `packages/sdk-next/docs/INTENT.md` | [[wiki/sdk-next\|sdk-next]] |

## Economics

- Matched (intent-to-intent) volume pays the ICE protocol fee (runtime default 0.02%, `set_protocol_fee`); AMM-routed volume pays only pool fees.
- Users receive at least their limit; DCA intents are bounded by `max(hard floor, oracle − slippage)` with the oracle taken along the actual route.

## Controls (TC majority)

`ICE.set_solver_mode(Disabled)` = kill switch (new intents refused); `Passthrough` = per-intent router execution at stored limits (emergency drain); `ICE.update_routing` = include/exclude venues (Omnipool asset, stablepool, XYK pair, Aave wrap, Uniswap v3 pool). See [[wiki/note-emergency-levers\|note-emergency-levers]].

## Related

Cross-chain intent corridors are a separate stack: [[wiki/xc-swap\|xc-swap]] and [[wiki/whm-intents\|whm-intents]].

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
