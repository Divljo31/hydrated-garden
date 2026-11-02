---
{"dg-publish":true,"permalink":"/wiki/hydration-ui-tech-stack/","title":"hydration-ui tech stack","tags":["frontend","stack","build","versions","architecture","hydration"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"hydration-ui tech stack","repo":"hydration-ui","paths":["package.json","yarn.lock","turbo.json",".nvmrc","AGENTS.md","CODING_STANDARDS.md","apps/main/package.json","apps/main/vite.config.ts","packages/ui/package.json","packages/ui/.storybook/main.ts",".github/workflows/ci.yml",".github/workflows/ipfs.yml"],"symbols":["defineConfig","tanstackRouter","devtools","mdx","babel","wasm","svgr","createHtmlPlugin","fontPreload"],"key_deps":["react","vite","vitest","@tanstack/react-router","@tanstack/react-query","@tanstack/charts","@emotion/react","zustand","polkadot-api","@galacticcouncil/sdk-next","@galacticcouncil/descriptors","@galacticcouncil/xc-swap","viem","ethers","zod","openapi-fetch","storybook","@mdx-js/rollup"],"tags":["frontend","stack","build","versions","architecture","hydration"],"last_updated":"2026-11-02"}}
---


# hydration-ui tech stack

**TL;DR:** React 19.2 SPA on **Vite 8 (Rolldown)**, TanStack Router (file-based) + Query, Emotion on a Theme-UI theme, Zustand + Immer, papi **2.2.2**, `@galacticcouncil/sdk-next` **2.5.0**, `descriptors` 2.9.0, `xc-*` 2.x + `xc-swap` 0.9. Yarn 1 workspaces + Turborepo. **Vitest** is now the test runner (CI runs it). Node **v26.12.1**. Versions below are resolved from `yarn.lock` at `770f868` (2026-11-01).

## Version matrix (resolved)

| Concern | Package | Range | Resolved |
|---|---|---|---|
| Runtime | Node (`.nvmrc`) | — | **v26.12.1** |
| Package manager / monorepo | yarn / turbo | `1.22.22` / `^2.9.5` | 1.22.22 / 2.9.6 |
| Language | typescript | `5.7.3` (root resolution) | 5.7.3 |
| UI | react / react-dom | `^19.1.1` | 19.2.5 |
| Bundler | vite / `@vitejs/plugin-react` / `@rolldown/plugin-babel` | `^8.0.10` / `^6.0.1` / `^0.2.0` | 8.0.10 / 6.0.1 / 0.2.3 |
| Tests | vitest | `^3.2.7` | 3.2.7 |
| MDX | `@mdx-js/rollup`, `remark-gfm` | `3.1.1`, `4.0.1` | same |
| Routing | `@tanstack/react-router` / `router-plugin` | `^1.170.10` / `^1.168.13` | 1.170.10 / 1.168.13 |
| Data | `@tanstack/react-query` (+ `query-persist-client-core`) | `5.101.4` (pinned) | 5.101.4 |
| Charts | `@tanstack/charts` / lightweight-charts | `0.14.0` / `^5.0.8` | 0.14.0 / 5.1.0 |
| Substrate | `polkadot-api` (root `resolutions:
{ #2}
.1.7`) | `^2.1.0` | **2.2.2** |
| Substrate | `@polkadot-api/tx-utils` | `^0.2.2` | 0.2.5 |
| SDK | `@galacticcouncil/sdk-next` / `descriptors` / `common` | `^2.5.0` / `^2.9.0` / `^1.2.0` | 2.5.0 / 2.9.0 / 1.2.0 |
| Cross-chain | `xc` / `xc-cfg` / `xc-core` / `xc-sdk` / `xc-scan` / **`xc-swap`** | `^2.1.0` / `^2.9.1` / `^2.8.0` / `^2.7.0` / `^0.5.0` / `^0.9.0` | 2.1.0 / 2.9.1 / 2.8.0 / 2.7.0 / 0.5.0 / 0.9.0 |
| EVM | viem / ethers (Aave only) | `^2.55.4` / `5.7.0` | 2.56.8 / 5.7.0 |
| Wallets | `@reown/appkit`, `@hot-labs/near-connect`, `@noir-wallet/sdk`, `@solana/web3.js`, `@mysten/wallet-standard` | | 1.8.19, 0.11.4, 0.1.9, 1.98.4, 0.19.9 |
| Lending | `@aave/contract-helpers`, `@aave/math-utils` | `1.23.1` | 1.23.1 |
| Indexer | `openapi-fetch` / `graphql-request` | `^0.17.0` / `^7.2.0` | 0.17.0 / 7.4.0 |
| Validation | zod | `^4.1.11` | 4.3.6 |
| State / forms | zustand, immer, react-hook-form | | 5.0.3, 10.2.0, 7.72.1 |
| Tokens / docs | style-dictionary / storybook | `^4.1.4` / `^10.3.5` | 4.4.0 / 10.3.5 |
| i18n / toasts / workers | i18next, sonner, comlink | | 23.16.8, 2.0.7, 4.4.2 |

**Not used:** recharts (replaced by `@tanstack/charts`), wagmi, `vite-tsconfig-paths`.

## Build — `apps/main/vite.config.ts`

```typescript
resolve: { tsconfigPaths: true },
build: { target: "es2022", outDir: "build", rolldownOptions: { output: { chunkFileNames: "chunk-[hash].js" } } },
plugins: [
  ...(mode !== "production" ? [devtools()] : []),
  mdx({ remarkPlugins: [remarkGfm], providerImportSource: "@mdx-js/react" }),
  tanstackRouter({ autoCodeSplitting: true }),
  react({ jsxImportSource: "@galacticcouncil/ui/jsx" }),
  babel({ include: /\.[jt]sx?$/, exclude: /node_modules/, plugins: ["@emotion/babel-plugin"] }),
  wasm(), svgr(…), createHtmlPlugin(…), fontPreload(),
]
```

- Emotion runs through `@rolldown/plugin-babel` (plugin-react 6 on Rolldown doesn't forward babel plugins); Storybook repeats the pairing.
- MDX is for localized long-form content (`src/i18n/content/*`), rendered with `Prose` ([[wiki/hydration-ui-design-system\|hydration-ui-design-system]]). `@mdx-js/react` is not a declared dep of `apps/main` — it resolves transitively via Storybook's addon-docs.
- `index.html` gets inlined `src/utils/head.js`, loader HTML + `critical.css`, and `src/utils/rpc-ping.js` templated with `PROVIDERS` URLs for `VITE_ENV`; `fontPreload` adds `<link rel=preload>` for the Geist / Gazpacho woff2 files.

## CI / deploy

- `.github/workflows/ci.yml`: Node from `.nvmrc`; `yarn install --frozen-lockfile && yarn run build && yarn run lint && yarn run test`.
- `.github/workflows/ipfs.yml`: on push to `master` (or manual), build → Kubo `ipfs add -r --cid-version=1` → CAR → pin to two pinning services.
- Netlify hosts the app and deploy previews.

## Workspace graph

```
Level 0:  utils, indexer, eslint-config, typescript-config
Level 1:  ui            → utils
Level 2:  web3-connect  → ui, utils, indexer
Level 3:  money-market  → ui, utils, web3-connect   (+ indexer, undeclared)
Level 4:  apps/main     → ui, utils, web3-connect, money-market (+ indexer via root hoist)
```

`turbo.json`: `build` depends on `^build` (outputs `build/**`); `lint` depends on `^build`; `test`, `lint:fix`, `i18n` uncached; `dev` / `preview` persistent. Packages ship **source TS**, but `apps/main` type-checks packages through `packages/<name>/build/` — after editing a package run `npx tsc --build` in it (stale-build errors otherwise; see repo `AGENTS.md`).

## Package export shapes

| Package | Shape |
|---|---|
| `@galacticcouncil/ui` | `"./*": "./src/*/index.ts"`, `./fonts.css`, `./assets/images/*` — directory-level |
| `@galacticcouncil/money-market` | `"./*": "./src/*/index.ts"` |
| `@galacticcouncil/indexer` | `./indexer`, `./neckwork`, `./multix`, `./scan` ([[wiki/hydration-ui-indexer\|hydration-ui-indexer]]) |
| `@galacticcouncil/utils`, `@galacticcouncil/web3-connect` | `main: ./src/index.ts` |

## Conventions

- Repo agent docs: root `CLAUDE.md` is just `@AGENTS.md`; `AGENTS.md` (workflow, where things are) + `CODING_STANDARDS.md` (one rule per file kind, with example files). These are current — earlier stale-doc caveats no longer apply, except `README.md` still says Node 22.13.
- TypeScript 5.7 strict, `moduleResolution: Bundler`; ESLint 8 with inline Prettier; `no-restricted-imports` bans relative paths (`@/…` inside `apps/main`).
- Custom JSX runtime `@galacticcouncil/ui/jsx`.
- Tests: Vitest for pure logic (`*.test.ts` in `apps/main/src`, `packages/web3-connect/src`, `packages/ui/src`, `packages/utils/src`). UI changes are verified by eye (app dev server or Storybook on 6006).
- Generated, never hand-edit: `apps/main/src/routeTree.gen.ts`, `packages/indexer/src/*/__generated__/`, `packages/ui/src/theme/tokens/index.ts`, `packages/ui/src/theme/assets/assetColors.json`, `AccountAvatar/emoji/snakewatch.ts`.
- Design tokens are fetched at `yarn theme` time from `galacticcouncil/hydration-styles` (branch `tertiary`).

## Protocol integration

- [[wiki/papi\|papi]] for RPC / typed storage / subscriptions; `@polkadot-api/tx-utils` for extrinsic + permit encoding ([[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]]).
- [[wiki/sdk-next\|sdk-next]] 2.5 — router, pool math (WASM), Uniswap V3 pools, ICE intent tx builders (`sdk.tx.intentLimit`).
- [[wiki/sdk-descriptors\|sdk-descriptors]] 2.9 — `hydration` and `hydrationNext` typed APIs (`papi` / `papiNext` on the RPC context).
- [[wiki/xc-sdk\|xc-sdk]] / [[wiki/xc-cfg\|xc-cfg]] / [[wiki/xc-core\|xc-core]] / [[wiki/xc-package\|xc-package]] / [[wiki/xc-scan\|xc-scan]] for transfers; [[wiki/xc-swap\|xc-swap]] for the Market swap tab's cross-chain quotes.

## Sources

[[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]], [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-main-app\|hydration-ui-main-app]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-ui-design-system\|hydration-ui-design-system]], [[wiki/papi\|papi]], [[wiki/sdk-next\|sdk-next]]
