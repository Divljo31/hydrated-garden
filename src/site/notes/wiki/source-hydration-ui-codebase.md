---
{"dg-publish":true,"permalink":"/wiki/source-hydration-ui-codebase/","title":"hydration-ui codebase","tags":["hydration","frontend","react","typescript"],"dg-note-properties":{"type":"source","title":"hydration-ui codebase","source_kind":"repo_clone","raw_path":"raw/hydration-ui/","upstream":"https://github.com/galacticcouncil/hydration-ui","cloned_at":"2026-04-20","last_refreshed":"2026-04-20","last_commit":"270eaae Merge pull request #3740 from galacticcouncil/fix/issue-3616 (2026-04-17)","produces_pages":["hydration-ui","hydration-ui-main-app","hydration-ui-api","hydration-ui-modules","hydration-ui-submit-tx","hydration-ui-web3-connect","hydration-ui-design-system","hydration-ui-money-market","hydration-ui-indexer","hydration-ui-utils","hydration-ui-tech-stack"],"tags":["hydration","frontend","react","typescript"],"last_updated":"2026-04-20","summary":"87 files modified (mostly borrow dashboard, staking, trade, transactions/review, xcm, layout notifications); 1 new dependency (@polkadot-api/tx-utils); 2 new components (MultisigNotification, ReviewMultisig)"}}
---


# hydration-ui codebase

**TL;DR:** Yarn monorepo (Turbo) with the primary web app (`apps/main/` — Vite + React 19 + Tanstack Router) and 6 packages (design system, wallet abstraction, Aave lending hooks, GraphQL clients, utilities, ESLint config).

## File tree (top 3 levels)

```
raw/hydration-ui/
├── apps/
│   └── main/
│       ├── src/
│       │   ├── App.tsx                     # Root app
│       │   ├── api/                        # React-query domain hooks
│       │   ├── components/                 # Shared app components
│       │   ├── config/
│       │   ├── form/
│       │   ├── hooks/
│       │   ├── i18n/
│       │   ├── modules/                    # Product modules
│       │   ├── providers/                  # rpcProvider, assetsProvider
│       │   ├── routes/                     # Tanstack Router (file-based)
│       │   ├── states/                     # Zustand stores
│       │   ├── styles/
│       │   ├── utils/
│       │   └── workers/                    # Comlink workers
│       └── vite.config.ts
├── packages/
│   ├── ui/                                 # Design system (Radix + Emotion)
│   ├── web3-connect/                       # Wallet adapter abstraction
│   ├── money-market/                       # Aave v3 hooks
│   ├── indexer/                            # GraphQL clients
│   ├── utils/                              # Shared helpers
│   ├── eslint-config/                      # Shared ESLint
│   └── typescript-config/                  # Shared tsconfig
├── package.json                            # Workspace root + resolutions
├── turbo.json
└── .nvmrc
```

## Stack summary

| Layer | Technology |
|-------|------------|
| **App framework** | React 19, Vite 7 |
| **Routing** | Tanstack Router (file-based, src/routes/) |
| **Data fetching** | Tanstack Query v5, react-query-devtools |
| **State** | Zustand (Immutable pattern w/ immer), react-use, react-hook-form, Zod validation |
| **Styling** | Emotion CSS-in-JS, Radix UI (design system) |
| **i18n** | i18next + react-i18next |
| **Async/Workers** | comlink (Web Workers) |
| **Notifications** | sonner (toasts) |
| **EVM** | viem, ethers, @aave/contract-helpers |
| **Crypto libs** | big.js, @noble/hashes, uuid |
| **Charts/Data** | recharts, lightweight-charts, react-number-format |
| **Icons** | lucide-react, jdenticon (identicons), react-jazzicon |

## Workspace packages

| Name | Purpose |
|------|---------|
| `@galacticcouncil/ui` | Radix-based design system; Emotion styling; style-dictionary token generation; Storybook |
| `@galacticcouncil/web3-connect` | Wallet connection abstraction supporting PJS extensions, WalletConnect, EVM wallets |
| `@galacticcouncil/money-market` | Aave v3 integration: contract-helpers + math-utils + React Query hooks |
| `@galacticcouncil/indexer` | Three GraphQL clients: indexer, squid, snowbridge; uses graphql-codegen |
| `@galacticcouncil/utils` | Small helpers: UUID, hashing, duration formatting, buffers |
| `@galacticcouncil/eslint-config` | Shared ESLint rules (workspace) |
| `@galacticcouncil/typescript-config` | Shared tsconfig (workspace) |

## SDK/Protocol dependencies (workspace root)

```json
{
  "polkadot-api": "1.23.3",
  "@galacticcouncil/sdk-next": "^0.41.0",
  "@galacticcouncil/descriptors": "^1.16.0",
  "@galacticcouncil/xc": "^0.6.0",
  "@galacticcouncil/xc-cfg": "^0.19.0",
  "@galacticcouncil/xc-core": "^0.14.0",
  "@galacticcouncil/xc-scan": "^0.5.0",
  "@galacticcouncil/xc-sdk": "^0.10.1",
  "@galacticcouncil/common": "^0.7.0"
}
```

## Key runtime characteristics

- **Polkadot API pinning:** `polkadot-api@1.23.3` pinned via workspace resolutions (critical for pallet-specific type generation)
- **Module structure:** Each product domain (`trade`, `liquidity`, `borrow`, `xcm`, `wallet`, `staking`, `stats`) is a self-contained module in `src/modules/`
- **Data layer:** React Query hooks colocated in `src/api/` by domain (omnipool, stableswap, aave, etc.)
- **Stores:** Zustand-based state in `src/states/` — account, assetRegistry, displayAsset, liquidity, provider, toasts, tradeSettings, transactions
- **Tx submission:** Centralized `submit-transaction` module handles signing, signer selection, status tracking
- **i18n:** Full i18n coverage via `src/i18n/` (config for supported locales)
- **Design tokens:** Generated by `yarn theme` in ui package (style-dictionary)

## Sources

[[wiki/hydration\|hydration]], [[wiki/papi\|papi]]
