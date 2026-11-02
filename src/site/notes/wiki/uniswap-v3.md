---
{"dg-publish":true,"permalink":"/wiki/uniswap-v3/","title":"Uniswap v3 on Hydration EVM","tags":["amm","uniswap-v3","evm","hollar","routing"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"Uniswap v3 on Hydration EVM","repo":"sdk","paths":["packages/sdk-next/src/pool/amm/uniswapv3/const.ts","packages/sdk-next/src/pool/amm/uniswapv3/UniswapV3PoolClient.ts","packages/sdk-next/src/pool/amm/uniswapv3/UniswapV3Query.ts","packages/sdk-next/src/pool/amm/uniswapv3/math","packages/sdk-next/src/pool/amm/assetAddress.ts"],"symbols":["UniswapV3PoolClient","UniswapV3Pool","UniswapV3Math","V3_POOLS","V3PoolConfig","windowWords","ticksInWord","PoolType.V3"],"tags":["amm","uniswap-v3","evm","hollar","routing"],"last_updated":"2026-11-02"}}
---


# Uniswap v3 on Hydration EVM

**TL;DR:** A Uniswap v3 deployment on Hydration's EVM is a routable venue in [[wiki/sdk-next\|sdk-next]] (`PoolType.V3 = 'UniswapV3'`, activated by default via `withV3()`). The factory address is read from chain storage **`Parameters.UniswapV3Factory`**; the pools the router uses are a **curated** list of HOLLAR pairs, because v3 pool creation is permissionless.

## Curated pools (`const.ts → V3_POOLS`, fee 3000 = 0.3%)

| Pair | Asset ids |
|---|---|
| aDOT / [[wiki/hollar\|HOLLAR]] | 1001 / 222 |
| atBTC / HOLLAR | 1006 / 222 |
| aPAXG / HOLLAR | 1039 / 222 |
| GETH / HOLLAR | 420 / 222 |
| GSOL / HOLLAR | 9001 / 222 |

Pools at unlisted fee tiers, and pools the factory has no deployment for, are skipped.

## How the SDK quotes it

- Tokens are addressed by their **registered ERC-20 contract** (`pool/amm/assetAddress.ts`), not the asset-id precompile alias.
- State read over EVM (batched via multicall); quoting is client-side with tick math **ported and pinned to the Uniswap reference** (`math/{tickMath,swapMath,tickList}.ts`) — no Uniswap SDK dependency.
- Depth = active liquidity `L` (not `balanceOf`). Tick window per side sized by price range (`windowWords(tickSpacing)`, ~20k ticks ≈ 7.4× price move, capped words); a `capped` warning flags a window cut short — quotes beyond it over-estimate.
- Exact-out the pool can't fully fill returns 0. Quotes are allowed when the price is outside the band but the trade moves back in.
- Factory unset → `isSupported()` false and the venue is silently absent; a transport failure is *not* cached as "not deployed".

## Routing

Counts as a **trusted** pool in the [[wiki/smart-order-router\|smart-order-router]] (everything except XYK is trusted).

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
- [[wiki/sdk-next\|sdk-next]]
