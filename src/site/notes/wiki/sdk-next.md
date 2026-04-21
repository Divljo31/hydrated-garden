---
{"dg-publish":true,"permalink":"/wiki/sdk-next/","title":"sdk-next","tags":["sdk","trading","router","typescript"],"dg-note-properties":{"type":"entity","entity_kind":"product","title":"sdk-next","tags":["sdk","trading","router","typescript"],"source_count":1,"last_updated":"2026-04-20"}}
---


# sdk-next

**TL;DR:** `@galacticcouncil/sdk-next` is the primary trading SDK for [[wiki/hydration\|hydration]], providing trade routing, pool queries, [[wiki/smart-order-router\|smart-order-router]], DCA/TWAP scheduling, and transaction building. Built on [[wiki/papi\|papi]] (polkadot-api) for type-safe runtime access via [[wiki/sdk-descriptors\|sdk-descriptors]].

## Quick Start

```ts
import { api, createSdkContext } from '@galacticcouncil/sdk-next';
import { createClient } from 'polkadot-api';

const provider = api.getWs('wss://hydradx-rpc.dwellir.com');
const client = createClient(provider);
const sdk = await createSdkContext(client);
```

The factory returns a `SdkCtx` with: `api` (trading APIs), `client` (asset/balance queries), `ctx` (pool context), `tx` (transaction building), `aave` (lending integration).

## Smart Order Router

The [[wiki/smart-order-router\|smart-order-router]] finds optimal trade paths across all [[wiki/hydration\|hydration]] pool types. See the dedicated page for details on the BFS algorithm and routing strategy. [[wiki/hydration-ui\|hydration-ui]] uses SDK Next for all trade estimation and routing.

Key methods on `sdk.api.router`:
- `getBestSell(assetIn, assetOut, amount)` — find best sell route by highest output
- `getBestBuy(assetIn, assetOut, amount)` — find best buy route by lowest input
- `getSpotPrice(assetIn, assetOut)` — price for 1 unit via most liquid route
- `getRoutes(assetIn, assetOut)` — enumerate all valid routes

## Pool Types Supported

Six pool implementations in `src/pool/`:

| Pool | Package | Description |
|------|---------|-------------|
| [[omnipool\|omnipool]] | `math-omnipool` | Hub-based AMM, routes through [[wiki/lrna\|lrna]] |
| [[wiki/xyk-pools\|XYK]] | `math-xyk` | Classic constant product (`x*y=k`) |
| [[wiki/stableswap\|stableswap]] | `math-stableswap` | Curve-style stable asset trading with amplification parameter |
| [[wiki/lbp\|LBP]] | `math-lbp` | Weighted pools for token launches |
| HSM | `math-hsm` | Isolated multi-pool (depends on Stableswap) |
| Aave | — | [[wiki/hydration-borrow\|hydration-borrow]] supply/withdraw via viem EVM calls |

All pools implement a common interface: `validatePair()`, `parsePair()`, `validateAndSell()`, `validateAndBuy()`, `calculateInGivenOut()`, `calculateOutGivenIn()`, spot price functions.

## Trade Scheduling

`TradeScheduler` provides automated order splitting:

- **[[wiki/dca\|DCA]]** — splits order into multiple trades targeting 0.1% price impact each
- **TWAP** — time-weighted execution over fixed intervals (6 blocks per interval, max 6 hours)
- Validates minimum budgets and maximum price impact (-5% threshold)

## Transaction Building

Fluent API via `TxBuilderFactory`:
```ts
const tx = await sdk.tx.trade(trade)
  .withSlippage(5)    // 5% slippage tolerance
  .build();
```

Automatically selects direct [[omnipool\|omnipool]] operations vs. multi-hop `Router.sell()`/`Router.buy()` depending on route complexity.

## Key Types

- `Trade` — result of a route calculation: amountIn/Out, spotPrice, fees, priceImpact, swaps[]
- `Swap` — single hop: pool type, poolAddress, assetIn/Out, amounts, fees
- `TradeOrder` — scheduled order: DCA or TWAP with tradeCount, tradePeriod, route

## Dependencies

Consumes [[wiki/sdk-common\|sdk-common]] (peer), [[wiki/sdk-descriptors\|sdk-descriptors]] (peer), and all 8 math-* WASM packages.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
