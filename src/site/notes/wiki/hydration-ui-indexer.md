---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-indexer/","title":"hydration-ui indexer","tags":["indexer","graphql","data-layer","hydration"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui indexer","repo":"hydration-ui","paths":["packages/indexer/src/indexer","packages/indexer/src/squid","packages/indexer/src/multix"],"key_exports":["getIndexerSdk","IndexerSdk","getSquidSdk","SquidSdk","getMultixSdk","MultixSdk"],"symbols":["getIndexerSdk","IndexerSdk","getSquidSdk","SquidSdk","getMultixSdk","MultixSdk","newFarmsQuery","scheduledOrdersQuery","ordersStatusQuery","orderTradesQuery","orderPlannedExecutionQuery","latestBlockHeightQuery","otcOrderStatusQuery","multisigsByAccountIdsQuery","parseIndexerErrorState","parseIndexerUrlName"],"key_deps":["graphql-request","graphql-codegen","graphql"],"tags":["indexer","graphql","data-layer","hydration"],"last_updated":"2026-08-15"}}
---


# hydration-ui indexer

**TL;DR:** GraphQL client library (`packages/indexer/`). **Three** clients as of Aug 2026: `indexer` (Subsquid archive — raw events/extrinsics), `squid` (PostGraphile aggregation backend), `multix` (self-hosted multisig backend). The `snowbridge` client was **deleted** (commit `0ce954f`); chart data is migrating off squid to Grafana SQL (`apps/main/src/api/grafana/`). Uses `graphql-request` + `graphql-codegen` for type-safe queries.

## Package exports

Per `packages/indexer/package.json`:

```json
"exports": {
  "./indexer":    "./src/indexer/index.ts",
  "./squid":      "./src/squid/index.ts",
  "./squid/lib/*":"./src/squid/lib/*.ts",
  "./multix":     "./src/multix/index.ts"
}
```

Each client exposes a `getXSdk(url)` factory backed by codegen output under `__generated__/`. Codegen scripts: `codegen:indexer`, `codegen:squid`, `codegen:multix`.

| Client | Factory / type | Codegen schema endpoint |
|---|---|---|
| `indexer` | `getIndexerSdk(url)` → `IndexerSdk` | `https://archive.nice.hydration.cloud/graphql` |
| `squid` | `getSquidSdk(url)` → `SquidSdk` | `https://orca-prod-pool-01.orca.hydration.cloud/graphql` |
| `multix` | `getMultixSdk(url)` → `MultixSdk` | `https://multix-graphql.lark.hydration.cloud/graphql` (self-hosted since `f155a36`) |

App-side URLs come from `apps/main/src/config/env.ts` (`VITE_INDEXER_URL`, `VITE_SQUID_URL`); there is **no** snowbridge env var any more. The multix GraphQL URL is a constant in `@galacticcouncil/utils` (`helpers/multix.ts` → `multix.graphql`).

## Purpose

Provides historical and aggregated data that the on-chain [[wiki/papi\|papi]] RPC cannot efficiently serve:
- Trade history and volumes
- Liquidity provider rewards / farm creation
- DCA schedules and past executions, OTC fill progress
- Multisig pending calls
- Analytics (TVL, fees, etc.)

Cross-chain transfer status is **no longer** served from here — see [[wiki/xc-sdk\|xc-sdk]] / `xc-scan`.

## Three clients

### indexer/

Subsquid **archive** — raw `events` / `extrinsics` tables, queried by event name + `args_jsonContains`. Query modules (each `*.graphql` + a `*.ts` `queryOptions` factory):

| File | Queries | Consumed by |
|---|---|---|
| `extrinsics.ts` | extrinsic lookup | tx status |
| `farms.ts` | `YieldFarmCreated` | `useNewFarms()` → NewFarmsBanner |
| `otc.ts` | `OtcOrderStatus` | OTC fill progress |
| `staking.graphql` | staking events | [[wiki/pallet-staking\|pallet-staking]] history |
| `trade-orders.ts` (NEW Aug 2026, `d08260e`) | `ScheduledOrders`, `OrdersStatus`, `OrderTrades`, `OrderPlannedExecution` | DCA past executions + planned execution ([[wiki/pallet-dca\|pallet-dca]]) |

Exports: `getIndexerSdk(url)` → `IndexerSdk`

**`farms.graphql`:**
```graphql
query YieldFarmCreated($blockNumber: Int!) {
  events(
    where: {
      name_eq: "OmnipoolLiquidityMining.YieldFarmCreated"
      block: { height_gte: $blockNumber }
    }
    orderBy: [block_height_ASC]
  ) { args }
}
```

`farms.ts` wraps it as a `queryOptions` factory (`newFarmsQuery(indexerSdk, blockNumber)`) consumed by `useNewFarms()` in `apps/main/src/api/farms.ts` to drive the `NewFarmsBanner` ([[wiki/hydration-ui-modules\|hydration-ui-modules]]).

**`trade-orders.ts` factories** (all `queryOptions`, keyed `["trade","orders",…]`):
```typescript
// packages/indexer/src/indexer/trade-orders.ts
export const scheduledOrdersQuery = (indexerSdk: IndexerSdk, who: string) => …
export const ordersStatusQuery = (indexerSdk: IndexerSdk, who: string) => …
export const orderTradesQuery = (indexerSdk: IndexerSdk, id: number) => …
export const orderPlannedExecutionQuery = (indexerSdk: IndexerSdk, id: number) => …
```
Backing queries filter `DCA.Scheduled`, `DCA.{Terminated,Completed}`, `DCA.{TradeExecuted,TradeFailed}`, `DCA.ExecutionPlanned`.

`otcOrderStatusQuery` dropped the `QUERY_KEY_BLOCK_PREFIX` key prefix in favour of `staleTime: Infinity` (`091428d`) — it no longer invalidates per block.

### squid/

PostGraphile aggregation backend (Relay-style connections, `nodes`/`edges`). Query modules:

| File | Purpose |
|---|---|
| `account-balances.ts` | historical account balances |
| `blocks.ts` | `latestBlockHeightQuery(squidSdk, url)` — indexer liveness probe |
| `money-market.ts` | Aave reserve history (feeds [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] charts) |
| `platform-total.ts` | protocol TVL / totals |
| `pool-metrics.ts` | per-pool volume, fees, TVL |
| `trade-orders.ts` | user orders, DCA schedules + executions |
| `trade-prices.ts` | historical price series |
| `volume.ts` | volume aggregates |
| `lib/parseIndexerErrorState.ts`, `lib/parseIndexerUrlName.ts` | error/URL helpers, exported via `./squid/lib/*` |

Exports: `getSquidSdk(url)` → `SquidSdk`

**`latestBlockHeightQuery` (Aug 2026 rewrite):** the `refetchInterval` parameter was dropped and hardcoded to `10_000`; failures now swallow to `null` instead of throwing, so a dead squid no longer surfaces as a query error.

```typescript
// packages/indexer/src/squid/blocks.ts
export const latestBlockHeightQuery = (squidSdk: SquidSdk, url: string) =>
  queryOptions({
    queryKey: ["latestBlockHeight", url],
    queryFn: async () => {
      try {
        const result = await squidSdk.LatestBlockHeightQuery()
        return result.blocks?.edges?.[0]?.node?.height ?? null
      } catch {
        return null
      }
    },
    refetchInterval: 10_000,
    retry: false,
  })
```

**`trade-orders.graphql` (extended):** the `Swap` and `RoutedTradeSwap` fragments now carry `event { paraBlockHeight indexInBlock call { extrinsic { indexInBlock } } }` so trades can deep-link into the Neckwork explorer (`neckwork.activityExtrinsic(...)` in [[wiki/hydration-ui-utils\|hydration-ui-utils]]). `DcaScheduleExecutions` gained `$first` + `orderBy: ID_DESC` + `totalCount` for paginated past executions.

### multix/

Self-hosted Multix multisig backend (`f155a36` moved it from `hydration.graphql.multix.cloud` to `multix-graphql.lark.hydration.cloud`). Tracks pending multisig calls, approvals and account membership via `accounts.graphql` (`MultisigsByAccountIds`).

Exports: `getMultixSdk(url)` → `MultixSdk`, plus `multisigsByAccountIdsQuery`.

**Consumer moved.** Multix is no longer wired through `apps/main/src/api/multisig.ts` alone — [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] re-exports `getMultixSdk` / `multisigsByAccountIdsQuery` / `MultixSdk` / `MultisigsByAccountIdsQuery` from its own `index.ts` and consumes them in `hooks/useAccountMultisigs.ts`.

### (removed) snowbridge/

Deleted in `0ce954f` — `src/snowbridge/`, `schema.snowbridge.graphql` (3199 lines) and the `codegen:snowbridge` script are gone, as is the `useSnowbridgeClient` wiring in `apps/main/src/api/provider.ts`. Snowbridge transfer status is now resolved through the [[wiki/xc-sdk\|xc-sdk]] / `xc-scan` path in the `xcm/` module ([[wiki/hydration-ui-modules\|hydration-ui-modules]]). The `packages/utils/src/helpers/snowbridge.ts` helper was removed in the same wave.

## Charts: squid → Grafana SQL

Chart series are migrating off the squid GraphQL client onto raw SQL executed against Grafana (`ed70c8b`):

```
apps/main/src/api/grafana/
├── fetchGrafana.ts      # POST { queries: [{ refId, rawSql, format: "table", datasourceId }] }
├── TradeChartApi.ts     # buildQuery(params) + transform(rawData)
├── tradeChart.{sql,ts}  # OHLC buckets for the trade chart
├── dcaAmount.{sql,ts}   # DCA spent/received series
└── reserveRate.{sql,ts} # money-market reserve rate history
```

`fetchGrafana(sql, refId, signal)` unwraps `data.results[refId].frames[0].data.values`. Config: `VITE_GRAFANA_URL` + `VITE_GRAFANA_DSN` in `apps/main/src/config/env.ts`. Query keys are prefixed `["grafana", …]`. See [[wiki/hydration-ui-api\|hydration-ui-api]].

## GraphQL codegen

Each client has a `codegen.ts` that:
1. Fetches the remote GraphQL schema
2. Emits a `schema.<client>.graphql` snapshot (schema-ast plugin)
3. Generates `__generated__/types.ts` (typescript), `operations.ts` (typescript-operations) and `sdk.ts` (typescript-graphql-request), wired together with the `import-types` preset

```bash
yarn codegen:indexer
yarn codegen:squid
yarn codegen:multix
```

Generated files are **committed** under `src/<client>/__generated__/{types,operations,sdk}.ts` (plus the schema snapshot at package root). Never edit by hand — edit the `*.graphql` document and re-run codegen.

## Client hooks (app side)

`useIndexerClient()` / `useSquidClient()` live in `apps/main/src/api/provider.ts` (not in this package); they memoize a `GraphQLClient` per URL. `useAccountMultisigs()` in [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] holds the multix client.

## Integration with API layer

The [[wiki/hydration-ui-api\|hydration-ui-api]] hooks use these clients:

```typescript
// apps/main/src/api/farms.ts (pattern)
import { newFarmsQuery } from "@galacticcouncil/indexer/indexer"

const indexerSdk = useIndexerClient()
const { data } = useQuery(newFarmsQuery(indexerSdk, blockNumber))
```

## Typical use cases

| Use case | Client | Hook / factory |
|----------|--------|------|
| Raw chain events (DCA, OTC, farms) | indexer | `useIndexerClient()` |
| DCA schedules + past executions | indexer + squid | `scheduledOrdersQuery`, `DcaScheduleExecutions` |
| TVL, volumes, fees, pool metrics | squid | `useSquidClient()` |
| Money-market reserve history | squid | `money-market.ts` |
| Indexer liveness | squid | `latestBlockHeightQuery` |
| Multisig pending calls | multix | `useAccountMultisigs()` ([[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]]) |
| OHLC / trade chart, DCA amounts | **Grafana SQL** | `tradeChartQuery`, `dcaAmount` |
| XCM / bridge transfer status | — | [[wiki/xc-sdk\|xc-sdk]] + `xc-scan` (snowbridge client removed) |

## For developers

To add a new GraphQL query:

1. **Define schema** (remote endpoint is fetched by codegen)
2. **Write query** in `src/<client>/*.graphql`
3. **Run codegen:** `yarn codegen:<client>`
4. **Commit the `__generated__/` diff**
5. **Wrap it in a `queryOptions` factory** in the sibling `*.ts` file, then consume it from `apps/main/src/api/`

The house pattern is a factory taking the SDK as first argument (not a hook), so callers own the client:

```typescript
// packages/indexer/src/indexer/farms.ts
export const newFarmsQuery = (indexerSdk: IndexerSdk, blockNumber: number) =>
  queryOptions({
    queryKey: ["farms", "new", blockNumber],
    queryFn: () => indexerSdk.YieldFarmCreated({ blockNumber }),
  })
```

For chart-shaped data prefer the Grafana SQL path (`apps/main/src/api/grafana/`) over adding a new squid query.

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]], [[wiki/hydration-ui-utils\|hydration-ui-utils]]
