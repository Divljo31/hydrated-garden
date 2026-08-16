---
{"dg-publish":true,"permalink":"/wiki/otc-trading/","title":"OTC Trading","tags":["trading","peer-to-peer"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"OTC Trading","tags":["trading","peer-to-peer"],"source_count":1,"last_updated":"2026-08-15"}}
---


# OTC Trading

**TL;DR:** OTC (Over-the-Counter) trading on [[wiki/hydration\|hydration]] enables peer-to-peer trades at user-defined prices with no slippage, with [[wiki/pallet-otc-settlements\|pallet-otc-settlements]] aligning order prices to [[omnipool\|omnipool]] spot prices via arbitrage.

## Aug 2026

- **`fill_order_with_deferred_delivery`** ([[wiki/pallet-otc\|pallet-otc]]) — the filler receives the maker's reserved `asset_out` *first*, a `deliver` closure sources `asset_in` from it (e.g. by routing through a pool), then `amount_in` is pulled to the maker. Whole thing is `#[require_transactional]`; a shortfall rolls the fill back.
- **otc-settlements arbitrage is now self-funded** — the mint/burn of the intermediate asset was removed.
- OTC is now a **product surface, not just a trading tool**: [[wiki/hollar\|hollar]] stable bonds are sold through fixed OTC offers (ids `1488` / `1489`) from the UI's `strategies/` module ([[wiki/hydration-ui-modules\|hydration-ui-modules]]).

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
