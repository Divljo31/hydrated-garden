---
{"dg-publish":true,"permalink":"/wiki/sdk-next/","title":"sdk-next","tags":["sdk","trading","router","typescript","intents","indexer","oracles","uniswap-v3"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"sdk-next","repo":"sdk","paths":["packages/sdk-next/src/factory.ts","packages/sdk-next/src/index.ts","packages/sdk-next/src/pool","packages/sdk-next/src/pool/PoolSync.ts","packages/sdk-next/src/pool/events/EventBus.ts","packages/sdk-next/src/pool/amm","packages/sdk-next/src/pool/amm/uniswapv3","packages/sdk-next/src/sor","packages/sdk-next/src/tx","packages/sdk-next/src/indexer","packages/sdk-next/src/oracle","packages/sdk-next/src/gho","packages/sdk-next/src/aave","packages/sdk-next/src/client/ChainParams.ts","packages/sdk-next/docs"],"key_exports":["createSdkContext","SdkCtx","SdkOptions","TradeRouter","TradeScheduler","TxBuilderFactory","PoolContextProvider","SnapshotPoolCtxProvider","PoolFactory","UniswapV3PoolClient","StakingApi","LiquidityMiningApi","AaveUtils","AaveMarkets","EvmClient","BalanceClient","AssetClient","ChainParams","indexBlocks","BlockFetcher","RpcPool","MmOracleClient","MmRouting","StableSwapPeg"],"tags":["sdk","trading","router","typescript","intents","indexer","oracles","uniswap-v3"],"source_count":1,"last_updated":"2026-11-02"}}
---


# sdk-next

**TL;DR:** `@galacticcouncil/sdk-next` (**v2.5.0**, papi v2) is the trading SDK for [[wiki/hydration\|hydration]]: trade routing ([[wiki/smart-order-router\|smart-order-router]]), pool state, DCA/TWAP scheduling, tx building (incl. ICE intents), staking, liquidity mining, money-market helpers and a block indexer. Peers: [[wiki/sdk-common\|sdk-common]] `>=1.2.0`, [[wiki/sdk-descriptors\|sdk-descriptors]] `>=2.7.0`, `polkadot-api
{ #2}
.1.7`, `viem
{ #2}
.38.3`. **Since Aug 2026:** pools moved to `src/pool/amm/*` and are driven by one event-driven sync engine (`PoolSync`, torn-snapshot bug fixed); new **Uniswap v3** venue; multiple Aave markets; 2-second block time; TWAP **buy** removed.

## Quick Start

```ts
import { api, createSdkContext } from '@galacticcouncil/sdk-next';
import { createClient } from 'polkadot-api';

const client = createClient(api.getWs('wss://hydradx-rpc.dwellir.com'));
const sdk = await createSdkContext(client);              // follows best
const sdkAt = await createSdkContext(client, { at: '0xabc…' }); // pinned
```

`createSdkContext` (`src/factory.ts`) returns `SdkCtx`:

- `api` — `{ aave: AaveUtils, router: TradeRouter, scheduler: TradeScheduler, staking: StakingApi, farm: LiquidityMiningApi }`
- `client` — `{ asset, balance, evm, params: ChainParams }`
- `ctx` — `{ pool: PoolContextProvider }`
- `tx` — `TxBuilderFactory(client, evm, at, aave)`
- `destroy()`

Default venues activated: **Aave, Omnipool, Stableswap, XYK, Uniswap v3** (`.withAave().withOmnipool().withStableswap().withXyk().withV3()`). HSM and LBP exist (`withHsm()` — pulls in Stableswap as a dependency — and `withLbp()`) but are not activated by the factory.

## `at` semantics (changed)

`PoolContextProvider.isSnapshot` is `at !== 'best'`. Only `'best'` (default) subscribes; **`'finalized'` and a block hash are both static, one-shot reads** — no subscriptions, and `getPools()` reloads each call. The sync engine itself always follows best: finality lags too much to price a market.

## Pool sync engine (`PoolSync` + `EventBus`, internal — not exported)

Design doc: `docs/SOR_ENGINE.md` (replaces the deleted `SOR_BLOCK_CONSISTENCY*.md` / `POOL_EVENT_SYNC*.md` series).

| Component | Role |
|---|---|
| `EventBus` (`pool/events/EventBus.ts`) | follows `client.bestBlocks$`, diffs each emission into `{ applied, orphaned, gap }`, reads a block's events pinned via `eventsAt(hash)` — reorgs and gaps come out of the feed, no `parentHash` walking |
| `PoolSync` (`pool/PoolSync.ts`) | process-wide singleton (`PoolSync.shared(client)`); owns the subscription, watchdog, client registration and **topological order** (sources before dependents, e.g. HSM ← Stableswap); dispatches each block to every client's own serial queue |
| `PoolClient` | pool logic + store + consumer API + matched-event ring + own queue; `dependencies()` declares sources |
| `PoolQuery` (`pool/PoolQuery.ts`) | all chain reads as named scopes over `QueryCache` (`utils/QueryCache.ts`) — coalesced, deadline-guarded, per-block |
| `PoolStore` / `PoolLog` | serialized commits / structured logging |

Guarantees (from the doc): **no tearing** (omnipool `reserve` + `hub_reserve`, stableswap reserves + pegs are never half-updated), one coherent point per block, best-block tracking, reorg healing within `REPLAY_DEPTH` by replaying matched events from the orphaned side. Correctness is checked differentially by `probeAmm` (live store vs a fresh pinned reload, every block). Repro of the old bug kept as `chopsticks/src/omni-tear.spec.ts` (now asserts coherence).

Consumer API: `getSubscriber()` — shared, ref-counted, emits **deltas** (only changed pools); `getPools(at?)` — ad-hoc pinned load that never registers with the engine.

`PoolContextProvider` won't serve a partial venue set: `getPools()` returns the map only once every active venue has loaded (`loaded` set); otherwise it does one shared `loadAll(at)` across active clients. A failed venue load never counts as loaded.

## Venues (`src/pool/amm/*`)

| Pool | `PoolType` | Client | Notes |
|------|-----------|--------|-------|
| [[omnipool\|omnipool]] | `Omnipool` | `OmniPoolClient` | `OmniPoolFee` (dynamic fee math) + `omni/utils.ts` (`getEmaPair`, `getEmaKey`) |
| [[wiki/stableswap\|stableswap]] | `Stableswap` | `StableSwapClient` | `StableSwapPeg` recomputes pegs off-chain (`recalculatePegs`, `max_peg_update`) |
| [[wiki/xyk-pools\|XYK]] | `XYK` | `XykPoolClient` | |
| [[wiki/lbp\|LBP]] | `LBP` | `LbpPoolClient` | not default-active |
| HSM | `HSM` | `HsmPoolClient` | depends on Stableswap; GHO facilitator EVM events |
| Aave | `Aave` | `AavePoolClient` | aTokens routable via `AaveTradeExecutor` runtime API; reserves enumerated per market |
| **[[wiki/uniswap-v3\|Uniswap v3]]** | `UniswapV3` | `UniswapV3PoolClient` | see below |

Every client extends `PoolClient<T>` (`pool/PoolClient.ts`) and has a sibling `*Query.ts` (read surface) and `*Math.ts`.

### Uniswap v3 venue (`pool/amm/uniswapv3/`)

- **Curated, not discovered** (`const.ts → V3_POOLS`): aDOT/HOLLAR, atBTC/HOLLAR, aPAXG/HOLLAR, GETH/HOLLAR, GSOL/HOLLAR — all fee 3000 (asset ids 1001, 1006, 1039, 420, 9001 vs 222). Unlisted pools/fee tiers are ignored because v3 pool creation is permissionless.
- Factory address read from chain: **`Parameters.UniswapV3Factory`**; unset → venue reports not configured (`isSupported()` false). A transport failure is not cached as "not deployed".
- Tokens addressed by their **registered ERC-20 contract**, not the asset-id alias (`pool/amm/assetAddress.ts`).
- Own tick math, ported and pinned to the Uniswap reference (`math/{tickMath,swapMath,tickList}.ts`) — no Uniswap SDK dependency. Trades are sized by **active liquidity**, not `balanceOf`. The tick window is sized by price range (`windowWords(tickSpacing)`, ~20k ticks per side, hard cap on bitmap words); `capped` warns when the window is cut short. Exact-out that the pool can't fully fill returns 0 (would revert on chain).

## Smart Order Router

`sdk.api.router` (`TradeRouter`, `sor/TradeRouter.ts`): `getBestSell`, `getSell`, `getSells`, `getBestBuy`, `getBuy`, `getBuys`, `getSpotPrice`, `getMostLiquidRoute`. Algorithm and audit: [[wiki/smart-order-router\|smart-order-router]], `docs/SOR_SPEC.md`, `docs/SOR_AUDIT.md`. Trade fee percentage keeps precision below 0.01%.

## Trade Scheduling (`TradeScheduler`)

- `getDcaOrder`, **`getOpenBudgetDcaOrder`** (new), `getTwapSellOrder`, `getMinimumOrderBudget`, `getTwapTradeCount`, `getTwapExecutionTime`.
- `TradeOrderType = Dca | TwapSell` — **TWAP buy is gone** (matches the node: DCA buy schedules return `NoLongerSupported`).
- Timing is block-time aware: `BLOCK_TIME_TARGET = 2_000` ms (`consts.ts`); `ChainParams.getBlockTime()` derives the real block time from recent blocks (TTL-cached), `getMinimalPeriod()`, `getMinOrderBudget()`. `TWAP_EXECUTION_INTERVAL_MS = 36_000`, `TWAP_MAX_DURATION` 6 h, `TWAP_MAX_PRICE_IMPACT = -5`, `ORDER_MIN_BLOCK_PERIOD = 15`.
- Every `TradeOrder` carries `assetOutEd` (destination existential deposit).

## Transaction Building (`TxBuilderFactory`)

| Method | Builder |
|---|---|
| `trade(trade)` | `TradeTxBuilder` — direct Omnipool op or `Router.sell/buy` |
| `order(order)` | `OrderTxBuilder` — DCA / TWAP schedule |
| `intentMarket(trade)` | `IntentMarketTxBuilder` → `IntentSwap`, min-out from `slippagePct` (default 1) |
| `intentLimit(trade)` | `IntentLimitTxBuilder` → `IntentLimitOrder`, `withMinAmountOut()`, `withPartial()` (default true) |
| `intentOrder(order)` | `IntentOrderTxBuilder` → `IntentDcaSchedule` (`.twap` variant) |

Intent builders target the **ICE** pallets ([[wiki/pallet-intent\|pallet-intent]], [[wiki/pallet-ice\|pallet-ice]]) — now whitelisted in the main `hydration` descriptor. `withBeneficiary(address)` is required; the call is wrapped in `Dispatcher.dispatch_with_extra_gas` when the beneficiary has borrows in a **`tradeable`** Aave market. Docs: `docs/INTENT.md`; examples `test/script/examples/ice/`.

## Aave (`src/aave/`)

- **Multiple markets** (`aave/const.ts → AAVE_MARKETS`): main (pool proxy `0x1b02E051…7B38`), **BIL** (`0x69310FdA…7a53`), **GIGAHDX** (`0x2Ce2CfFF…8923`, `tradeable: false` — its aToken never trades). A reserve in several markets resolves to the first (main leads).
- `AaveMarkets` lists aTokens from each market's reserves (new reserves need no config) and caches per session; `AaveClient`, `AaveUtils`, `AaveMath`, `AaveLog`. If the runtime pools call returns none, reserves are enumerated from the market directly.

## Money-market oracles (`src/oracle/`)

`MmOracleClient` + `MmOracleLog` + `mappings.ts` resolve `Stableswap.PoolPegs` `MMOracle` addresses into prices; **`MmRouting`** (new) routes an oracle-update signal to the `mmAddress` it refreshes (by Managed/hybrid emitter, DIA `key`, or EMA update slot). Inventory: `docs/MM_ORACLES.md` (Managed / DIA wrapper / hybrid aggregator); design `docs/ORACLE_SPEC.md`.

## EVM client

`EvmClient` honours `at`; viem client with `batch: { multicall: true }` and a chain definition pointing **multicall3** at Hydration's MultiView (`0x61f69c81…95d8`, `evm/chain.ts`) — Hydration EVM reads are batched. EVM log parsers: `aave/AaveLog.ts`, `gho/GhoTokenLog.ts`, `oracle/MmOracleLog.ts`.

## Block indexer (`src/indexer/`)

Not on `SdkCtx`; `import { indexer } from '@galacticcouncil/sdk-next'`. `indexBlocks(opts)`, `BlockFetcher` (`chain_getBlock` and/or raw `System.Events` via `SYSTEM_EVENTS_KEY`), `RpcPool`, `Semaphore`, `IndexerStats`, `decodeCompactLength`.

## Staking & Liquidity Mining

- `sdk.api.staking` — `StakingApi` / `StakingClient`; period maths accounts for the 2 s block-time switch.
- `sdk.api.farm` — `LiquidityMiningApi` / `LiquidityMiningClient`; `RewardClaimSimulator` mirrors `math-liquidity-mining` to preview claims.

## Balances

`BalanceClient.getBreakdown()`: transferable = `free - max(frozen - reserved, 0)`.

## Gotchas

- **Stableswap share supply is still read from the wrong storage.** `StableSwapQuery.issuance` reads `Tokens.TotalIssuance(poolId)` and `StableSwapClient` feeds it to all `StableMath` calls; [[wiki/pallet-stableswap\|pallet-stableswap]] uses its own `Stableswap.ShareIssuance` map. Unfixed at d16ff85.
- `'finalized'` is static now (see `at` semantics) — code that relied on it subscribing will silently stop updating.
- v3 quotes are only as wide as the loaded tick window; heed the `capped` warning on large trades.

## Key Types

`Trade`, `Swap`, `TradeOrder` (`Dca | TwapSell`, `assetOutEd`), `BlockAt` (`'best' | 'finalized' | hash`), `PoolType` (`Aave, LBP, Omnipool, Stableswap, XYK, HSM, UniswapV3`), `SnapshotPoolCtx` (for `SnapshotPoolCtxProvider`), `AaveMarket`, `V3PoolConfig`, `RawBlock` / `IndexBlocksOptions`.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
