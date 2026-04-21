---
{"dg-publish":true,"permalink":"/wiki/hydration-ui/","title":"hydration-ui","tags":["ui","frontend","hydration","react","typescript"],"dg-note-properties":{"type":"package","title":"hydration-ui","repo":"hydration-ui","paths":["apps/main/src/App.tsx","packages/ui/src","packages/web3-connect/src","packages/money-market/src","packages/indexer/src","packages/utils/src"],"key_exports":["App","ThemeProvider","Web3ConnectModal","useAccount","useSquidClient"],"tags":["ui","frontend","hydration","react","typescript"],"last_updated":"2026-04-20"}}
---


# hydration-ui

**TL;DR:** The primary frontend monorepo for the Hydration protocol — Yarn workspace with the main app (`apps/main`) and 6 packages implementing the design system, wallet connectors, lending hooks, GraphQL clients, and shared utilities.

## Role

Implements the user-facing interface for all [[wiki/hydration\|hydration]] protocol features: trading ([[omnipool\|omnipool]], [[wiki/stableswap\|stableswap]]), liquidity management ([[wiki/nft-lp-positions\|nft-lp-positions]]), lending ([[wiki/hydration-borrow\|hydration-borrow]]), cross-chain transfers ([[wiki/xc-sdk\|xc-sdk]]), staking, and governance. The main app orchestrates these via [[wiki/papi\|papi]]-based data fetching and [[wiki/sdk-next\|sdk-next]] for transaction construction.

## App entry point: `apps/main/`

Root file: `src/App.tsx`. Sets up:
- [[wiki/papi\|papi]] RPC provider (via `rpcProvider` context)
- Tanstack Query client with error boundary
- Tanstack Router with file-based routes (`src/routes/`)
- i18next for translations
- Emotion CSS-in-JS + [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] theme
- Sonner toast notifications
- DataProviderResolver (chain selection)

```typescript
// apps/main/src/App.tsx (excerpt)
export const App = () => {
  return (
    <I18nextProvider i18n={i18n}>
      <QueryClientProvider client={queryClient}>
        <DataProviderResolver>
          <ThemeProvider>
            <TooltipProvider delayDuration={0}>
              <RouterProvider router={router} />
              <Toaster />
            </TooltipProvider>
          </ThemeProvider>
        </DataProviderResolver>
      </QueryClientProvider>
    </I18nextProvider>
  )
}
```

## Product modules

Organized under `src/modules/`:
- `trade/` — Swap UI (omnipool, stableswap, OTC orders) → links [[omnipool\|omnipool]], [[wiki/smart-order-router\|smart-order-router]]
- `liquidity/` — LP position management → links [[wiki/nft-lp-positions\|nft-lp-positions]]
- `borrow/` — Lending UI (Aave v3 protocol) → links [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/pallet-liquidation\|pallet-liquidation]]
- `xcm/` — Cross-chain transfer UI → links [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-package\|xc-package]]
- `wallet/` — Asset portfolio + history
- `staking/` — HDX staking → links [[wiki/pallet-staking\|pallet-staking]]
- `stats/` — Analytics dashboard
- `transactions/` — Tx history display
- `submit-transaction/` — Centralized tx signing modal
- `layout/` — App shell + navigation

For details on each, see [[wiki/hydration-ui-modules\|hydration-ui-modules]].

## Data layer: `src/api/`

React Query hooks organized by domain:
- `aave.ts` — Aave market data (via @aave/contract-helpers)
- `account.ts` — Account state, balances, nonces
- `assets.ts` — Asset metadata + registry
- `borrow/` — Lending market queries
- `omnipool.ts` — Pool state, asset pricing
- `stableswap.ts` — Stableswap pool data
- `spotPrice.ts` — Real-time pricing subscriptions
- `staking.ts` — Staking rewards, locks
- `trade.ts` — Trade execution data
- `xcm.ts` — Cross-chain transfer state

See [[wiki/hydration-ui-api\|hydration-ui-api]] for the full inventory.

## State stores: `src/states/`

Zustand-based stores (with immer for immutability):
- `account.ts` — Account address, sign method, permissions
- `assetRegistry.ts` — Asset metadata cache
- `displayAsset.ts` — UI-level asset selection
- `liquidity.ts` — LP position form state
- `provider.ts` — RPC provider endpoint + chain selection
- `toasts.ts` — Toast notification queue
- `tradeSettings.ts` — Trade slippage, execution mode
- `transactions.ts` — Pending/confirmed tx queue

## Providers: `src/providers/`

Context providers wrapping SDK/papi integration:
- `rpcProvider.tsx` — [[wiki/papi\|papi]] RPC client initialization; exposes `useRpcProvider()` hook
- `assetsProvider.tsx` — Asset registry fetching + caching

## Cross-chain: [[wiki/xc-sdk\|xc-sdk]] integration

[[wiki/hydration-ui\|hydration-ui]] consumes [[wiki/xc-sdk\|xc-sdk]] and [[wiki/xc-package\|xc-package]] for the `xcm/` module. Chain selection, balance transfers, and XCMP message handling are orchestrated here.

## Tech stack highlights

- React 19, Tanstack Router (file-based routing), Tanstack Query (data fetching)
- Emotion + [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] (Radix-based component library)
- i18next (i18n)
- Zustand (state)
- viem + ethers (EVM integration for [[wiki/hydration-ui-money-market\|hydration-ui-money-market]])
- [[wiki/papi\|papi]] 1.23.3 (pinned) for runtime type-safety
- comlink (Web Workers for async tasks)

## Monorepo structure

- **apps/main/** — The web app
- **packages/ui/** — Design system (see [[wiki/hydration-ui-design-system\|hydration-ui-design-system]])
- **packages/web3-connect/** — Wallet abstraction (see [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]])
- **packages/money-market/** — Aave hooks (see [[wiki/hydration-ui-money-market\|hydration-ui-money-market]])
- **packages/indexer/** — GraphQL clients (see [[wiki/hydration-ui-indexer\|hydration-ui-indexer]])
- **packages/utils/** — Utilities (see [[wiki/hydration-ui-utils\|hydration-ui-utils]])
- **packages/eslint-config, typescript-config/** — Workspace config

## For developers

To understand a specific flow (trade, lend, stake, cross-chain):
1. Start with [[wiki/hydration-ui-modules\|hydration-ui-modules]]
2. Drill into the module's API hooks in [[wiki/hydration-ui-api\|hydration-ui-api]]
3. Check the state stores in `src/states/`
4. Review [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] for tx signing
5. Cross-reference [[wiki/papi\|papi]] + [[wiki/sdk-next\|sdk-next]] docs for chain-level details

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration\|hydration]], [[wiki/papi\|papi]], [[wiki/sdk-next\|sdk-next]], [[wiki/xc-sdk\|xc-sdk]]
