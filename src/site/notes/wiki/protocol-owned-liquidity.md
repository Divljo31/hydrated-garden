---
{"dg-publish":true,"permalink":"/wiki/protocol-owned-liquidity/","title":"Protocol-Owned Liquidity (POL)","tags":["economics","treasury","liquidity","omnipool"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"Protocol-Owned Liquidity (POL)","tags":["economics","treasury","liquidity","omnipool"],"source_count":2,"last_updated":"2026-04-13"}}
---


# Protocol-Owned Liquidity (POL)

**TL;DR:** Protocol-Owned Liquidity (POL) refers to liquidity positions owned by [[wiki/hydration\|hydration]] itself in the [[omnipool\|omnipool]], accumulated when asset prices fall at LP withdrawal and serving as a floor/liquidity of last resort for [[wiki/lrna\|lrna]] value.

## How POL Accumulates

When an LP withdraws from the [[omnipool\|omnipool]] and the asset price has **fallen** since entry, the protocol claims a portion of the LP's shares. These shares become protocol-owned:

```
ΔB = max(-(p₀ - p)/(p + p₀) * Δs, 0)
```

The claimed shares are tracked in `protocol_shares` within the asset's `AssetState`.

## Role as Liquidity of Last Resort

POL is never withdrawn speculatively. It acts as a floor, ensuring a base level of liquidity remains even if all third-party LPs exit. This sets a lower bound on how far [[wiki/lrna\|lrna]]'s value could fall in a mass-withdrawal scenario — one of the three mechanisms counteracting negative LRNA imbalance.

## Treasury and Bonds

The [[wiki/hydration\|hydration]] treasury accumulates and diversifies POL via [[wiki/opengov\|opengov]] governance. DAOs on [[wiki/polkadot\|polkadot]] can also manage their treasuries on Hydration via [[wiki/xcm\|xcm]]. Users can purchase [[wiki/hdx\|hdx]] bonds to grow POL in exchange for fixed-rate yield.

## Sources

- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
