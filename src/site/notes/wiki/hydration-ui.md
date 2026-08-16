---
{"dg-publish":true,"permalink":"/wiki/hydration-ui/","title":"hydration-ui","tags":["ui","frontend","hydration","react","typescript","monorepo"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui","repo":"hydration-ui","paths":["apps/main/src/App.tsx","apps/main/src/modules","apps/main/src/api","apps/main/src/states","apps/main/src/providers","packages/ui/src","packages/web3-connect/src","packages/money-market/src","packages/indexer/src","packages/utils/src"],"key_exports":["App","RouterContext","ThemeProvider","MoneyMarketProvider","useAccount","useSquidClient","useIndexerClient","useMultixClient"],"symbols":["DataProviderResolver","AssetRegistryGate","MultisigProvider","rpcProvider","assetsProvider","routeTree"],"tags":["ui","frontend","hydration","react","typescript","monorepo"],"last_updated":"2026-08-15"}}
---


# hydration-ui

**TL;DR:** The frontend monorepo for the Hydration protocol — one deployable app (`apps/main`) plus 7 workspace packages. As of `b469e8a` (2026-08-12) it ships **13 feature modules**, a **GigaHDX liquid-staking** front page for `/staking`, three money markets, an NTT-based bridge stack, and no Snowbridge.

## Role

The user-facing surface for every [[wiki/hydration\|hydration]] product: trading ([[omnipool\|omnipool]], [[wiki/stableswap\|stableswap]], [[wiki/otc-trading\|otc-trading]], [[wiki/dca\|dca]]), liquidity, lending ([[wiki/hydration-borrow\|hydration-borrow]]), [[wiki/gigahdx\|gigahdx]] liquid staking, RWA/bond strategies, cross-chain transfers ([[wiki/xc-sdk\|xc-sdk]]), fiat on/off-ramp, and stats. Chain access is [[wiki/papi\|papi]]; trade construction and pool math come from [[wiki/sdk-next\|sdk-next]]; cross-chain routing from [[wiki/xc-sdk\|xc-sdk]] / [[wiki/xc-package\|xc-package]] / [[wiki/xc-swap\|xc-swap]].

## App entry point — `apps/main/src/App.tsx`

```typescript
// apps/main/src/App.tsx (excerpt)
export const App = () => (
  <I18nextProvider i18n={i18n}>
    <QueryClientProvider client={queryClient}>
      <DataProviderResolver>
        <ThemeProvider>
          <TooltipProvider>
            <RouterProvider router={router} />
            <Toaster />
          </TooltipProvider>
        </ThemeProvider>
      </DataProviderResolver>
    </QueryClientProvider>
  </I18nextProvider>
)
```

- `createRouter({ routeTree, context: { queryClient, i18n }, defaultPreload: "intent", defaultNotFoundComponent: Page404, defaultErrorComponent: RouteError, scrollRestoration: true })`; `RouterContext` is the typed router context interface, and `declare module "@tanstack/react-router"` registers the router type globally.
- `QueryClient` installs `QueryCache` / `MutationCache` `onError` handlers that `console.error("[RQ]", …)` — there is no global toast on query failure.
- A `window.addEventListener("vite:preloadError", () => window.location.reload())` guard handles stale chunk hashes after a deploy.
- `import "@galacticcouncil/ui/fonts.css"` is the first statement — fonts come from the design-system package.

Deeper walkthrough: [[wiki/hydration-ui-main-app\|hydration-ui-main-app]].

## Product modules — `apps/main/src/modules/` (13)

| Module | What it covers |
|---|---|
| `trade/` | Swap, limit/DCA orders, `orders/TradeOrdersHistory` (replaced wallet tx history) → [[omnipool\|omnipool]], [[wiki/smart-order-router\|smart-order-router]], [[wiki/dca\|dca]] |
| `liquidity/` | LP positions, farms → [[wiki/nft-lp-positions\|nft-lp-positions]] |
| `borrow/` | Aave v3 lending UI → [[wiki/hydration-ui-money-market\|hydration-ui-money-market]], [[wiki/pallet-liquidation\|pallet-liquidation]] |
| `staking/` | **[[wiki/gigahdx\|gigahdx]] liquid staking** at `/staking`; legacy HDX staking survives at `/staking-old` → [[wiki/pallet-staking\|pallet-staking]], [[wiki/pallet-gigahdx\|pallet-gigahdx]] |
| `strategies/` | **new** — BIL RWA vault (ERC-4626 + ERC-7540 async redeem) and [[wiki/hollar\|hollar]] stable bonds sold via OTC offers |
| `onramp/` | **new** — CEX (Kraken, Binance, Kucoin, Coinbase, Gate.io) + bank wizard, routed via the `assethub_cex` chain; served at `/deposit` and `/withdraw` |
| `governance/` | **new** — stub; nav entry commented out |
| `xcm/` | Cross-chain transfers, bridge selection (NTT / Snowbridge V1+V2 / Basejump) → [[wiki/xc-sdk\|xc-sdk]], [[wiki/snowbridge\|snowbridge]] |
| `wallet/` | Portfolio and balances (transaction history removed) |
| `stats/` | Analytics dashboards |
| `transactions/` | Tx construction helpers and history display |
| `submit-transaction/` | Central signing modal → [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] |
| `layout/` | App shell, navigation, banners |

Per-module detail: [[wiki/hydration-ui-modules\|hydration-ui-modules]].

## Data layer — `apps/main/src/api/`

React Query hooks by domain. Notable additions this cycle: **`gigaStake.ts`** and **`gigaApr.ts`** ([[wiki/gigahdx\|gigahdx]]) and **`grafana/`** (`fetchGrafana`, `TradeChartApi`, plus `.sql`/`.ts` pairs for `tradeChart`, `dcaAmount`, `reserveRate`) as charts migrate off squid. Existing: `aave` `account` `assets` `balances` `bonds` `borrow/` `chain` `democracy` `dryRun` `evm` `external/` `farms` `locks` `multisig` `omnipool` `otc` `payments` `pools` `provider` `referrals` `rpc` `spotPrice` `stableswap` `staking` `stats` `subscriptions` `trade` `transaction` `xcm` `xyk`. Full inventory: [[wiki/hydration-ui-api\|hydration-ui-api]].

## State — `apps/main/src/states/` (Zustand + Immer)

`account` `assetRegistry` `banners` `displayAsset` `externalApy` `liquidity` `multisigWatch` `provider` `swapForm` `toasts` `tradeSettings` `transactions`.

## Providers — `apps/main/src/providers/`

`rpcProvider.tsx` ([[wiki/papi\|papi]] client + `useRpcProvider()`), `assetsProvider.tsx`, `AssetRegistryGate.tsx` (blocks render until the registry resolves), `MultisigProvider.tsx` (multix-backed multisig context — the multix GraphQL client itself now lives in [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]]).

## Packages

| Package | Page |
|---|---|
| `packages/ui/` — design system, 76 components, Storybook 10 | [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] |
| `packages/web3-connect/` — wallets (Substrate, EVM, Solana, Sui, Near, Zcash) + address book | [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] |
| `packages/money-market/` — Aave v3 for `hydration_v3`, `gigahdx_v3`, `bil_v3` | [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] |
| `packages/indexer/` — three GraphQL clients | [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] |
| `packages/utils/` — address guards, Neckwork explorer links, formatting | [[wiki/hydration-ui-utils\|hydration-ui-utils]] |
| `packages/eslint-config/`, `packages/typescript-config/` | [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] |

### Build order

```
Level 0:  utils, indexer, eslint-config, typescript-config
Level 1:  ui           → utils
Level 2:  web3-connect → ui, utils, indexer
Level 3:  money-market → ui, utils, web3-connect
Level 4:  apps/main    → ui, utils, web3-connect, money-market, indexer
```

Packages ship **source TypeScript**, not `dist/` — every consumer re-bundles through Vite/Rolldown.

## Stack in one line

React 19.2 + Vite 8 (Rolldown) + TanStack Router/Query + Emotion on a Theme-UI theme + Zustand; papi resolved to 2.1.7, `sdk-next` 1.6.0, `descriptors` 2.6.0, `xc-*` 2.3.0; viem 2.48 + ethers 5.7 (Aave only) + Reown AppKit for EVM. No test runner, **no wagmi**. Version matrix and build-pipeline detail: [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]].

## Gotchas

- The repo's own `CLAUDE.md` / `README.md` are stale on toolchain facts (Vite 7, `vite-tsconfig-paths`, Node 22.13, CI Node 20) — [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] is the corrected version.
- `MoneyMarketProvider` changed shape (`market: CustomMarket`, not `env: MoneyMarketEnv`) — any older snippet will not compile.
- Money-market APY fields are `number | null` now; unguarded arithmetic on them is a live runtime hazard ([[wiki/hydration-ui-money-market\|hydration-ui-money-market]]).
- `@galacticcouncil/ui` has **directory-level** exports only — `@galacticcouncil/ui/components`, never `@galacticcouncil/ui/button`.

## For developers

1. [[wiki/hydration-ui-modules\|hydration-ui-modules]] — find the feature
2. [[wiki/hydration-ui-api\|hydration-ui-api]] — its data hooks
3. `src/states/` — its client state
4. [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] — how it signs and submits
5. [[wiki/papi\|papi]] / [[wiki/sdk-next\|sdk-next]] / [[wiki/xc-sdk\|xc-sdk]] — chain-level semantics

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration\|hydration]], [[wiki/papi\|papi]], [[wiki/sdk-next\|sdk-next]], [[wiki/xc-sdk\|xc-sdk]]
