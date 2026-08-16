---
{"dg-publish":true,"permalink":"/wiki/source-sdk-codebase/","title":"Galactic SDK — Codebase","tags":["sdk","codebase","typescript","wasm","cross-chain","trading"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"source","title":"Galactic SDK — Codebase","author":"Galactic Council (galacticcouncil/sdk)","source_kind":"repo_clone","raw_path":"raw/sdk/","upstream":"https://github.com/galacticcouncil/sdk","date_ingested":"2026-04-13","source_date":"2026-08-12","last_refreshed":"2026-08-15","last_commit":"c57d172 apps: revert grid styles (2026-08-12)","prev_commit":"96b4b0e move web ctx to apps (2026-05-13)","cloned_at":"2026-08-15","produces_pages":["sdk-next","sdk-common","sdk-descriptors","xc-package","xc-sdk","xc-cfg","xc-core","xc-scan","xc-swap","route-suggester"],"tags":["sdk","codebase","typescript","wasm","cross-chain","trading"],"last_updated":"2026-08-15"}}
---


# Source: Galactic SDK — Codebase

**Raw location:** `raw/sdk/` (git clone of `galacticcouncil/sdk`)

## Summary

**TL;DR:** The Galactic SDK is a TypeScript monorepo providing everything to build on [[wiki/hydration\|hydration]] — from pool math to trading and cross-chain tooling, built on polkadot-api (papi v2), with **17 packages** under `@galacticcouncil/*`.

## What changed since the May 2026 sync

219 commits, 441 files (`96b4b0e..c57d172`). Three themes:

1. **MRL → Wormhole NTT.** The Moonbeam-routed Wormhole stack (TokenBridge / TokenRelayer / MRL) was deleted and replaced by Native Token Transfers — direct Hydration ↔ Ethereum / Base / Solana / Sui. 5 chains removed (`moonbeam`, `interlay`, `crust`, `ajuna`, `laos`), 1 added (`assethub_cex`), all `*_mwh` asset keys renamed `*_wh`. See [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]].
2. **Route/balance model rework.** Balance and minimum reads moved from route builders onto the chain object; `AssetRoute.transact` gone; routes may be one-way with a `reversible` flag; a transfer is now an ordered **sequence** of calls. See [[wiki/xc-core\|xc-core]], [[wiki/xc-sdk\|xc-sdk]], `packages/xc/docs/unidirectional-routes.md`.
3. **New surfaces in [[wiki/sdk-next\|sdk-next]]:** ICE intent tx builders, a standalone block indexer, a money-market oracle client, stableswap peg recomputation — plus the new [[wiki/xc-swap\|xc-swap]] package.

## Architecture

```
Your dApp
├── sdk-next ──── Trade routing, pool queries, smart order router, staking,
│   │             liquidity-mining (farm), historical reads, offline snapshots,
│   │             ICE intent tx builders, block indexer, MM oracles
│   └── math-* ── Pool math (WASM, compiled from Rust)
├── xc ────────── Cross-chain transfers (batteries-included)
│   ├── xc-sdk ── Wallet, ordered call sequences, platform adapters, claims
│   ├── xc-cfg ── Route configs, DEX, bridge builders (NTT, Snowbridge, Basejump),
│   │             transfer validations (circuit-breaker + NTT rate limits)
│   └── xc-core ─ Core types, chain & asset definitions, bridge primitives
├── xc-swap ───── NEAR Intent Routing swaps (sibling of xc, not under it)
├── xc-scan ───── Journey tracking (standalone; xc no longer depends on it)
├── common ────── Shared utilities (big numbers, XCM, EVM, substrate, rxjs ops)
├── descriptors ─ papi v2 metadata for hydration / hydrationNext / hydrationIce / hub
└── polkadot-api ─ Substrate SDK (foundation, v2.1.7)
```

## File tree (top 2 levels)

```
sdk/
├── packages/          17 packages (below)
├── chopsticks/        NEW — Chopsticks integration tests (was integration-tests/xc-test/)
│   └── src/           call.spec.ts, e2e.spec.ts, omni-tear.spec.ts, lib/, spec/, utils/
├── apps/              Not published — rescue helper (was redeem)
│   └── src/rescue/    recover funds stranded on the ETH\0 phantom account
├── examples/          sdk-next-cjs, sdk-next-esm, xc-transfer
├── crates/            route-suggester (Rust)
├── scripts/           build-math, changeset-*, NEW rpc-latency.sh / rpc-latency-catfish.sh
└── CLAUDE.md          repo-level agent instructions
```

npm workspaces are `packages/*`, `examples/*`, `chopsticks`.

## Package Versions

| Package | Version | Was | Notes |
|---|---|---|---|
| `@galacticcouncil/common` | 1.2.0 | 1.0.0 | `rx.debounceAfterFirst`, `changedEntries` |
| `@galacticcouncil/descriptors` | 2.6.0 | 2.1.0 | metadata regen; new `hydrationIce` entry + `wasm/ice/ice.wasm` |
| `@galacticcouncil/sdk-next` | 1.6.0 | 1.0.1 | ICE intents, indexer, MM oracles, stableswap pegs |
| `@galacticcouncil/xc` | 2.1.0 | 1.0.0 | `dexAlias`; 4 new reference docs |
| `@galacticcouncil/xc-cfg` | 2.3.0 | 1.1.0 | **NTT replaces MRL**; NTT clients, rate-limit validation, bridge constants |
| `@galacticcouncil/xc-core` | 2.3.0 | 1.0.0 | chain-native balances, `Ntt` primitive, slimmer `AssetRoute` |
| `@galacticcouncil/xc-sdk` | 2.3.0 | 1.0.0 | `buildCalls()`, `reversible`, `SubstrateEvm`, balance reads removed |
| `@galacticcouncil/xc-swap` | 0.6.0 | — | **new** — NEAR Intent Routing swaps |
| `@galacticcouncil/xc-scan` | 0.5.0 | 0.5.0 | unchanged |
| `@galacticcouncil/math-stableswap` | 2.5.0 | — | `recalculatePegs` consumed by `StableSwapPeg` |
| other `math-*` | 1.2.0–1.4.0 | — | ema, hsm, lbp, liquidity-mining, omnipool, staking, xyk |

## Packages (17)

### General
- **[[wiki/sdk-next\|sdk-next]]** (`sdk-next`) — Trade router, pool queries, [[wiki/smart-order-router\|smart-order-router]], DCA/TWAP scheduling, tx building, staking + farm APIs, historical reads, offline snapshots, ICE intent builders, block indexer, MM oracle client
- **[[wiki/sdk-common\|sdk-common]]** (`common`) — big-number ops, XCM transforms, EVM/ERC20 helpers, substrate RPC, rxjs subscription operators
- **[[wiki/sdk-descriptors\|sdk-descriptors]]** (`descriptors`) — papi v2 metadata for `hydration`, `hydrationNext`, `hydrationIce`, `hub`

### Cross-Chain (XC)
- **[[wiki/xc-package\|xc-package]]** (`xc`) — context factory + the stack's reference `docs/`
- **[[wiki/xc-sdk\|xc-sdk]]** (`xc-sdk`) — wallet, ordered call sequences, platform adapters, claims, Wormhole scan/transfer
- **[[wiki/xc-cfg\|xc-cfg]]** (`xc-cfg`) — chain/asset/route data, DEX integrations, bridge builders, validations, NTT + Executor clients
- **[[wiki/xc-core\|xc-core]]** (`xc-core`) — core types, chain & asset definitions, `Ntt`/`Wormhole`/`Snowbridge`/`Basejump` primitives, EVM prerequisites
- **[[wiki/xc-swap\|xc-swap]]** (`xc-swap`) — **new**; NEAR Intent Routing swaps via `IntentEmitter` + 1Click
- **[[wiki/xc-scan\|xc-scan]]** (`xc-scan`) — cross-chain transaction scanning (standalone)

### Math (WASM)
8 packages compiled from Rust (`HydraDX-wasm`), one per pool type: `math-omnipool`, `math-stableswap`, `math-xyk`, `math-lbp`, `math-hsm`, `math-ema`, `math-staking`, `math-liquidity-mining`.

### Supporting
- **[[wiki/route-suggester\|route-suggester]]** — Rust crate for high-performance DEX route discovery (BFS, cycle prevention)

## Chopsticks integration tests (`chopsticks/`)

`integration-tests/xc-test/` was **moved to a top-level `chopsticks/` workspace**. Private, `@acala-network/chopsticks`
{ #1}
.4.0, depends on `sdk-next`
{ #1}
.6.0 + the three xc packages
{ #2}
.3.0.

| Script | Spec | What it covers |
|---|---|---|
| `npm run spec:calldata` | `src/call.spec.ts` | snapshot-verified call data for every route (`src/spec/calldata/`) |
| `npm run spec:e2e` | `src/e2e.spec.ts` | forked-network cross-chain transfers, event + storage assertions (`src/spec/e2e/`) |
| `npm run spec:omni` | `src/omni-tear.spec.ts` | **red test** pinning the Omnipool torn-snapshot bug — see [[wiki/sdk-next\|sdk-next]] |

> The repo's `CLAUDE.md` still documents `cd integration-tests/xc-test` for these commands. That path no longer exists.

## Examples & apps

- **`examples/xc-transfer/`** — the working demo surface. New pages: `src/ntt.ts` + `public/ntt/` (NTT transfers, VAA + Executor helpers in `src/utils/{vaa,executor}.ts`) and `src/swap.ts` + `public/swap/` (the [[wiki/xc-swap\|xc-swap]] demo). Existing: transfer, redeem, scan, circuitbreaker.
- **`apps/`** — not published. `src/redeem/` was replaced by **`src/rescue/`**: dispatches an `EVM.call` through the pallet-evm dispatch precompile (`0x…0401`) to recover funds stranded on the `ETH\0` phantom account, metering gas in WETH (asset id 20 — an `ETH\0` account can't set a fee currency, so `account_currency` defaults it to `EvmAssetId`, which is WETH).
- **`scripts/rpc-latency.sh`** / **`rpc-latency-catfish.sh`** — round-trip latency of the `subway.*` / `rpc.kril` endpoints and the 4 `node-catfish-N` nodes, 3 tries each via `chain_getBlock`.

## Key Technical Details

- **Toolchain:** TypeScript 5.7 (strict, ES2022), Node 25+ (`.nvmrc` = 25; CI on 25.9.0), esbuild, Turborepo, Jest
- **Dual output:** ESM + CJS for all packages (except xc-scan and descriptors, ESM-only)
- **Math packages:** WASM binaries fetched from an external repo, not compiled locally
- **Descriptors:** generated via `papi whitelist` from `src/whitelist.ts` → `.papi/whitelist.ts`; `hydrationIce` metadata comes from a committed runtime wasm, not an RPC
- **Peer deps:** common, descriptors, polkadot-api (`^2.1.7`), viem (`^2.38.3`) are intentionally peers
- **Releases:** SemVer via Changesets (`changeset:version`, `changeset:snapshot`)
- **CI:** GitHub Actions (build + test on push/PR; snapshot releases on PRs)
- **Patches:** `patches/@polkadot-api+cli+0.20.4.patch` was **deleted** this cycle — the papi CLI is no longer vendored, though `postinstall` still runs `patch-package`

## Cross-repo notes ([[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]])

- Node spec 419 → 439, 39 → 42 pallets. The regenerated `hydration.scale` **contains** [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] and [[wiki/pallet-fee-processor\|pallet-fee-processor]], but none is whitelisted — no typed API for them yet.
- [[wiki/pallet-stableswap\|pallet-stableswap]] gained virtual share-issuance storage (v1→v2). `sdk-next`'s `StableSwapClient` still derives share supply from `Tokens.TotalIssuance` — **flagged as wrong** on [[wiki/sdk-next\|sdk-next]].
- Trade fees now route through [[wiki/pallet-fee-processor\|pallet-fee-processor]] only, and 50% of the Omnipool asset fee leaves the pool — relevant to any SDK-side fee/APR reconstruction.

## Entities Referenced

- [[wiki/hydration\|hydration]], [[omnipool\|omnipool]], [[wiki/hdx\|hdx]], [[wiki/lrna\|lrna]], [[wiki/gigahdx\|gigahdx]]
- [[wiki/sdk-next\|sdk-next]], [[wiki/xc-package\|xc-package]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xc-scan\|xc-scan]], [[wiki/xc-swap\|xc-swap]]
- [[wiki/smart-order-router\|smart-order-router]], [[wiki/route-suggester\|route-suggester]]
- [[wiki/polkadot\|polkadot]], [[wiki/xcm\|xcm]], [[wiki/snowbridge\|snowbridge]], [[wiki/wormhole\|wormhole]]

## Concepts Referenced

- [[wiki/smart-order-router\|smart-order-router]] — BFS-based trade routing
- [[wiki/dca\|dca]], [[wiki/stableswap\|stableswap]], [[wiki/xyk-pools\|xyk-pools]], [[wiki/lbp\|lbp]]
- [[wiki/pallet-frontier\|pallet-frontier]] — EVM integration via viem

## Sources

- Raw clone: `raw/sdk/` @ `c57d172` (2026-08-12) — <https://github.com/galacticcouncil/sdk>
- In-repo docs: `packages/xc/docs/`, `packages/sdk-next/docs/`, `packages/xc-swap/docs/spec.md`, root `CLAUDE.md`
- Cross-repo: [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
