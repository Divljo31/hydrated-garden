---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-tech-stack/","title":"hydration-ui tech stack","tags":["frontend","stack","build","versions","architecture","hydration"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"hydration-ui tech stack","repo":"hydration-ui","paths":["package.json","yarn.lock","turbo.json",".nvmrc","apps/main/package.json","apps/main/vite.config.ts","packages/ui/package.json","packages/ui/.storybook/main.ts",".github/workflows/ci.yml"],"symbols":["defineConfig","tanstackRouter","devtools","mdx","babel","wasm","svgr","createHtmlPlugin"],"key_deps":["react","vite","@tanstack/react-router","@tanstack/react-query","@emotion/react","zustand","polkadot-api","@galacticcouncil/sdk-next","@galacticcouncil/descriptors","viem","ethers","zod","storybook","@mdx-js/rollup"],"tags":["frontend","stack","build","versions","architecture","hydration"],"last_updated":"2026-08-15"}}
---


# hydration-ui tech stack

**TL;DR:** React 19.2 SPA on **Vite 8 (Rolldown)**, TanStack Router (file-based) + Query, Emotion on a Theme-UI theme, Zustand + Immer, papi resolved to **2.1.7**, `@galacticcouncil/sdk-next` 1.6.0 / `descriptors` 2.6.0 / `xc-*` 2.x. Yarn 1 workspaces + Turborepo 2.9.6. **No test runner, no wagmi.** Version numbers below are re-derived from `package.json` + `yarn.lock` at `b469e8a`.

## Version matrix (resolved from `yarn.lock`, not the manifests)

| Concern | Package | Manifest range | **Resolved** | vs. 2026-05-13 (`90d653f`) |
|---|---|---|---|---|
| Runtime | Node (`.nvmrc`) | — | **v25.9.0** | unchanged |
| Package manager | yarn | `packageManager` | **1.22.22** | unchanged |
| Monorepo | turbo | `^2.9.5` | **2.9.6** | unchanged |
| Language | typescript | `5.7.3` | **5.7.3** | unchanged |
| UI | react / react-dom | `^19.1.1` | **19.2.5** | unchanged |
| Bundler | vite | `^8.0.10` | **8.0.10** | unchanged (Rolldown) |
| React plugin | `@vitejs/plugin-react` | `^6.0.1` | **6.0.1** | unchanged |
| Emotion babel | `@rolldown/plugin-babel` | `^0.2.0` | **0.2.3** | unchanged |
| MDX | `@mdx-js/rollup` | `3.1.1` | **3.1.1** | **NEW** |
| MDX | `remark-gfm` | `4.0.1` | **4.0.1** | **NEW** |
| Devtools | `@tanstack/devtools-vite` | `0.7.0` | **0.7.0** | **NEW** |
| Devtools | `@tanstack/react-devtools` | `0.10.5` | **0.10.5** | **NEW** |
| Routing | `@tanstack/react-router` | `^1.170.10` | **1.170.10** | 1.139.0 → |
| Routing | `@tanstack/router-plugin` | `^1.168.13` | **1.168.13** | 1.139.0 → |
| Data | `@tanstack/react-query` | `^5.100.14` | **5.100.14** | 5.59.15 → |
| Table / virt. | `@tanstack/react-table` / `react-virtual` | `^8.20.6` / `^3.14.2` | 8.x / **3.14.2** | virtual 3.13.12 → |
| Substrate | `polkadot-api` (**root `resolutions`**) | `^2.1.7` | **2.1.7** | `^2.1.0` (2.1.0) → |
| Substrate | `@polkadot-api/tx-utils` | `^0.2.2` | **0.2.5** | unchanged range |
| SDK | `@galacticcouncil/sdk-next` | `^1.6.0` | **1.6.0** | 1.0.1 → |
| SDK | `@galacticcouncil/descriptors` | `^2.6.0` | **2.6.0** | 2.0.0 → |
| SDK | `@galacticcouncil/common` | `^1.2.0` | **1.2.0** | 1.0.0 → |
| Cross-chain | `@galacticcouncil/xc` | `^2.1.0` | **2.1.0** | 1.0.0 → |
| Cross-chain | `xc-cfg` / `xc-core` / `xc-sdk` | `^2.3.0` | **2.3.0** | 1.0.0 → |
| Cross-chain | `@galacticcouncil/xc-scan` | `^0.5.0` | **0.5.0** | unchanged |
| EVM | viem | `^2.38.3` | **2.48.1** | unchanged |
| EVM | ethers (Aave only) | `5.7.0` | **5.7.0** | unchanged |
| EVM wallets | `@reown/appkit` | `^1.8.19` | **1.8.19** | unchanged |
| Solana / Sui | `@solana/web3.js` / `@mysten/wallet-standard` | `^1.98.0` / `^0.19.9` | 1.98.4 / 0.19.9 | unchanged |
| Lending | `@aave/contract-helpers`, `@aave/math-utils` | `1.23.1` | **1.23.1** | unchanged |
| Validation | zod | `^4.1.11` | **4.3.6** | unchanged (already v4) |
| State | zustand / immer | `^5.0.1` / `^10.0.3` | 5.0.3 / 10.x | unchanged |
| Forms | react-hook-form | `^7.54.2` | 7.x | unchanged |
| Styling | `@emotion/react` / `styled` | `^11.13.x` | 11.x | unchanged |
| Tokens | style-dictionary | `^4.1.4` | **4.4.0** | unchanged |
| Storybook | storybook + addons | `^10.3.5` | **10.3.5** | unchanged |
| Charts | recharts / lightweight-charts | `3.1.2` / `^5.0.8` | 3.1.2 / **5.1.0** | unchanged |
| GraphQL | graphql-request | `^7.2.0` | **7.4.0** | unchanged |
| i18n | i18next / react-i18next | `^23.16.4` / `^15.7.3` | 23.16.8 / 15.x | unchanged |
| Toasts | sonner | `^2.0.1` | **2.0.7** | unchanged |
| Workers | comlink | `^4.4.2` | 4.4.2 | unchanged |
| Lint | eslint / prettier | `8.57.0` / `3.3.3` | 8.57.0 / 3.3.3 | unchanged |

**Correction to earlier notes:** Vite 8 / Rolldown, `@vitejs/plugin-react` 6, `@rolldown/plugin-babel`, zod 4, Storybook 10.3.5, viem 2.48.1 and React 19.2.5 were **already in place at `90d653f`** — they are not part of this sync window. What actually moved 2026-05 → 2026-08 is: the papi resolution, the whole `@galacticcouncil/*` family, TanStack Router/Query, and the newly-added MDX + Devtools plugins.

## Build pipeline — `apps/main/vite.config.ts`

Plugin order is load-bearing:

```typescript
// apps/main/vite.config.ts (excerpt)
resolve: { tsconfigPaths: true },
build: { target: "es2022", outDir: "build",
         rolldownOptions: { output: { chunkFileNames: "chunk-[hash].js" } } },
plugins: [
  ...(mode !== "production" ? [devtools()] : []),
  mdx({ remarkPlugins: [remarkGfm], providerImportSource: "@mdx-js/react" }),
  tanstackRouter({ autoCodeSplitting: true }),
  react({ jsxImportSource: "@galacticcouncil/ui/jsx" }),
  babel({ include: /\.[jt]sx?$/, exclude: /node_modules/,
          plugins: ["@emotion/babel-plugin"] }),
  wasm(),
  svgr({ svgrOptions: { svgo: true } }),
  createHtmlPlugin({ /* minify + head injection */ }),
]
```

Key facts:
- **`vite-tsconfig-paths` is gone.** Path aliases now come from Vite 8's built-in `resolve.tsconfigPaths: true`. The repo's own `CLAUDE.md` still names the plugin — stale.
- **`build.rolldownOptions`** (not `rollupOptions`) — Vite 8 uses Rolldown. Manual chunking was dropped in favour of `chunkFileNames: "chunk-[hash].js"`.
- **Emotion runs through `@rolldown/plugin-babel`**, not `@vitejs/plugin-react`'s `babel` option, because plugin-react 6 on Rolldown no longer forwards babel plugins. The same pairing is repeated in `packages/ui/.storybook/main.ts`.
- **MDX is registered ahead of the router plugin.** It is used for long-form localized content, not routes — `apps/main/src/i18n/content/{propeller-vault,bil-vault,hollar-bonds-25-08-26}/en.mdx` — rendered through `providerImportSource: "@mdx-js/react"` and styled by the new `Prose` component in [[wiki/hydration-ui-design-system\|hydration-ui-design-system]].
- **`@mdx-js/react` is not a declared dependency** of `apps/main` — it resolves only as a transitive of `@storybook/addon-docs` plus `.npmrc auto-install-peers`. Fragile: removing Storybook would break the app build.
- Three inline scripts are read from disk and injected into `index.html`: `src/utils/head.js`, `src/components/Loader/loader.html` + `src/styles/critical.css`, and **`src/utils/rpc-ping.js`** — the latter is templated with `__RPC_URLS__` from `src/config/rpc.ts` (`PROVIDERS` filtered by `VITE_ENV`). This inlined pinger is where `pingRpc()` moved to after being removed from [[wiki/hydration-ui-utils\|hydration-ui-utils]].

## Node version drift (three-way)

| Source | Value |
|---|---|
| `.nvmrc` | `v25.9.0` |
| `.github/workflows/ci.yml` (`actions/setup-node`) | `25.x` |
| `README.md` prerequisites | "Node 22.13" |
| `CLAUDE.md` toolchain line | "Node 22.13 (`.nvmrc`)" |
| `CLAUDE.md` CI section | "Node version on CI is 20.x" |

`.nvmrc` and CI agree on **25**; both prose docs are stale (and disagree with each other). Trust `.nvmrc` / CI.

## Workspace graph

```
Level 0:  utils, indexer, eslint-config, typescript-config   (no internal deps)
Level 1:  ui            → utils
Level 2:  web3-connect  → ui, utils, indexer
Level 3:  money-market  → ui, utils, web3-connect
Level 4:  apps/main     → ui, utils, web3-connect, money-market, indexer
```

`turbo.json` (`$schema` pinned to `v2-9-5`) orders `build` via `dependsOn: ["^build"]`, output `build/**`; every other task (`lint`, `lint:fix`, `i18n`, `dev`, `dev:production`, `preview`) is `cache: false`. Packages ship **source TypeScript** — `main: ./src/index.ts` or an `exports` map into `src/` — so there is no `dist/` between workspaces and every consumer re-bundles.

## Package export shapes (differ per workspace)

| Package | Shape |
|---|---|
| `@galacticcouncil/ui` | `exports: { "./*": "./src/*/index.ts", "./fonts.css": …, "./assets/images/*": … }` — directory-level ([[wiki/hydration-ui-design-system\|hydration-ui-design-system]]) |
| `@galacticcouncil/money-market` | `exports: { "./*": "./src/*/index.ts" }` ([[wiki/hydration-ui-money-market\|hydration-ui-money-market]]) |
| `@galacticcouncil/indexer` | explicit map: `./indexer`, `./squid`, `./squid/lib/*`, `./multix` — `./snowbridge` **removed** ([[wiki/hydration-ui-indexer\|hydration-ui-indexer]]) |
| `@galacticcouncil/utils` | `main: ./src/index.ts` ([[wiki/hydration-ui-utils\|hydration-ui-utils]]) |
| `@galacticcouncil/web3-connect` | `main: ./src/index.ts` ([[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]]) |

## Conventions

- **TypeScript 5.7 strict**, `target/module: ESNext`, `moduleResolution: Bundler`; `noUnusedLocals` / `noUnusedParameters` / `noUncheckedIndexedAccess` only in `apps/main`.
- **Prettier is inline in ESLint** (`packages/eslint-config/index.js`): `semi: false`, `trailingComma: "all"`, `arrowParens: "always"`, `printWidth: 80`, double quotes in JSX.
- **`no-restricted-imports` bans relative paths** — `@/…` inside `apps/main`, package names across workspaces. Plus `simple-import-sort`, `react-hooks/exhaustive-deps`, `@typescript-eslint/switch-exhaustiveness-check`, `eqeqeq`.
- **Custom JSX runtime** `@galacticcouncil/ui/jsx` in both `apps/main` and `packages/ui` — no `import React` needed for JSX.
- **No test runner.** Validation = `yarn lint` (eslint + `tsc --noEmit`), `yarn build`, Storybook, manual browser checks. CI (`.github/workflows/ci.yml`, name `Lint`, on every push) runs `yarn install --frozen-lockfile && yarn run build && yarn run lint` on Node 25.x.
- **Generated, never hand-edited:** `apps/main/src/routeTree.gen.ts`, `packages/indexer/src/*/__generated__/`, `packages/ui/src/theme/tokens/index.ts`.
- **Design tokens are fetched remotely** at `yarn theme` time from `galacticcouncil/hydration-styles` (branch `tertiary`) — `packages/ui/style-dictionary/tokens/` is empty in-repo, contradicting the repo `CLAUDE.md`.
- **No wagmi** anywhere in the tree — EVM wallet connection is Reown AppKit + viem directly ([[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]]).

## Protocol integration

- **papi** (`polkadot-api`, root `resolutions:
{ #2}
.1.7`) for RPC, typed storage, subscriptions; `@polkadot-api/tx-utils` for extrinsic/permit encoding ([[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]]).
- **[[wiki/sdk-next\|sdk-next]] 1.6.0** for trade routing, pool math (WASM via `vite-plugin-wasm`), money-market oracles and ICE intent builders.
- **[[wiki/sdk-descriptors\|sdk-descriptors]] 2.6.0** for typed pallet metadata — note that `GigaHdx` / `GigaHdxRewards` / `FeeProcessor` exist in the SCALE metadata but are **not whitelisted**, so [[wiki/gigahdx\|gigahdx]] calls in the UI go through raw/unsafe paths rather than typed descriptors.
- **[[wiki/xc-sdk\|xc-sdk]] / [[wiki/xc-cfg\|xc-cfg]] / [[wiki/xc-core\|xc-core]] 2.3.0 + [[wiki/xc-package\|xc-package]] 2.1.0** for cross-chain; the NTT bridge model replaced Wormhole MRL in this window.

## For new developers

1. [[wiki/hydration-ui\|hydration-ui]] — repo overview and where each concern lives
2. [[wiki/hydration-ui-main-app\|hydration-ui-main-app]] — entry point, providers, routing
3. [[wiki/hydration-ui-api\|hydration-ui-api]] — data-fetching hook patterns
4. [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] — components, theme, tokens
5. [[wiki/hydration-ui-modules\|hydration-ui-modules]] — pick a feature and trace it
6. [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] — how a tx actually gets signed and sent

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-main-app\|hydration-ui-main-app]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-ui-design-system\|hydration-ui-design-system]], [[wiki/papi\|papi]], [[wiki/sdk-next\|sdk-next]]
