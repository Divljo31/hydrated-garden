---
{"dg-publish":true,"permalink":"/wiki/ema-oracle/","title":"EMA Oracle","tags":["oracle","pricing","security","omnipool"],"dg-note-properties":{"type":"concept","title":"EMA Oracle","tags":["oracle","pricing","security","omnipool"],"source_count":1,"last_updated":"2026-04-13"}}
---


# EMA Oracle

**TL;DR:** The [[omnipool\|omnipool]] maintains an on-chain exponential moving average (EMA) oracle for each asset, providing time-weighted average price data over a configurable window (e.g., 10 blocks) to resist single-block manipulation.

## Usage

The EMA oracle is a critical input to two risk controls:
- **[[wiki/price-barrier\|price-barrier]]** — compares spot price against EMA price to gate liquidity operations
- **[[wiki/dynamic-fees\|Dynamic withdrawal fee]]** — calculates the fee as the percentage difference between spot and oracle price

By using a time-weighted average rather than the instantaneous spot price, the oracle resists single-block manipulation. An attacker would need to sustain a manipulated price over multiple blocks to influence the oracle, which is far more expensive.

## Implementation

Exposed via the `ExternalPriceOracle` config trait in the [[omnipool\|omnipool]] pallet.

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
