---
{"dg-publish":true,"permalink":"/wiki/source-sdk-codebase/","title":"Galactic SDK — Codebase","tags":["sdk","codebase","typescript","wasm","cross-chain","trading"],"dg-note-properties":{"type":"source","title":"Galactic SDK — Codebase","author":"Galactic Council (galacticcouncil/sdk)","date_ingested":"2026-04-13","source_date":"2026-04-20","last_refreshed":"2026-04-20","last_commit":"4f5ad1b xc: add Basejump bridge support (2026-04-17)","cloned_at":"2026-04-20","tags":["sdk","codebase","typescript","wasm","cross-chain","trading"]}}
---


# Source: Galactic SDK — Codebase

**Raw location:** `raw/sdk/` (git clone of `galacticcouncil/sdk`)

## Summary

**TL;DR:** The Galactic SDK is a TypeScript monorepo providing everything to build on [[wiki/hydration\|hydration]] — from pool math to trading and cross-chain tooling, built on polkadot-api (papi), with 16 npm packages under `@galacticcouncil/*`.

## Architecture

The SDK has a layered architecture:

```
Your dApp
├── sdk-next ──── Trade routing, pool queries, smart order router
│   └── math-* ── Pool math (WASM, compiled from Rust)
├── xc ────────── Cross-chain transfers (batteries-included)
│   ├── xc-sdk ── Wallet interface, multi-platform transfers
│   ├── xc-cfg ── Route configs, DEX integrations, bridge builders
│   └── xc-core ─ Core types, chain & asset definitions, bridge primitives
├── common ────── Shared utilities (big numbers, XCM, EVM, substrate)
├── descriptors ─ Hydration papi type-safe metadata
└── polkadot-api ─ Substrate SDK (foundation)
```

**Recent:** Basejump bridge support added (Apr 17) with tag-based multi-bridge route selection.

## Packages (16 published)

### General
- **[[wiki/sdk-next\|sdk-next]]** (`@galacticcouncil/sdk-next`) — Trade router, pool queries, smart order router, DCA/TWAP scheduling, transaction building
- **[[wiki/sdk-common\|sdk-common]]** (`@galacticcouncil/common`) — Shared utilities: big number ops, XCM transformations, EVM/ERC20 helpers, substrate RPC
- **[[wiki/sdk-descriptors\|sdk-descriptors]]** (`@galacticcouncil/descriptors`) — Hydration papi chain metadata descriptors, generated from whitelist

### Cross-Chain (XC)
- **[[wiki/xc-package\|xc-package]]** (`@galacticcouncil/xc`) — High-level context factory, batteries-included entry point
- **[[wiki/xc-sdk\|xc-sdk]]** (`@galacticcouncil/xc-sdk`) — Wallet interface for multi-platform transfers (Substrate, EVM, Solana, Sui)
- **[[wiki/xc-cfg\|xc-cfg]]** (`@galacticcouncil/xc-cfg`) — Pre-built route configs, DEX integrations (HydrationDex, AssetHub)
- **[[wiki/xc-core\|xc-core]]** (`@galacticcouncil/xc-core`) — Core types, chain & asset definitions, config builder
- **[[wiki/xc-scan\|xc-scan]]** (`@galacticcouncil/xc-scan`) — Cross-chain transaction scanning and journey tracking

### Math (WASM)
8 packages compiled from Rust (`HydraDX-wasm` repo), one per pool type:
- `math-omnipool`, `math-stableswap`, `math-xyk`, `math-lbp`, `math-hsm`, `math-ema`, `math-staking`, `math-liquidity-mining`

### Supporting
- **[[wiki/route-suggester\|route-suggester]]** — Rust crate for high-performance DEX route discovery (BFS, cycle prevention)

## Key Technical Details

- **Toolchain:** TypeScript 5.7 (strict), Node 22+, esbuild, Turborepo, Jest
- **Dual output:** ESM + CJS for all packages (except xc-scan, ESM-only)
- **Math packages:** WASM binaries fetched from external repo, not compiled locally
- **Descriptors:** Generated via `papi whitelist` from `src/whitelist.ts`
- **Peer deps:** common, descriptors, polkadot-api, viem are intentionally peers to avoid duplication
- **Releases:** SemVer via Changesets
- **CI:** GitHub Actions (build + test on push/PR; snapshot releases on PRs)

## Entities Referenced

- [[wiki/hydration\|hydration]], [[omnipool\|omnipool]], [[wiki/hdx\|hdx]], [[wiki/lrna\|lrna]]
- [[wiki/sdk-next\|sdk-next]], [[wiki/xc-package\|xc-package]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xc-scan\|xc-scan]]
- [[wiki/smart-order-router\|smart-order-router]], [[wiki/route-suggester\|route-suggester]]
- [[wiki/polkadot\|polkadot]], [[wiki/xcm\|xcm]], [[wiki/snowbridge\|snowbridge]], [[wiki/wormhole\|wormhole]]

## Concepts Referenced

- [[wiki/smart-order-router\|smart-order-router]] — BFS-based trade routing
- [[wiki/dca\|dca]], [[wiki/stableswap\|stableswap]], [[wiki/xyk-pools\|xyk-pools]], [[wiki/lbp\|lbp]]
- [[wiki/pallet-frontier\|pallet-frontier]] — EVM integration via viem
