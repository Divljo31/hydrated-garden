---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-indexer/","title":"hydration-ui indexer","tags":["indexer","graphql","data-layer","hydration"],"dg-note-properties":{"type":"package","title":"hydration-ui indexer","repo":"hydration-ui","paths":["packages/indexer/src/indexer","packages/indexer/src/squid","packages/indexer/src/snowbridge"],"symbols":["useIndexerClient","useSquidClient","useSnowbridgeClient"],"key_deps":["graphql-request","graphql-codegen","graphql"],"tags":["indexer","graphql","data-layer","hydration"],"last_updated":"2026-04-20"}}
---


# hydration-ui indexer

**TL;DR:** GraphQL client library (`packages/indexer/`). Three subgraph clients: `indexer` (custom), `squid` (Subsquid-based), `snowbridge` (bridge events). Uses `graphql-request` + `graphql-codegen` for type-safe queries.

## Purpose

Provides historical and aggregated data that the on-chain [[wiki/papi\|papi]] RPC cannot efficiently serve:
- Trade history and volumes
- Liquidity provider rewards
- Cross-chain transfer status
- Analytics (TVL, fees, etc.)

## Three clients

### indexer/

Custom indexing backend. Indexes:
- Swap events (trade pairs, volumes, prices)
- Pool state snapshots
- Fee collection history
- User trading patterns

Exports: `useIndexerClient()` hook

**Typical queries:**
```graphql
query GetTradeHistory($account: String!) {
  trades(where: { account: $account }, orderBy: timestamp_DESC) {
    id
    assetIn
    assetOut
    amountIn
    amountOut
    timestamp
  }
}
```

### squid/

Subsquid-based indexing. More comprehensive event/state tracking:
- All extrinsics + calls
- Account state changes
- Historical balances
- Governance actions

Exports: `useSquidClient()` hook

**Typical queries:**
```graphql
query GetAccountBalance($account: String!) {
  accountBalances(where: { account: { id_eq: $account } }) {
    asset
    balance
    timestamp
  }
}
```

### snowbridge/

Bridge-specific indexing (XCM / Snowbridge):
- Outbound messages (Polkadot → EVM)
- Inbound messages (EVM → Polkadot)
- Bridge fee history
- Channel status

Exports: `useSnowbridgeClient()` hook

**Typical queries:**
```graphql
query GetXcmHistory($account: String!) {
  xcmTransfers(where: { sender: $account }) {
    id
    sourceChain
    destChain
    amount
    status
    timestamp
  }
}
```

## GraphQL codegen

Each client has a codegen config (`codegen.ts` or `codegen.json`) that:
1. Fetches the remote GraphQL schema
2. Generates TypeScript types from schema
3. Generates typed query/mutation hooks (via `typescript-graphql-request` plugin)

```bash
yarn codegen:indexer
yarn codegen:squid
yarn codegen:snowbridge
```

Generated files are in `src/<client>/generated/` (gitignored; regenerated on build).

## Package exports

```typescript
export { useIndexerClient } from "./indexer"
export { useSquidClient } from "./squid"
export { useSnowbridgeClient } from "./snowbridge"

// Sub-exports for direct query access
export * from "./indexer/index"
export * from "./squid/lib"
export * from "./snowbridge/index"
```

## Integration with API layer

The [[wiki/hydration-ui-api\|hydration-ui-api]] hooks use these clients:

```typescript
// apps/main/src/api/stats.ts (example)
import { useSquidClient } from "@galacticcouncil/indexer"

export const useStats = () => {
  const squid = useSquidClient()
  
  return useSuspenseQuery({
    queryKey: ["stats"],
    queryFn: async () => {
      const result = await squid.request(GetStatsDocument, {})
      return result
    }
  })
}
```

## Typical use cases

| Use case | Client | Hook |
|----------|--------|------|
| Trade history | indexer | `useIndexerClient()` |
| Account activity | squid | `useSquidClient()` |
| XCM transfer status | snowbridge | `useSnowbridgeClient()` |
| TVL, volumes, fees | squid | `useSquidClient()` |
| Pool trading pairs | indexer | `useIndexerClient()` |

## For developers

To add a new GraphQL query:

1. **Define schema** (remote endpoint is fetched by codegen)
2. **Write query** in `src/<client>/*.graphql`
3. **Run codegen:** `yarn codegen:<client>`
4. **Use generated hook** in [[wiki/hydration-ui-api\|hydration-ui-api]]

Example adding a trade volume query:

```graphql
# src/indexer/queries.graphql
query GetTradeVolume($assetIn: String!, $assetOut: String!, $period: Int!) {
  trades(where: {
    assetIn: $assetIn
    assetOut: $assetOut
    timestamp_gte: $period
  }) {
    amountIn
    amountOut
  }
}
```

Then in API layer:

```typescript
const { useGetTradeVolume } = useIndexerClient()

const useTradeVolume = (assetIn, assetOut, period) => {
  return useSuspenseQuery({
    queryKey: ["trade-volume", assetIn, assetOut, period],
    queryFn: async () => {
      const client = useIndexerClient()
      return client.request(GetTradeVolumeDocument, {
        assetIn, assetOut, period
      })
    }
  })
}
```

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-modules\|hydration-ui-modules]]
