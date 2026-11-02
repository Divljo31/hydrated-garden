---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-indexer/","title":"hydration-ui indexer","tags":["indexer","graphql","rest","neckwork","data-layer","hydration"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui indexer","repo":"hydration-ui","paths":["packages/indexer/src/indexer","packages/indexer/src/multix","packages/indexer/src/neckwork","packages/indexer/src/scan","packages/indexer/package.json","apps/main/src/api/neckwork.ts","apps/main/src/api/neckworkSync.ts","apps/main/src/api/scan.ts","apps/main/src/api/indexer.ts","apps/main/src/api/grafana"],"key_exports":["getIndexerSdk","IndexerSdk","getMultixSdk","MultixSdk","getNeckworkClient","NeckworkClient","getScanClient","ScanClient"],"symbols":["getIndexerSdk","newFarmsQuery","getMultixSdk","multisigsByAccountIdsQuery","getNeckworkClient","NeckworkApiError","isNeckworkNotFound","NECKWORK_ACCOUNT_KEY","NECKWORK_BASE_STALE_TIME","withEpoch","neckworkStatusQuery","platformStatsQuery","accountsBalancesQuery","dcaSchedulesQuery","dcaExecutionsInfiniteQuery","intentsQuery","intentEventsInfiniteQuery","routedTradesQuery","marketTradesQuery","pairCandlesQuery","pairCandlesInfiniteQuery","pairReferencePriceQuery","omnipoolVolumeQuery","stablepoolVolumeQuery","xykVolumeQuery","uniswapV3VolumeQuery","omnipoolYieldMetricsQuery","stablepoolYieldMetricsQuery","feesChartQuery","moneyMarketEventsQuery","stakingEventsQuery","otcOrderQuery","extrinsicByHashQuery","getScanClient","ScanApiError","scanTransfersQuery","subscribeScanEvents","useNeckworkSync"],"key_deps":["graphql-request","openapi-fetch","openapi-typescript","graphql-codegen","zod"],"tags":["indexer","graphql","rest","neckwork","data-layer","hydration"],"last_updated":"2026-11-02"}}
---


# hydration-ui indexer

**TL;DR:** `@galacticcouncil/indexer` (`packages/indexer`) holds one client per backend, imported by subpath: **`/neckwork`** (typed REST over the Neckwork Hydration API — now the main historical-data source), **`/scan`** (zod-validated REST for the transfers and intents scan apps), **`/indexer`** (GraphQL, farms only) and **`/multix`** (GraphQL, multisigs). The squid GraphQL client was retired at `770f868`.

## Package exports

```json
// packages/indexer/package.json → exports
{ "./indexer": "./src/indexer/index.ts",
  "./neckwork": "./src/neckwork/index.ts",
  "./multix": "./src/multix/index.ts",
  "./scan": "./src/scan/index.ts" }
```

Source TypeScript, no `dist/`. Deps: `graphql`, `graphql-request`, `openapi-fetch`; dev: `@graphql-codegen/*`, `openapi-typescript`.

## neckwork/ — Neckwork REST API (primary)

`getNeckworkClient(baseUrl)` = `openapi-fetch` `createClient<paths>` + a middleware that throws `NeckworkApiError { status, path }` on non-2xx (server `error.message` included). Types come from `src/neckwork/openapi.json` → `__generated__/schema.ts`:

```bash
yarn workspace @galacticcouncil/indexer schema:api    # curl https://hydration-api.neckwork.net/docs/json
yarn workspace @galacticcouncil/indexer codegen:api   # openapi-typescript --alphabetize
```

Query factories (React Query options objects, take a `NeckworkClient`):

| File | Exports |
|---|---|
| `accounts.ts` | `accountsBalancesQuery` |
| `dca.ts` | `dcaSchedulesQuery`, `dcaSchedulesCountQuery`, `dcaExecutionsInfiniteQuery`, `DCA_STATUSES` / `DCA_OPEN_STATUSES` / `DCA_HISTORY_STATUSES` |
| `intents.ts` | `intentsQuery`, `intentsCountQuery`, `intentQuery`, `intentEventsInfiniteQuery`, `INTENT_STATUSES`, `INTENT_KINDS` ([[wiki/ice\|ice]] intents) |
| `trades.ts` | `routedTradesQuery`, `marketTradesQuery` |
| `prices.ts` | `pairCandlesQuery`, `pairCandlesInfiniteQuery`, `pairReferencePriceQuery`, candle helpers (`CANDLE_BUCKETS`, `mergeCandles`, `invertCandle`, `peggedCandles`) |
| `pools.ts` | `omnipoolVolumeQuery`, `stablepoolVolumeQuery`, `xykVolumeQuery`, `uniswapV3VolumeQuery`, `omnipoolYieldMetricsQuery`, `stablepoolYieldMetricsQuery` |
| `fees.ts` | `feesChartQuery`, `foldFeesChart`, `FEE_STREAMS`, `TIME_RANGES` (feeds `/stats/fees`) |
| `money-market.ts` | `moneyMarketEventsQuery` |
| `staking.ts` | `stakingEventsQuery` |
| `otc.ts` | `otcOrderQuery` |
| `extrinsics.ts` | `extrinsicByHashQuery`, `extrinsicByBlockAndIndexQuery` |
| `stats.ts` | `platformStatsQuery`, `neckworkStatusQuery` |
| `index.ts` | `NECKWORK_BASE_STALE_TIME` (60 s), `NECKWORK_STATUS_STALE_TIME` (10 s), `NECKWORK_ACCOUNT_KEY = ["neckwork","account"]`, `withEpoch`, `isNeckworkNotFound` |

App wiring: `apps/main/src/api/neckwork.ts` → `neckworkClient = getNeckworkClient(ENV.VITE_NECKWORK_URL)`. **Post-tx freshness:** `api/neckworkSync.ts` → `useNeckworkSync()` arms on a submitted tx (`states/neckwork.ts` store), polls `/v1/status` every 10 s until the indexer passes the tx block (+10 block grace, 120 s timeout), then invalidates the `NECKWORK_ACCOUNT_KEY` subtree once. Disabled on forks (`isFork`) and when `useNeckworkEnabled()` is false.

## scan/ — transfers & intents scan apps

`getScanClient(baseUrl)` → `{ baseUrl, get(path, zodSchema, query) }`; 10 s `AbortSignal.timeout`, throws `ScanApiError`. No spec upstream, so every response is parsed with zod. Exports `scanTransfersQuery`, `scanTransferQuery`, `subscribeScanEvents` (SSE-style `ScanFlow` / `ScanUpdate` handlers). App: `api/scan.ts` → `transfersScanClient` (`ENV.VITE_TRANSFERS_SCAN_URL`) and `intentsScanClient` (`intentscan.apiUrl`). Used by cross-chain history ([[wiki/hydration-ui-modules\|hydration-ui-modules]] `xcm/history`).

## indexer/ — GraphQL (farms only)

`getIndexerSdk(url)` over `graphql-request`; one operation left: `newFarmsQuery(indexerSdk, blockNumber)` (`farms.graphql`). App: `api/indexer.ts` (endpoint from `ENV.VITE_INDEXER_URL`). Codegen: `codegen:indexer` against `schema.indexer.graphql`.

## multix/ — multisigs

`getMultixSdk(url)`, `multisigsByAccountIdsQuery`. Consumed by [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] (`hooks/useAccountMultisigs.ts`), which re-exports it. Codegen: `codegen:multix`.

## Grafana (app side, not in this package)

`apps/main/src/api/grafana/` now holds only `fetchGrafana.ts` + `reserveRate.{sql,ts}` (money-market reserve-rate chart; `VITE_GRAFANA_URL`, `VITE_GRAFANA_DSN`). Trade charts and DCA amounts moved to Neckwork `prices.ts` / `dca.ts`.

## Gotchas

- Only `/indexer` and `/multix` are GraphQL; everything new is REST. Do not add squid-style `.graphql` operations for data Neckwork already serves.
- Neckwork timestamps are ISO strings; wrap rows with `withEpoch` before charting.
- Account-scoped Neckwork queries **must** key under `NECKWORK_ACCOUNT_KEY` or `useNeckworkSync` will not refresh them after a tx.

## Sources

- [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]]
- [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]]
