---
{"dg-publish":true,"permalink":"/wiki/hydration-ui/","title":"hydration-ui","tags":["ui","frontend","hydration","react","typescript","monorepo"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"hydration-ui","repo":"hydration-ui","paths":["apps/main/src/App.tsx","apps/main/src/routes/__root.tsx","apps/main/src/modules","apps/main/src/api","apps/main/src/states","apps/main/src/providers","packages/ui/src","packages/web3-connect/src","packages/money-market/src","packages/indexer/src","packages/utils/src","AGENTS.md"],"key_exports":["App","RouterContext","ThemeProvider","MoneyMarketProvider","useAccount","useRpcProvider","neckworkClient","useIndexerClient","useMultixClient"],"symbols":["DataProviderResolver","AssetRegistryGate","MultisigProvider","TutorialProvider","RpcProvider","routeTree","TProviderContext"],"tags":["ui","frontend","hydration","react","typescript","monorepo"],"last_updated":"2026-11-02"}}
---


# hydration-ui

**TL;DR:** Frontend monorepo for [[wiki/hydration\|hydration]] — one deployable app (`apps/main`) plus 7 workspace packages. At `770f868` (2026-11-01): 14 feature modules; Market swap tab runs on [[wiki/xc-swap\|xc-swap]] (on-chain + cross-chain), Limit orders are [[wiki/ice\|ice]] intents, `/portfolio` (multichain + tracked wallets) replaced the wallet page, Gamma/Uniswap V3 vaults sit in liquidity, history data comes from the **Neckwork** REST API, and wallets cover Substrate, EVM, Solana, Sui, NEAR and Zcash.

## Role

User-facing surface for trading ([[omnipool\|omnipool]], [[wiki/stableswap\|stableswap]], [[wiki/otc-trading\|otc-trading]], [[wiki/dca\|dca]], [[wiki/ice\|ice]]), liquidity (incl. V3 vaults), lending ([[wiki/hydration-borrow\|hydration-borrow]]), [[wiki/gigahdx\|gigahdx]] liquid staking, strategies (BIL vault, [[wiki/hollar\|hollar]] bonds), cross-chain transfers and swaps, CEX/bank on/off-ramp, portfolio and fee stats. Chain access is [[wiki/papi\|papi]]; trade construction and pool math come from [[wiki/sdk-next\|sdk-next]]; cross-chain from [[wiki/xc-sdk\|xc-sdk]] / [[wiki/xc-package\|xc-package]] / [[wiki/xc-swap\|xc-swap]].

## Entry & providers

`App.tsx` → QueryClient (+ portfolio persistence) → router; `routes/__root.tsx` stacks `AssetsProvider → RpcProvider → TutorialProvider → MultisigProvider → AssetRegistryGate` and mounts `Services` (tx manager, wallet session/modal, block/price/indexer subscriptions). Detail: [[wiki/hydration-ui-main-app\|hydration-ui-main-app]].

## Modules — `apps/main/src/modules/`

| Module | Covers |
|---|---|
| `trade/` | Market (XcSwap), Limit (ICE intents), TWAP/DCA, OTC, order history |
| `liquidity/` | Omnipool / stableswap / isolated / money-market LP, farms, **V3 vaults** |
| `borrow/` | Aave v3 money market → [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] |
| `strategies/` | BIL RWA vault, Hollar stable bonds (incl. rollover) |
| `portfolio/` | Overview (assets, liquidity, bonds, rewards) + tracked wallets |
| `onramp/` | `/deposit`, `/withdraw` CEX + bank wizard |
| `xcm/` | Cross-chain transfer + scan-backed history and claims |
| `staking/` | GigaHDX at `/staking`; legacy at `/staking-old` |
| `stats/` | Fees (other stats routes are empty placeholders) |
| `transactions/`, `submit-transaction/` | Review / sign / submit → [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] |
| `tutorials/`, `layout/` | First-run hints; shell, breadcrumbs, nav |
| `governance/` | Not routed (dead until wired) |

Detail: [[wiki/hydration-ui-modules\|hydration-ui-modules]].

## Data layer — `apps/main/src/api/`

`queryOptions` factories + hooks per domain, gated on `isReady` from `useRpcProvider()`. Sources: papi (`papi` / `papiNext`), sdk-next, **Neckwork** (`neckworkClient`, `useNeckworkSync`), transfers/intents scan apps, GraphQL indexer (farms), Grafana (reserve rates), external APIs (Kraken, DefiLlama, Kamino, Subsquare). Inventory: [[wiki/hydration-ui-api\|hydration-ui-api]], clients: [[wiki/hydration-ui-indexer\|hydration-ui-indexer]].

## Packages

| Package | Page |
|---|---|
| `packages/ui/` — 77 components, Storybook 10, `@tanstack/charts` | [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] |
| `packages/web3-connect/` — wallets, signers, address book, multisig | [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] |
| `packages/money-market/` — Aave v3 for `hydration_v3`, `gigahdx_v3`, `bil_v3` | [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] |
| `packages/indexer/` — `/neckwork`, `/scan`, `/indexer`, `/multix` | [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] |
| `packages/utils/` — address guards, explorer links, intl, constants | [[wiki/hydration-ui-utils\|hydration-ui-utils]] |
| `packages/eslint-config/`, `packages/typescript-config/` | [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] |

Build order: `utils, indexer, *-config` → `ui` → `web3-connect` → `money-market` → `apps/main`. Packages ship source TS; the app type-checks them via `packages/<name>/build/`, so rebuild a package (`npx tsc --build`) after editing it.

## Stack in one line

React 19.2 + Vite 8 (Rolldown) + TanStack Router/Query + Emotion/Theme-UI + Zustand; papi 2.2.2, sdk-next 2.5.0, descriptors 2.9.0, xc-* 2.x, xc-swap 0.9; viem 2.56 + ethers 5.7 (Aave only) + Reown AppKit; Vitest; Node 26. Detail: [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]].

## Gotchas

- `useRpcProvider()`'s default context is load-bearing (boot skeleton renders outside the provider) — never make it throw.
- Account queries that read Neckwork must key under `NECKWORK_ACCOUNT_KEY` or they won't refresh after a tx.
- Money-market APY fields are `number | null`.
- `@galacticcouncil/ui` has directory-level exports only (`@galacticcouncil/ui/components`).
- `utils` and `money-market` import `@galacticcouncil/xc-*` / `indexer` without declaring them (root hoisting).

## For developers

1. Repo `AGENTS.md` / `CODING_STANDARDS.md` (current) — workflow and per-file-kind rules
2. [[wiki/hydration-ui-modules\|hydration-ui-modules]] → find the feature
3. [[wiki/hydration-ui-api\|hydration-ui-api]] → its data hooks
4. [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] → how it signs and submits
5. [[wiki/papi\|papi]] / [[wiki/sdk-next\|sdk-next]] / [[wiki/xc-sdk\|xc-sdk]] → chain-level semantics

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration\|hydration]], [[wiki/papi\|papi]], [[wiki/sdk-next\|sdk-next]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-swap\|xc-swap]]
