---
{"dg-publish":true,"permalink":"/wiki/sdk-next/","title":"sdk-next","tags":["sdk","trading","router","typescript","intents","indexer","oracles"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"sdk-next","repo":"sdk","paths":["packages/sdk-next/src/factory.ts","packages/sdk-next/src/index.ts","packages/sdk-next/src/pool","packages/sdk-next/src/sor","packages/sdk-next/src/tx","packages/sdk-next/src/indexer","packages/sdk-next/src/oracle","packages/sdk-next/src/gho","packages/sdk-next/src/aave","packages/sdk-next/docs"],"key_exports":["createSdkContext","SdkCtx","SdkOptions","TradeRouter","TradeScheduler","TxBuilderFactory","PoolContextProvider","SnapshotPoolCtxProvider","StakingApi","LiquidityMiningApi","AaveUtils","EvmClient","BalanceClient","AssetClient","indexBlocks","BlockFetcher","RpcPool","MmOracleClient","StableSwapPeg"],"tags":["sdk","trading","router","typescript","intents","indexer","oracles"],"source_count":1,"last_updated":"2026-08-15"}}
---


# sdk-next

**TL;DR:** `@galacticcouncil/sdk-next` (v1.6.0, papi v2) is the primary trading SDK for [[wiki/hydration\|hydration]], providing trade routing, pool queries, [[wiki/smart-order-router\|smart-order-router]], DCA/TWAP scheduling, transaction building, staking and liquidity-mining (farm) APIs. Built on [[wiki/papi\|papi]] (polkadot-api `^2.1.7`) for type-safe runtime access via [[wiki/sdk-descriptors\|sdk-descriptors]] (`>=2.6.0`). **Aug 2026 additions:** ICE intent tx builders, a standalone block `indexer`, a money-market oracle client, and peg recomputation for [[wiki/stableswap\|stableswap]].

## Quick Start

```ts
import { api, createSdkContext } from '@galacticcouncil/sdk-next';
import { createClient } from 'polkadot-api';

const provider = api.getWs('wss://hydradx-rpc.dwellir.com');
const client = createClient(provider);
const sdk = await createSdkContext(client);

// Optional: pin all reads to a specific block (historical / replay)
const sdkAt = await createSdkContext(client, { at: '0xabc…' });
```

The factory (`packages/sdk-next/src/factory.ts`) returns a `SdkCtx`:

- `api` — `{ aave, router, scheduler, staking, farm }` (trading + staking + liquidity mining)
- `client` — `{ asset, balance, evm }`
- `ctx` — `{ pool }` ([[wiki/sdk-next#Pool Context Providers\|#Pool Context Providers]])
- `tx` — `TxBuilderFactory`
- `destroy()` — cleans up pool subscriptions

By default the SDK auto-activates Aave + Omnipool + Stableswap + XYK pool clients. HSM and LBP exist on `poolCtx` (`withHsm()`, `withLbp()`) but are not activated by `createSdkContext`.

## Historical Queries (`at` option)

`createSdkContext(client, { at })` pins all chain reads to a block. Accepted `BlockAt` values (`packages/sdk-next/src/api/Papi.ts`):

- `'best'` (default) — read at best block, **subscribe** to live updates
- `'finalized'` — read at finalized block, **subscribe** to live updates
- `'0x…'` (block hash) — pinned read, **no subscriptions** (one-shot, no refresh)

When `at` is a block hash, `PoolContextProvider.isHistorical` is true and no rxjs subscriptions are created — the context is effectively static.

## Smart Order Router

The [[wiki/smart-order-router\|smart-order-router]] finds optimal trade paths across all [[wiki/hydration\|hydration]] pool types. See the dedicated page for details on the BFS algorithm and routing strategy. [[wiki/hydration-ui\|hydration-ui]] uses SDK Next for all trade estimation and routing.

Key methods on `sdk.api.router`:
- `getBestSell(assetIn, assetOut, amount)` — find best sell route by highest output
- `getBestBuy(assetIn, assetOut, amount)` — find best buy route by lowest input
- `getSpotPrice(assetIn, assetOut)` — price for 1 unit via most liquid route
- `getRoutes(assetIn, assetOut)` — enumerate all valid routes

## Pool Types Supported

Pool client implementations in `src/pool/` — each has a dedicated `*PoolClient.ts` that the [[wiki/smart-order-router\|smart-order-router]] traverses:

| Pool | Client | Math pkg | Description |
|------|--------|----------|-------------|
| [[omnipool\|omnipool]] | `OmniPoolClient` | `math-omnipool` | Hub-based AMM, routes through [[wiki/lrna\|lrna]] |
| [[wiki/xyk-pools\|XYK]] | `XykPoolClient` | `math-xyk` | Classic constant product (`x*y=k`) |
| [[wiki/stableswap\|stableswap]] | `StableSwapClient` | `math-stableswap` | Curve-style stable asset trading with amplification |
| [[wiki/lbp\|LBP]] | `LbpPoolClient` | `math-lbp` | Weighted pools for token launches |
| HSM | `HsmPoolClient` | `math-hsm` | HOLLAR stability mechanism — depends on `StableSwapClient`, syncs via EVM GHO facilitator events |
| Aave | `AavePoolClient` | — | aTokens routable through the trade router via `AaveTradeExecutor` runtime API; supply/withdraw via viem EVM |

All pool clients extend `PoolClient<T>` (`packages/sdk-next/src/pool/PoolClient.ts`) and expose `getPoolType()`, `loadPools()`, `getPoolFees()`, `getSubscriber()`. The common pool API exposes `validatePair()`, `parsePair()`, `validateAndSell()`, `validateAndBuy()`, `calculateInGivenOut()`, `calculateOutGivenIn()`, spot price functions.

Per-pool snapshot types are exported from `src/pool/{omni,xyk,lbp,stable}/types.ts` (`OmniSnapshot`, `XykSnapshot`, `LbpSnapshot`, stable fees) — used by [[wiki/sdk-next#Pool Context Providers\|#Pool Context Providers]] snapshot mode.

> **Stale share supply — verify before use.** `StableSwapClient` derives a pool's share supply from `Tokens.TotalIssuance(shareAssetId)` (`src/pool/stable/StableSwapClient.ts`, both the initial read and `subscribeIssuance()`) and feeds it to every `StableMath.*` call as `shareIssuance`. [[wiki/pallet-stableswap\|pallet-stableswap]] gained **virtual share-issuance storage** in its v1→v2 migration on the node side, so total issuance is no longer the authoritative share supply. Any add/remove-liquidity or share-price math routed through this client is suspect until it reads the pallet's own issuance storage. Flagged, not fixed, at HEAD `c57d172`.

### EVM log parsers

Three pure parsers turn `event.EVM.Log` payloads into typed events, for the clients that sync off EVM state rather than storage: `aave/AaveLog.ts`, `gho/GhoTokenLog.ts`, `oracle/MmOracleLog.ts`. The HSM types module moved `pool/hsm/types.ts` → `gho/types.ts`, and `pool/aave/AaveAbi.ts` was folded into `aave/abi.ts`.

### Stableswap pegs (`StableSwapPeg`)

`src/pool/stable/StableSwapPeg.ts` recomputes a pool's pegs and peg-adjusted fee off-chain, mirroring the pallet: `StableSwapPeg.compute(pool, poolPegs, latest, blockNumber)` calls `StableMath.recalculatePegs()` with the recent pegs, `max_peg_update` (perbill) and the base fee; `getDefault()` / `getRecent()` cover the no-peg and current-storage cases. `StableSwapClient` drives it from four subscriptions — `Stableswap.PoolPegs`, `EmaOracle.Oracles`, MM-oracle EVM logs, and the block stream — behind a `QueryBus` with scoped caches (`emaOracles`, `mmOracles`, `pegs`).

### Money-market oracles (`src/oracle/`)

`MmOracleClient` + `MmOracleLog` + `mappings.ts` resolve the `mmAddress`-es referenced by `Stableswap.PoolPegs` (kind `MMOracle`) into live prices. `docs/MM_ORACLES.md` enumerates all 7 deployed oracles in three kinds: **Managed** (Chainlink-compatible, keeper-pushed, `PriceUpdated(uint80,int256,uint256)`), **DIA wrapper** (facade over a DIA feed, `OracleUpdate(string,uint128,uint128)` on the *underlying* contract), and one **hybrid aggregator** (derived — no event, read `latestRoundData().updatedAt`). `docs/ORACLE_SPEC.md` is the design; `test/script/examples/oracle/discoverMmOracles.ts` re-derives the list.

### OmniPoolFee

Dynamic Omnipool fee math is extracted into `OmniPoolFee` (`packages/sdk-next/src/pool/omni/OmniPoolFee.ts`). Pure computation given `(pair, block, dynamicFee, oracleAssetFee, oracleProtocolFee, assetFeeParams, protocolFeeParams, maxSlipFee)` — usable by both live `OmniPoolClient` and the snapshot provider. Shared helpers (`getEmaPair`, `getEmaKey`) live in `omni/utils.ts`.

## Pool Context Providers

`IPoolCtxProvider` (`packages/sdk-next/src/pool/types.ts`) has two implementations:

**`PoolContextProvider`** (`packages/sdk-next/src/pool/PoolContextProvider.ts`) — live, subscription-based. Holds individual pool clients (`aave`, `omnipool`, `stableswap`, `hsm`, `xyk`, `lbp`) and exposes builder-style activators: `withAave()`, `withOmnipool()`, `withStableswap()`, `withHsm()` (auto-activates Stableswap), `withXyk(override?)`, `withLbp()`. When constructed with an `at` block hash the provider is `isHistorical` and no subscriptions are spun up.

**`SnapshotPoolCtxProvider`** (`packages/sdk-next/src/pool/SnapshotPoolCtxProvider.ts`) — stateless, offline. Implements `IPoolCtxProvider` from a static `SnapshotPoolCtx`:

```ts
interface SnapshotPoolCtx {
  block: number;
  pools: { aave; xyk; lbp; stable; omni };
  states: { omni: OmniSnapshot; xyk: XykSnapshot; lbp: LbpSnapshot };
}
```

Drop-in replacement for indexers, workers, replays, simulations, tests — anywhere a live RPC subscription is undesirable. No `destroy()` needed.

```ts
const ctx = new SnapshotPoolCtxProvider(snapshot);
const router = new TradeRouter(ctx);
```

## Subscription hygiene & the torn-snapshot bug

**Known, documented, unfixed at HEAD.** Pool state is kept live by *independent* storage subscriptions. On an Omnipool add-liquidity, `Omnipool.Assets[asset]` (`hub_reserve`, `shares`) and the pool account's token balance move in the same block but arrive 0.5–2 s apart, so the store briefly holds `hub_reserve@N` + `balance@N-1` — a **torn snapshot** whose implied price is wrong until the lagging half lands. Observed in production: a 16,700 aDOT add (~0.9% of supply) made sdk-next report a ~0.9% inflated price for ~20 blocks (#12514444–#12514464).

- Repro: `chopsticks/src/omni-tear.spec.ts` (forks Hydration, injects the coupled effect via `dev.setStorage`).
- Design history: `docs/SOR_BLOCK_CONSISTENCY.md` (v1, trigger + pinned read) → `SOR_BLOCK_CONSISTENCY_V2.md` (delta payload) → `POOL_EVENT_SYNC.md` + `POOL_EVENT_SYNC_IMPL.md` (v3, event-driven sync — **design accepted, not yet implemented**).

Related throughput work that *did* land: `OmniPoolClient.subscribeEmaOracles()` replaced ~23 per-pair `watchValue` calls with **one** prefix-scoped `watchEntries(ORACLE_NAME)` (~23 storage reads/block → ~1 merkle probe/block, descendant reads only when an Omnipool oracle actually changes). Delta-gating is now shared via `changedEntries()` and `rx.debounceAfterFirst()` from [[wiki/sdk-common\|sdk-common]].

`BalanceClient.getBreakdown()` also changed semantics — transferable is now `free - max(frozen - reserved, 0)` instead of `free - frozen`, so a balance whose freeze is already covered by a reserve is no longer under-reported.

## Trade Scheduling

`TradeScheduler` provides automated order splitting:

- **[[wiki/dca\|DCA]]** — splits order into multiple trades targeting 0.1% price impact each
- **TWAP** — time-weighted execution over fixed intervals (6 blocks per interval, max 6 hours)
- Validates minimum budgets and maximum price impact (-5% threshold)
- Every returned `TradeOrder` now carries **`assetOutEd`** (the destination asset's existential deposit, raw and in the `toHuman()` decimal form) so a caller can reject an order whose per-trade output would land below ED

## Transaction Building

Fluent API via `TxBuilderFactory`:
```ts
const tx = await sdk.tx.trade(trade)
  .withSlippage(5)    // 5% slippage tolerance
  .build();
```

Automatically selects direct [[omnipool\|omnipool]] operations vs. multi-hop `Router.sell()`/`Router.buy()` depending on route complexity.

### ICE intent builders (new)

Three builders submit to the `Intent` pallet of the **ICE** protocol, alongside the existing trade/order builders (`docs/INTENT.md`):

| Factory method | Builder | Emits | Notes |
|---|---|---|---|
| `sdk.tx.intentMarket(trade)` | `IntentMarketTxBuilder` | `IntentSwap` | min-out derived from `slippagePct` (default 1) |
| `sdk.tx.intentLimit(trade)` | `IntentLimitTxBuilder` | `IntentLimitOrder` | explicit `withMinAmountOut()` or fall back to `trade.amountOut`; `withPartial()` (default `true`) |
| `sdk.tx.intentOrder(order)` | `IntentOrderTxBuilder` | `IntentDcaSchedule` / `IntentDcaSchedule.twap` | dispatches on `TradeOrder.type` (`Dca` uses `amount_out = 1n`) |

`withBeneficiary(address)` is **required** — it drives an Aave debt check, and the call is auto-wrapped in `Dispatcher.dispatch_with_extra_gas` when the beneficiary has active borrow positions. Slippage is encoded on chain as `slippagePct * 10_000`. These builders run against the separate **`hydrationIce`** descriptor, so [[wiki/sdk-descriptors\|sdk-descriptors]] must be built first. Example scripts: `test/script/examples/ice/`.

## Block indexer (`src/indexer/`, new)

Standalone low-level primitives for bulk historical scans, independent of the pool/router stack. Not on `SdkCtx` — reached via the package namespace: `import { indexer } from '@galacticcouncil/sdk-next'`.

| Export | Role |
|---|---|
| `indexBlocks(opts)` | drive a range of blocks through a `BlockHandler`; returns `IndexBlocksResult` |
| `BlockFetcher` | fetch `chain_getBlock` (header + extrinsics) and/or the raw SCALE `System.Events` blob per block |
| `RpcPool` / `ClientSelector` | round-robin across several `PolkadotClient`s |
| `Semaphore` | bounded concurrency |
| `IndexerStats` / `IndexerSnapshot` | throughput + bytes accounting |
| `decodeCompactLength` | event count straight off the blob's compact prefix, without full decode |

`SYSTEM_EVENTS_KEY` (twox128("System") ++ twox128("Events")) is exported so the events blob can be pulled with a raw storage read. `RawBlock` carries `{ number, hash, header?, extrinsics, eventsHex?, eventsCount, bytes }` — both payloads are opt-in via `FetchBlockOptions { withBlock, withEvents }`. Benchmark: `test/script/examples/indexer/bench.ts`; endpoint latency helpers in `scripts/rpc-latency*.sh`.

## Staking & Liquidity Mining

Both are exposed via `sdk.api`:

- `sdk.api.staking` — `StakingApi` backed by `StakingClient` (`packages/sdk-next/src/staking/StakingClient.ts`). Queries `pallet-staking` constants (`PalletId`, `PeriodLength`, `UnclaimablePeriods`, `NFTCollectionId`), positions stored as Uniques NFTs, vote/conviction data.
- `sdk.api.farm` — `LiquidityMiningApi` backed by `LiquidityMiningClient`. Reads global/yield farms, deposits and entries. The `RewardClaimSimulator` (`packages/sdk-next/src/farm/RewardClaimSimulator.ts`) mirrors on-chain math (`math-liquidity-mining`) to estimate rewards: syncs global farm (`calculate_global_farm_rewards`), syncs yield farm (`calculate_yield_farm_delta_rpvs`), computes per-deposit reward (`calculate_user_reward`) with loyalty multipliers (`calculate_loyalty_multiplier`). Used to preview claim amounts without submitting an extrinsic.

## Key Types

- `Trade` — result of a route calculation: amountIn/Out, spotPrice, fees, priceImpact, swaps[]
- `Swap` — single hop: pool type, poolAddress, assetIn/Out, amounts, fees
- `TradeOrder` — scheduled order: DCA or TWAP with tradeCount, tradePeriod, route, `assetOutEd`
- `BlockAt` — `'best' | 'finalized' | (string & {})` — block selector for all reads
- `SnapshotPoolCtx` — offline pool context input for `SnapshotPoolCtxProvider`
- `RawBlock` / `IndexBlocksOptions` / `IndexBlocksResult` — indexer inputs and outputs

## EVM client honours `at`

`EvmClient` and `EvmRpcAdapter` now take the `BlockAt` from `SdkOptions` (default `'best'`) and thread it through `Ethereum.CurrentBlock` reads and `EthereumRuntimeRPCApi.call`, so an EVM read at a pinned block hash actually resolves at that block. Previously they were hardcoded to `'best'`, silently mixing live EVM state into an otherwise historical context.

## Dependencies

Consumes [[wiki/sdk-common\|sdk-common]] (peer, `>=1.2.0`), [[wiki/sdk-descriptors\|sdk-descriptors]] (peer, `>=2.6.0`), `polkadot-api
{ #2}
.1.7` (peer), `viem
{ #2}
.38.3` (peer), `@thi.ng/cache`, `@thi.ng/memoize`, `@noble/hashes`, `big.js`, and all math-* WASM packages (`omnipool`
{ #1}
.4.0, `stableswap`
{ #2}
.5.0, `xyk`, `lbp`, `hsm`, `staking`, `liquidity-mining`).

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
