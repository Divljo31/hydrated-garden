---
{"dg-publish":true,"permalink":"/wiki/smart-order-router/","title":"Smart Order Router (SOR)","tags":["trading","routing","algorithm","sdk"],"dg-note-properties":{"type":"concept","title":"Smart Order Router (SOR)","tags":["trading","routing","algorithm","sdk"],"source_count":1,"last_updated":"2026-04-13"}}
---


# Smart Order Router (SOR)

**TL;DR:** The Smart Order Router is the core routing engine in [[wiki/sdk-next\|sdk-next]], using BFS to find optimal multi-hop trade paths across all [[wiki/hydration\|hydration]] pool types with cycle prevention and trusted/isolated pool classification.

## Algorithm

The SOR uses **Breadth-First Search (BFS)** with graph-based pathfinding:
- **Nodes** = assets
- **Edges** = pool pairs connecting assets
- **Maximum path length:** 10 hops (prevents infinite loops)
- Prevents cycles by tracking both asset revisits and pool reuse

## Routing Strategy

The router classifies pools into two categories:
- **Trusted pools** (deep liquidity): [[omnipool\|omnipool]], [[wiki/stableswap\|stableswap]], [[wiki/lbp\|lbp]], Aave, HSM
- **Isolated pools** (used as bridges): [[wiki/xyk-pools\|XYK]] only

Three routing modes based on where the assets live:
1. **Both in trusted pools** → search trusted pools only (faster, deeper liquidity)
2. **Both in isolated pools** → search relevant isolated pools only
3. **Mixed** → search across both trusted and isolated pools

## Route Selection

**For sells:** The router enumerates all valid routes and selects the one with the **highest output amount**.

**For buys:** Routes are evaluated in reverse order, selecting the one with the **lowest input amount required**.

## Most Liquid Route (MLR) Caching

The SOR caches the most liquid route per asset pair to avoid redundant calculations. MLR is determined based on 0.1% of the highest available liquidity. Cache key: `${assetIn}->${assetOut}::${poolCount}`.

## Trade Impact Calculation

```
tradeFeePct = (zeroFeeOutput - actualOutput) / zeroFeeOutput * 100
priceImpactPct = (spotPriceAmount - zeroFeeOutput) / spotPriceAmount * 100
```

## Rust Implementation

The [[wiki/route-suggester\|route-suggester]] Rust crate provides a high-performance alternative implementation of route discovery, also using BFS with the same trusted/isolated pool strategy. It's designed for runtime integration where performance is critical.

## Pool Filtering

The router supports `.withFilter()` to exclude or whitelist specific pool types — useful for limiting routes to specific pool kinds.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
