---
{"dg-publish":true,"permalink":"/wiki/price-barrier/","title":"Price Barrier","tags":["security","risk","omnipool","oracle"],"dg-note-properties":{"type":"concept","title":"Price Barrier","tags":["security","risk","omnipool","oracle"],"source_count":1,"last_updated":"2026-04-13"}}
---


# Price Barrier

**TL;DR:** The Price Barrier is a risk control on [[omnipool\|omnipool]] liquidity operations that compares spot price against [[wiki/ema-oracle\|ema-oracle]] price, pausing operations when deviation exceeds the 1% threshold to prevent manipulation.

## Mechanism

If the deviation between spot price and oracle price exceeds the configured threshold (currently **1%**), the liquidity operation is **paused** for that asset. This prevents manipulation of LP entry/exit conditions — an attacker cannot move the spot price with a trade and then immediately add or remove liquidity at the manipulated price.

## Configuration

The threshold is set via the `PriceBarrier` config trait in the [[omnipool\|omnipool]] pallet. It is governance-controllable by [[wiki/hdx\|hdx]] holders via [[wiki/opengov\|opengov]].

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
