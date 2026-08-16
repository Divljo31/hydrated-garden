---
{"dg-publish":true,"permalink":"/wiki/dca/","title":"DCA (Dollar-Cost Averaging)","tags":["trading","automation"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"DCA (Dollar-Cost Averaging)","tags":["trading","automation"],"source_count":1,"last_updated":"2026-08-15"}}
---


# DCA (Dollar-Cost Averaging)

**TL;DR:** DCA is a [[wiki/hydration\|hydration]] trading tool that enables automated recurring trades over time using the [[omnipool\|omnipool]]. A related feature, **Split Trade / Easy DCA**, splits large trades into smaller chunks targeting <0.1% slippage per chunk.

## Sell-only since Aug 2026

`schedule` now rejects any `Order::Buy` with `Error::NoLongerSupported` (`pallets/dca/src/lib.rs`) — **only sell schedules can be created**. Buy schedules stored before the restriction keep executing to completion, so the buy execution path in `on_initialize` is still live and must not be treated as dead code. See [[wiki/pallet-dca\|pallet-dca]].

[[wiki/sdk-next\|sdk-next]] `TradeScheduler` now also returns `assetOutEd` on every order DTO (`src/sor/TradeScheduler.ts`) — per-trade `amount_out` must clear the destination asset's existential deposit.

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
