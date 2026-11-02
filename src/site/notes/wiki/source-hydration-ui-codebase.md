---
{"dg-publish":true,"permalink":"/wiki/source-hydration-ui-codebase/","title":"hydration-ui codebase","tags":["hydration","frontend","react","typescript"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"source","title":"hydration-ui codebase","source_kind":"repo_clone","raw_path":"raw/hydration-ui/","upstream":"https://github.com/galacticcouncil/hydration-ui","cloned_at":"2026-04-20","last_refreshed":"2026-11-02","last_commit":"770f868 Merge pull request #4173 from galacticcouncil/feat/ipfs-deploy (2026-11-01)","previous_commit":"b469e8a Merge pull request #4021 from galacticcouncil/fix/wrong-route-tag (2026-08-12)","produces_pages":["hydration-ui","hydration-ui-main-app","hydration-ui-api","hydration-ui-modules","hydration-ui-submit-tx","hydration-ui-web3-connect","hydration-ui-design-system","hydration-ui-money-market","hydration-ui-indexer","hydration-ui-utils","hydration-ui-tech-stack"],"tags":["hydration","frontend","react","typescript"],"last_updated":"2026-11-02","summary":"b469e8a → 770f868 (2026-08-12 → 2026-11-01): 477 commits, 1376 files, +68408/−145409 (most deletions = retired squid codegen). Headline: Market swap = XcSwap (on-chain + cross-chain via xc-swap); Limit tab = ICE intents; wallet/ module deleted → portfolio/ (multichain + tracked wallets); squid GraphQL retired → Neckwork REST (openapi-fetch) + scan apps; Gamma/Uniswap V3 vaults in liquidity; NEAR + Zcash signing; web3-connect modal v2; tutorials; recharts → @tanstack/charts; Vitest in CI; IPFS deploy; papi 2.2.2, sdk-next 2.5, descriptors 2.9, xc-swap 0.9; Node 26."}}
---


# hydration-ui codebase

**TL;DR:** Yarn 1 workspace monorepo (Turborepo) — one deployable app (`apps/main/`, Vite 8 + React 19 + TanStack Router) and 7 packages, all `private: true`, exporting raw TypeScript from `src/`. Snapshot at **`770f868` (2026-11-01)**; `.vault-clone-info` in `raw/hydration-ui/` records the sha.

## Clone metadata

| Field | Value |
|---|---|
| Upstream | `github.com/galacticcouncil/hydration-ui`, branch `master` |
| HEAD | `770f868` — "Merge pull request #4173 … feat/ipfs-deploy", 2026-11-01 |
| Previous sync | `b469e8a` (2026-08-12) — 477 commits, 1376 files |
| Node | `.nvmrc` `v26.12.1`; CI uses `.nvmrc` |
| Agent docs | `AGENTS.md` (CLAUDE.md → `@AGENTS.md`), `CODING_STANDARDS.md` |

### Change volume by area (`b469e8a..770f868`)

| Path | Files | Page |
|---|---|---|
| `apps/main/src/modules` | 729 | [[wiki/hydration-ui-modules\|hydration-ui-modules]] |
| `packages/ui` | 163 | [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] |
| `packages/web3-connect` | 143 | [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] |
| `apps/main/src/api` | 68 | [[wiki/hydration-ui-api\|hydration-ui-api]] |
| `apps/main/src/components` | 66 | [[wiki/hydration-ui-main-app\|hydration-ui-main-app]] |
| `packages/indexer` | 56 | [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] |
| `apps/main/src/routes` | 27 | [[wiki/hydration-ui-modules\|hydration-ui-modules]] |
| `packages/utils` | 25 | [[wiki/hydration-ui-utils\|hydration-ui-utils]] |
| `packages/money-market` | 21 | [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] |
| `apps/main/src/states` | 14 | [[wiki/hydration-ui-main-app\|hydration-ui-main-app]] |

## File tree (top levels)

```
raw/hydration-ui/
├── apps/main/
│   ├── src/
│   │   ├── App.tsx  index.tsx  routeTree.gen.ts (generated)
│   │   ├── api/          # query layer: rpcClient, balances/, portfolio/, gamma/, borrow/, grafana/, external/, neckwork, scan, …
│   │   ├── components/   config/   form/   hooks/   i18n/   providers/   states/   styles/   types/   utils/   workers/
│   │   ├── modules/      # 14 feature folders
│   │   └── routes/       # TanStack file-based routes
│   ├── vite.config.ts  vitest.config.ts  netlify.toml
├── packages/
│   ├── ui/               # 77 components, theme, style-dictionary, scripts/, .storybook
│   ├── web3-connect/     # components config hooks signers utils wallets
│   ├── money-market/     # vendored Aave v3 UI
│   ├── indexer/          # indexer/ multix/ neckwork/ scan/
│   ├── utils/  eslint-config/  typescript-config/
├── package.json  turbo.json  .nvmrc  AGENTS.md  CODING_STANDARDS.md
└── .github/workflows/ci.yml  ipfs.yml
```

`apps/main/src/modules/` (14): `borrow` `governance` (unrouted) `layout` `liquidity` `onramp` `portfolio` `staking` `stats` `strategies` `submit-transaction` `trade` `transactions` `tutorials` `xcm`.

## Workspace packages

| Name | Purpose | Page |
|---|---|---|
| `@galacticcouncil/ui` | Radix + Emotion design system, Theme-UI theme, tokens, Storybook 10 | [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] |
| `@galacticcouncil/web3-connect` | Wallets (Substrate, EVM, Solana, Sui, NEAR, Zcash), signers, address book, multisig | [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] |
| `@galacticcouncil/money-market` | Aave v3 UI for `hydration_v3`, `gigahdx_v3`, `bil_v3` | [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] |
| `@galacticcouncil/indexer` | Neckwork REST, scan apps, GraphQL indexer (farms), multix | [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] |
| `@galacticcouncil/utils` | Address guards, explorer links, intl, constants, hooks | [[wiki/hydration-ui-utils\|hydration-ui-utils]] |
| `@galacticcouncil/eslint-config`, `typescript-config` | Shared lint / TS presets | [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] |

## SDK / protocol dependencies (workspace root `package.json`)

```json
"resolutions": { "polkadot-api": "^2.1.7", "typescript": "5.7.3", "strip-ansi": "6.0.1" },
"dependencies": {
  "@galacticcouncil/common": "^1.2.0", "@galacticcouncil/descriptors": "^2.9.0",
  "@galacticcouncil/sdk-next": "^2.5.0", "@galacticcouncil/xc": "^2.1.0",
  "@galacticcouncil/xc-cfg": "^2.9.1", "@galacticcouncil/xc-core": "^2.8.0",
  "@galacticcouncil/xc-scan": "^0.5.0", "@galacticcouncil/xc-sdk": "^2.7.0",
  "@galacticcouncil/xc-swap": "^0.9.0", "@polkadot-api/tx-utils": "^0.2.2",
  "polkadot-api": "^2.1.0", "zod": "^4.1.11", "zustand": "^5.0.1"
}
```

`polkadot-api` resolves to **2.2.2**. Full matrix: [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]].

## What changed b469e8a → 770f868

- **Trade:** Market tab = `XcSwap` ([[wiki/xc-swap\|xc-swap]]: on-chain quotes via `sdk.api.router`, cross-chain quotes incl. NEAR/Zcash destinations); Limit tab = [[wiki/ice\|ice]] intents (`sdk.tx.intentLimit`), order history from Neckwork intents/DCA/trades.
- **Portfolio replaces wallet:** `modules/wallet/` deleted (90 files); `/wallet*` redirect to `/portfolio`; multichain balances + tracked wallets; portfolio query persistence.
- **Data:** squid client retired (−44.5k generated lines); new Neckwork REST client (`openapi-fetch`) and scan-app client; Basejump scan SSE client replaced by the generic scan stream; Grafana only for reserve rates; GIGAHDX APR from Neckwork.
- **Liquidity:** Gamma-managed Uniswap V3 vaults (`/liquidity/vault/$address`, add/remove/transfer).
- **Wallets:** web3-connect modal v2 (wallet management screens, `Web3ConnectSession`), NEAR (near-connect) and Zcash (Noir) signers, Backpack / OKX / Trust / Ledger etc.
- **UI kit:** recharts → `@tanstack/charts`; emoji account avatars; `Hint`, `InputSuggestions`, `Popover`; `Combobox`, `SliderTabs` removed.
- **App:** breadcrumbs + user menu, tutorials, app-update banner, Vitest in CI, IPFS deploy workflow.

## Sources

[[wiki/hydration\|hydration]], [[wiki/papi\|papi]], [[wiki/sdk-next\|sdk-next]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-swap\|xc-swap]]
