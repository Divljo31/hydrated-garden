---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-main-app/","title":"hydration-ui main app","tags":["ui","frontend","react","routing","providers","state"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui main app","repo":"hydration-ui","paths":["apps/main/src/App.tsx","apps/main/src/routes/__root.tsx","apps/main/src/routes","apps/main/src/config/navigation.ts","apps/main/src/config/env.ts","apps/main/src/config/portfolio.ts","apps/main/src/config/xcSwap.ts","apps/main/src/providers/rpcProvider.tsx","apps/main/src/providers/AssetRegistryGate.tsx","apps/main/src/states","apps/main/src/hooks","apps/main/src/i18n","apps/main/src/utils/appUpdate.ts","apps/main/src/utils/deploy.ts","apps/main/vite.config.ts"],"symbols":["App","RouterContext","routeTree","RootComponent","Services","RpcProvider","useRpcProvider","AssetsProvider","AssetRegistryGate","MultisigProvider","TutorialProvider","AppUpdateBanner","LINKS","ENV","PORTFOLIO_CHAINS","TRACKED_CHAINS","XC_SWAP_CONFIG","setupPortfolioPersistence","applyResetTutorialsParam","getDeployType","useNeckworkSync","useScanJourneysSubscription","useReloadOnStaleBlocks"],"tags":["ui","frontend","react","routing","providers","state"],"last_updated":"2026-11-02"}}
---


# hydration-ui main app

**TL;DR:** `apps/main` — the only deployable: a React 19 SPA (Vite 8 / Rolldown) with TanStack file-based routing. `App.tsx` builds the QueryClient + router; `routes/__root.tsx` stacks the providers (`AssetsProvider → RpcProvider → TutorialProvider → MultisigProvider → AssetRegistryGate`) and mounts background `Services`. Deployed to Netlify and, since `770f868`, also pinned to **IPFS** on every `master` push.

## `App.tsx`

```typescript
// apps/main/src/App.tsx (abridged)
import "@galacticcouncil/ui/fonts.css"
import "@/utils/appUpdate"
const queryClient = new QueryClient({ queryCache: new QueryCache({ onError: (e, q) => console.error("[RQ]", e, q) }), /* + MutationCache */ })
setupPortfolioPersistence(queryClient)   // api/portfolio/persistence.ts
applyResetTutorialsParam()               // ?resetTutorials
const router = createRouter({
  routeTree, context: { queryClient, i18n },
  defaultPreload: "intent", defaultPreloadStaleTime: 0, defaultPendingMs: 0,
  defaultNotFoundComponent: Page404, defaultErrorComponent: RouteError,
  scrollRestoration: true, scrollRestorationBehavior: "smooth",
})
export const App = () => { useRemoveInitialLoader(); return (
  <I18nextProvider i18n={i18n}><QueryClientProvider client={queryClient}>
    <DataProviderResolver><ThemeProvider><TooltipProvider>
      <RouterProvider router={router} /> {/* + <Toaster /> */}
```

No global toast on query errors. `utils/appUpdate.ts` polls the deployed `index.html` (hourly + on refocus) and compares the entry-script hash; a mismatch shows `AppUpdateBanner`.

## Root route — `routes/__root.tsx`

```tsx
<HeadContent />
<AssetsProvider><RpcProvider><TutorialProvider><MultisigProvider><AssetRegistryGate>
  <MainLayout /> <Services /> <Footer /> <AppUpdateBanner /> {/* MobileTabBar when no top navbar */}
</AssetRegistryGate></MultisigProvider></TutorialProvider></RpcProvider></AssetsProvider>
```

- `pendingComponent` = `LayoutSkeleton` inside `AssetsProvider` only — it renders **outside** `RpcProvider`, which is why `useRpcProvider()` must keep its default value ([[wiki/hydration-ui-api\|hydration-ui-api]]).
- `Services`: lazy `TransactionManager`, `Web3ConnectSession`, `Web3ConnectModal` (gets `neckworkClient` + `papi`); when `isReady` → `ApiSubscriptions` (`useInvalidateOnBlock`, `useReloadOnStaleBlocks`, `usePriceSubscriber`, `useNeckworkSync`); when connected → `AccountSubscriptions` (`useAccountBalances`, `useScanJourneysSubscription`, `useXcScanSubscription`).
- `Devtools` lazy in dev only.

## Routing

File-based (`@tanstack/router-plugin`), generated `routeTree.gen.ts`. Pathless `_history` layout under `/trade` hosts swap tabs and history. Full map in [[wiki/hydration-ui-modules\|hydration-ui-modules]]. Routes declare `staticData.crumb` for the breadcrumb bar and `head` → `getPageMeta(key, t)`.

`config/navigation.ts`: `LINKS` (typed against `FileRouteTypes["to"]`), nav tree (`trade` → swap market/limit/twap + otc; `borrow` → dashboard/markets/history; `strategies` → bil/hollar bonds; `liquidity`; `portfolio` → overview/tracked; `crossChain` → transfer/history; `stats` → overview/treasury/money-market/hollar/amm/fees; `staking` → GigaStake/old; external `explorer`), `EXTERNAL_LINKS.explorer = neckwork.base`. Commented-out keys: `bridge`, `governance`, `referrals`, `memepad`. `/wallet`, `/wallet/assets` exist only as redirects to `/portfolio`.

## Config

| File | Contents |
|---|---|
| `env.ts` | zod-parsed `ENV`: `VITE_PROVIDER_URL`, `VITE_INDEXER_URL`, `VITE_NECKWORK_URL`, `VITE_TRANSFERS_SCAN_URL`, `VITE_GRAFANA_URL`, `VITE_GRAFANA_DSN`, `VITE_ENV`, `VITE_TRSRY_ADDR`, `VITE_EVM_CHAIN_ID`, `VITE_DISPLAY_ASSET_ID`, flags `VITE_HSM_ENABLED`, `VITE_DRY_RUN_ENABLED`, `VITE_WORMHOLE_DISABLED`, `VITE_BASEJUMP_DISABLED`, `VITE_SNOWBRIDGE_DISABLED`, `VITE_SNOWBRIDGE_INBOUND_DISABLED` |
| `rpc.ts` | `PROVIDERS` (name, url, env list) — also read by `vite.config.ts` to inline the RPC-ping script |
| `portfolio.ts` | `PORTFOLIO_CHAINS` (ethereum, robinhood, base, solana, hyperevm, sui, near, assethub, bifrost), `TRACKED_CHAINS`, cache buster / 24 h max age |
| `xcSwap.ts` | `XC_SWAP_CONFIG` — emitter `0x98f1…93f2`, receiver `0x2173…63c8`, default pair HOLLAR → HDX |
| `navigation.ts`, `links.ts`, `seo.ts` | nav, external links, SEO meta |

Runtime feature flags (`TFeatureFlags` in `api/rpcClient.ts`): `hollarBondsEnabled`, `bilEnabled`, `isIceEnabled` — all `true` once the RPC data loads.

## Providers — `src/providers/`

- `rpcProvider.tsx` — `RpcProvider` / `useRpcProvider()` → `TProviderContext` (`papi`, `papiNext`, `sdk`, `evm`, `featureFlags`, `isReady`, `isEndpointSettled`, `isFork`, …). RPC switching goes through `switchRpc` (`api/rpcClient.ts`).
- `assetsProvider.tsx`, `AssetRegistryGate.tsx` (blocks render until the registry resolves), `MultisigProvider.tsx`.

## Zustand stores — `src/states/`

`account`, `assetRegistry`, `banners`, `displayAsset`, `externalApy`, **`intents`** (ICE opt-in, `useIsIceEnabled`), `liquidity`, `multisigWatch`, **`neckwork`** (indexer health probe + sync arming), `provider`, **`recentProviderAccounts`**, `swapForm`, `toasts`, **`trackedWallets`**, `tradeSettings`, `transactions` (`BRIDGE_PROVIDER_TAGS`), **`tutorials`**.

## Hooks — `src/hooks/`

`usePapiValue`, `usePapiEntries`, `useObservable`, `useObservableQuery` (rxjs bridges), data-table URL state (`useDataTableUrlPagination` / `Search` / `Sorting`), `useCsvDownload`, `useCountdown`, `useRelativeDate`, `useRouteBlock`, `useSaveFormOnChange`, `useVisibleElements`, `useIsOverlayOpen`, `useAssetColor`, `useDebouncedValue`, `useIsActiveQueries`, `data/`.

## i18n — `src/i18n/`

i18next; hand-edited `locales/en/{common,borrow,liquidity,onramp,staking,stats,strategies,trade,wallet,xcm}.json`; MDX explainers in `content/` (`bil-vault`, `hollar-bonds-25-08-26`, `propeller-vault`). `yarn i18n` is a placeholder.

## Workers & utils

`workers/ping/` (RPC latency worker, comlink). `utils/`: `rpc-ping.js` + `head.js` (inlined into `index.html` by `vite-plugin-html`), `appUpdate.ts`, `deploy.ts` (`getDeployType`: localhost / testnet / paseo / edge / hollarnet / deploy-preview), `indexedDB.ts`, `rxjs.ts`, `trade.ts`, `validators.ts`, `formatApyPercent.ts`.

## Build — `apps/main/vite.config.ts`

Vite 8 (Rolldown): `@vitejs/plugin-react` + `@rolldown/plugin-babel` (Emotion), `@tanstack/router-plugin`, `@mdx-js/rollup` (+ remark-gfm), `vite-plugin-svgr`, `vite-plugin-wasm`, `vite-plugin-html` (inlines head script, RPC ping with `PROVIDERS` URLs filtered by `VITE_ENV`, loader HTML, critical CSS), a custom `fontPreload` plugin (Geist / Gazpacho woff2), `@tanstack/devtools-vite` outside production. `resolve.tsconfigPaths: true`, `build.target: es2022`, `outDir: build`.

**IPFS deploy** (`.github/workflows/ipfs.yml`): on push to `master`, build → `ipfs add -r --cid-version=1 apps/main/build` in a Kubo container → export CAR → pin to two IPFS pinning services. Netlify remains the primary host (deploy previews detected by hostname).

## Shared components — `src/components/`

`AssetSelect`/`AssetSelectModal`, `AssetLogo`, `AssetAmount`, `AssetPrice`, `ChainLogo`, `AccountIdentity`, `AddressTag`, `AuthorizedAction`, `BondRedeemButton`, `Breadcrumb`, `ChartState`, `ChartTimeRange`, `DataProviderSelect` (RPC picker; `DataProviderResolver` wraps the app), `DetailedApy`, `DynamicFee`, `GigaNews`, `Markdown`, `PendingApproval`, `RouteError`, `Page404`, `TradeFee`, `TransactionItem`, `TransactionStatus`, `AppUpdateBanner`, `Devtools`.

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]], [[wiki/papi\|papi]], [[wiki/sdk-next\|sdk-next]]
