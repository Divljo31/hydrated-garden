---
{"dg-publish":true,"permalink":"/wiki/source-hydration-ui-codebase/","title":"hydration-ui codebase","tags":["hydration","frontend","react","typescript"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"source","title":"hydration-ui codebase","source_kind":"repo_clone","raw_path":"raw/hydration-ui/","upstream":"https://github.com/galacticcouncil/hydration-ui","cloned_at":"2026-04-20","last_refreshed":"2026-08-15","last_commit":"b469e8a Merge pull request #4021 from galacticcouncil/fix/wrong-route-tag (2026-08-12)","previous_commit":"90d653f Merge pull request #3801 from galacticcouncil/fix/external-account-url (2026-05-13)","produces_pages":["hydration-ui","hydration-ui-main-app","hydration-ui-api","hydration-ui-modules","hydration-ui-submit-tx","hydration-ui-web3-connect","hydration-ui-design-system","hydration-ui-money-market","hydration-ui-indexer","hydration-ui-utils","hydration-ui-tech-stack"],"tags":["hydration","frontend","react","typescript"],"last_updated":"2026-08-15","summary":"90d653f → b469e8a (2026-05-13 → 2026-08-12): 613 commits, 780 files, +38269/−12905. Headline: /staking replaced by the GigaHDX liquid-staking UI (legacy → /staking-old, feature flag removed, new api/gigaStake.ts + api/gigaApr.ts); three new modules (strategies/ BIL RWA vault + Hollar bonds, onramp/ CEX+bank wizard, governance/ stub) taking the count to 13; Wormhole MRL → NTT across the transfer stack; Snowbridge removed at every level (GraphQL client + schema + codegen + utils/helpers/snowbridge.ts) leaving three GraphQL clients; charts migrating squid → Grafana SQL (api/grafana/); wallet tx history replaced by trade/orders + Neckwork explorer deep links (Neckwork also replaced Subscan as default explorer); money-market now serves three markets (hydration_v3, gigahdx_v3, bil_v3) with a breaking MoneyMarketProvider signature; web3-connect gained a persisted address book and Near + Zcash wallet modes; packages/ui grew six components (Prose, PromoteBanner, OptionCard, PositionCard, EditableText, MorphLabel) to 76. Dep moves: papi resolution
{ #2}
.1.0 →
{ #2}
.1.7, sdk-next 1.0.1 → 1.6.0, descriptors 2.0 → 2.6, xc-* 1.x → 2.x, TanStack Router 1.139 → 1.170, Query 5.59 → 5.100, new MDX + TanStack Devtools Vite plugins. Root CLAUDE.md and README.md changed only to drop Snowbridge mentions and are otherwise stale (Vite 7, vite-tsconfig-paths, Node 22.13, CI Node 20)."}}
---


# hydration-ui codebase

**TL;DR:** Yarn 1 workspace monorepo (Turborepo) — one deployable app (`apps/main/`, Vite 8 + React 19 + TanStack Router) and 7 packages. Nothing is published to npm; every workspace is `private: true` at version `0.0.0` and exports raw TypeScript from `src/`.

## Clone metadata

| Field | Value |
|---|---|
| Upstream | `github.com/galacticcouncil/hydration-ui` (private) |
| HEAD at this sync | **`b469e8a`** — "Merge pull request #4021 from galacticcouncil/fix/wrong-route-tag", **2026-08-12** |
| Previous sync | `90d653f` (2026-05-13) |
| Delta | **613 commits, 780 files changed, +38 269 / −12 905** |
| Default branch | `master` |
| Node | `.nvmrc` `v25.9.0`; CI `25.x` |

### Change volume by area

| Path | Files | Notes |
|---|---|---|
| `apps/main/src/modules` | 428 | 3 new modules; staking rewritten — [[wiki/hydration-ui-modules\|hydration-ui-modules]] |
| `packages/ui` | 100 | 6 new components, no deletions — [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] |
| `packages/web3-connect` | 41 | address book, Near/Zcash — [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] |
| `packages/money-market` | 28 | three markets — [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] |
| `packages/indexer` | 23 | **−6 271 lines** (Snowbridge schema) — [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] |
| `packages/utils` | 17 | Neckwork, multix, invariant — [[wiki/hydration-ui-utils\|hydration-ui-utils]] |
| `apps/main/src/api` | 33 | gigaStake, gigaApr, grafana — [[wiki/hydration-ui-api\|hydration-ui-api]] |
| `apps/main/src/components` | 38 | |
| `apps/main/src/states` | 9 | |
| `apps/main/src/routes` | 17 | `/staking` ↔ `/staking-old` swap, new `/strategies`, `/deposit` + `/withdraw` (onramp) |

## File tree (top 3 levels)

```
raw/hydration-ui/
├── apps/
│   └── main/
│       ├── src/
│       │   ├── App.tsx          index.tsx
│       │   ├── api/             # papi/SDK query layer (+ grafana/, borrow/, external/, utils/)
│       │   ├── components/      # app-level shared components
│       │   ├── config/          # rpc, seo, env
│       │   ├── form/            # react-hook-form wrappers
│       │   ├── hooks/
│       │   ├── i18n/            # i18next + locales + content/*.mdx
│       │   ├── modules/         # 13 feature modules (see below)
│       │   ├── providers/
│       │   ├── routes/          # TanStack Router file-based
│       │   ├── states/          # Zustand stores
│       │   ├── styles/          # incl. critical.css (inlined)
│       │   ├── types/  utils/  workers/
│       │   └── *.d.ts           # i18next.d.ts, intl.d.ts, mdx.d.ts, vite-env.d.ts
│       ├── vite.config.ts       # Vite 8 / Rolldown
│       └── netlify.toml
├── packages/
│   ├── ui/                      # 76 components, theme, style-dictionary, .storybook
│   ├── web3-connect/            # components config context hooks i18n signers types utils wallets
│   ├── money-market/            # components helpers hooks libs services store types ui-config utils
│   ├── indexer/                 # indexer/ squid/ multix/   (snowbridge/ DELETED)
│   ├── utils/                   # constants helpers hooks lib types
│   ├── eslint-config/
│   └── typescript-config/
├── package.json                 # workspaces + resolutions
├── turbo.json   .nvmrc   CLAUDE.md   README.md
└── .github/workflows/ci.yml
```

`apps/main/src/modules/` (13): `borrow` `governance` `layout` `liquidity` `onramp` `staking` `stats` `strategies` `submit-transaction` `trade` `transactions` `wallet` `xcm` — `governance/`, `onramp/`, `strategies/` are new this cycle.

`apps/main/src/api/` (37 entries): `aave` `account` `assets` `balances` `bonds` `borrow/` `chain` `constants` `democracy` `dryRun` `errors` `evm` `external/` `farms` **`gigaApr`** **`gigaStake`** **`grafana/`** `locks` `mock` `multisig` `omnipool` `otc` `payments` `pools` `provider` `referrals` `rpc` `spotPrice` `stableswap` `staking` `stats` `subscriptions` `trade` `transaction` `utils/` `xcm` `xyk`.

## Workspace packages

| Name | Purpose | Page |
|---|---|---|
| `@galacticcouncil/ui` | Radix + Emotion design system, Theme-UI theme, style-dictionary tokens, Storybook 10 | [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] |
| `@galacticcouncil/web3-connect` | Wallet abstraction: Substrate extensions, EVM (Reown AppKit), Solana, Sui, **Near**, **Zcash**; address book; multix multisig client | [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] |
| `@galacticcouncil/money-market` | Vendored Aave v3 UI logic for three markets (`hydration_v3`, `gigahdx_v3`, `bil_v3`) | [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] |
| `@galacticcouncil/indexer` | **Three** GraphQL clients + codegen: `indexer` (Subsquid), `squid` (PostGraphile), `multix` | [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] |
| `@galacticcouncil/utils` | Address guards, explorer links (Neckwork), formatting, hooks, asset-id constants | [[wiki/hydration-ui-utils\|hydration-ui-utils]] |
| `@galacticcouncil/eslint-config` | Shared ESLint + inline Prettier | [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] |
| `@galacticcouncil/typescript-config` | `base.json` / `vite.json` presets | [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] |

## SDK / protocol dependencies (workspace root)

```json
// package.json (2026-08-12)
"resolutions": { "polkadot-api": "^2.1.7", "strip-ansi": "6.0.1" },
"dependencies": {
  "@galacticcouncil/common":     "^1.2.0",
  "@galacticcouncil/descriptors":"^2.6.0",
  "@galacticcouncil/sdk-next":   "^1.6.0",
  "@galacticcouncil/xc":         "^2.1.0",
  "@galacticcouncil/xc-cfg":     "^2.3.0",
  "@galacticcouncil/xc-core":    "^2.3.0",
  "@galacticcouncil/xc-scan":    "^0.5.0",
  "@galacticcouncil/xc-sdk":     "^2.3.0",
  "@polkadot-api/tx-utils":      "^0.2.2",
  "polkadot-api":                "^2.1.0",
  "zod": "^4.1.11", "zustand": "^5.0.1", "remeda": "^2.19.0"
}
```

The **`resolutions` entry (`^2.1.7`) is what ships**, not the looser `dependencies` entry (`^2.1.0`). Full matrix in [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]].

## What changed 90d653f → b469e8a

- **[[wiki/gigahdx\|gigahdx]] liquid staking owns `/staking`.** Legacy staking demoted to `/staking-old`; feature flag removed. New `api/gigaStake.ts`, `api/gigaApr.ts`, `gigahdx_v3` money market, `HDXClassic` + `ArrowMigration` icons, `GigaHDXCans.webp`.
- **Three new modules → 13.** `strategies/` (BIL RWA vault, ERC-4626 + ERC-7540 async redeem; [[wiki/hollar\|hollar]] stable bonds via OTC offers 1488/1489), `onramp/` (Kraken / Binance / Kucoin / Coinbase / Gate.io + bank wizard through the new `assethub_cex` chain, served at `/deposit` and `/withdraw`), `governance/` (stub, nav entry commented out).
- **NTT replaces Wormhole MRL** across the transfer stack; `transactions/utils/wormhole.ts` deleted, `XcmTag.NttExecutor` added to `BRIDGE_PROVIDER_TAGS`.
- **Snowbridge fully removed** (`0ce954f`): `packages/indexer/src/snowbridge/`, `schema.snowbridge.graphql` (−3 199 lines), the `codegen:snowbridge` script, the `./snowbridge` export, `packages/utils/src/helpers/snowbridge.ts`, and the `api/provider.ts` wiring.
- **Charts migrating squid → Grafana SQL** (`apps/main/src/api/grafana/` with `.sql` + `.ts` pairs).
- **Wallet transaction history deleted**, replaced by `trade/orders/TradeOrdersHistory` + **Neckwork** explorer deep links; Neckwork is now the default explorer (was Subscan).
- **Cross-chain circuit-breaker limits surfaced** (`useCrossChainDepositLimit`, `useCrossChainGlobalWithdrawLimit`, `CBreaker*`) — see [[wiki/circuit-breaker\|circuit-breaker]].
- **Breaking package APIs:** `MoneyMarketProvider` now takes `market: CustomMarket` (was `env: MoneyMarketEnv`); `dataToVersionedTx` is async; `pingRpc()` / `jsonRpcFetch()` left `@galacticcouncil/utils` for an inlined `apps/main/src/utils/rpc-ping.js`.
- **Build:** MDX pipeline (`@mdx-js/rollup` 3.1.1 + `remark-gfm` 4.0.1) and TanStack Devtools added; papi resolution and the whole `@galacticcouncil/*` family bumped. Vite 8 / Rolldown, zod 4 and Storybook 10 were **already** in place at `90d653f`.

## Repo docs are partially stale

Root `CLAUDE.md` and `README.md` were touched in this window (`0ce954f`) **only** to drop Snowbridge from the package descriptions. Everything else in them predates the current tree — they still claim Vite 7, `vite-tsconfig-paths`, Node 22.13, CI Node 20.x, and a vendored `style-dictionary/tokens/` directory (tokens are fetched from the remote `hydration-styles` repo at build time). Treat [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] as authoritative over the repo's own docs.

## Sources

[[wiki/hydration\|hydration]], [[wiki/papi\|papi]], [[wiki/sdk-next\|sdk-next]], [[wiki/xc-sdk\|xc-sdk]]
