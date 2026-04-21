---
{"dg-publish":true,"permalink":"/wiki/nft-lp-positions/","title":"NFT LP Positions","tags":["liquidity","omnipool","nft"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"NFT LP Positions","tags":["liquidity","omnipool","nft"],"source_count":1,"last_updated":"2026-04-13"}}
---


# NFT LP Positions

**TL;DR:** In the [[omnipool\|omnipool]], LP positions are represented as NFTs rather than fungible tokens, enabling precise per-LP accounting and partial withdrawals with entry-price tracking for [[wiki/impermanent-loss\|IL]] calculations.

## Data Model

Each position is stored in the `Positions` storage map, keyed by `PositionItemId`. A `Position<Balance, AssetId>` contains:
- `shares` — number of pool shares owned
- `price` — asset price at time of entry (`p₀`), used for [[wiki/impermanent-loss\|IL]] calculation and withdrawal mechanics
- `asset_id` — which asset the position belongs to

## Why NFTs

The NFT model allows:
- **Per-LP price tracking** — each position remembers its entry price, enabling precise IL calculations
- **Partial withdrawals** — LPs can withdraw a portion of their position
- **Non-fungibility** — two positions in the same asset at different entry prices are distinct, avoiding the averaging problem of fungible share tokens

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
