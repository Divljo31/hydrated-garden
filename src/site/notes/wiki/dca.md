---
{"dg-publish":true,"permalink":"/wiki/dca/","title":"DCA (Dollar-Cost Averaging)","tags":["trading","automation"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"DCA (Dollar-Cost Averaging)","tags":["trading","automation"],"source_count":1,"last_updated":"2026-11-02"}}
---


# DCA (Dollar-Cost Averaging)

**TL;DR:** DCA is a [[wiki/hydration\|hydration]] trading tool that enables automated recurring trades over time through the router; new DCA flow is ICE DCA intents ([[wiki/ice\|ice]]). A related feature, **Split Trade / Easy DCA**, splits large trades into smaller chunks targeting <0.1% slippage per chunk.

## Two implementations

| | Legacy schedules | ICE DCA intents |
|---|---|---|
| Pallet | [[wiki/pallet-dca\|pallet-dca]] (index 66) | [[wiki/pallet-intent\|pallet-intent]] + [[wiki/pallet-ice\|pallet-ice]] |
| Execution | `on_initialize` per schedule via the router | Solver batch; may be matched against opposite intents before AMMs |
| Price protection | stability threshold + slippage vs oracle | `max(hard floor, route-oracle price − slippage)`, `MaxDcaSlippage` 50% |
| Orders | sell only (`Order::Buy` → `NoLongerSupported`; stored buys still execute) | sell only |
| Status | migration source: with `DCA.set_migration_enabled(true)` each due schedule converts into an ICE DCA intent or is cancelled + refunded | target |

Periods are in blocks (2 s); legacy `MinimalPeriod` = 15 blocks.

[[wiki/sdk-next\|sdk-next]] `TradeScheduler` returns `assetOutEd` on every order DTO (`src/sor/TradeScheduler.ts`) — per-trade `amount_out` must clear the destination asset's existential deposit.

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
