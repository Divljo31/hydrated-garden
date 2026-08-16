---
{"dg-publish":true,"permalink":"/wiki/xc-package/","title":"xc (Cross-Chain Package)","tags":["sdk","cross-chain","transfers","xc","ntt"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"xc (Cross-Chain Package)","repo":"sdk","paths":["packages/xc/src/factory.ts","packages/xc/src/dex.ts","packages/xc/src/types.ts","packages/xc/docs/overview.md","packages/xc/docs/ntt.md","packages/xc/docs/chain-native-balances.md","packages/xc/docs/unidirectional-routes.md"],"key_exports":["createXcContext","XcCtx","XcOpts","registerDexes"],"tags":["sdk","cross-chain","transfers","xc","ntt"],"source_count":1,"last_updated":"2026-08-15"}}
---


# xc (Cross-Chain Package)

**TL;DR:** `@galacticcouncil/xc` (v2.1.0, papi v2) is the batteries-included entry point for cross-chain transfers — a context factory wiring [[wiki/xc-core\|xc-core]], [[wiki/xc-cfg\|xc-cfg]] and [[wiki/xc-sdk\|xc-sdk]] into one API. Four source files, ~50 lines of logic; its real weight this cycle is **`docs/`, which is now the XC stack's reference documentation**.

## Quick Start

```ts
import { createXcContext } from '@galacticcouncil/xc';
const xc = await createXcContext();
```

Can share pool context with [[wiki/sdk-next\|sdk-next]]:
```ts
const sdk = await createSdkContext(client);
const xc = await createXcContext({ poolCtx: sdk.ctx.pool });
```

Returns `XcCtx = { config, wallet, wormhole: { scan, transfer } }`. `XcOpts` is `{ poolCtx?: pool.PoolContextProvider }`.

## Architecture

```
xc (this package) ← Start here
├── xc-sdk  ← Wallet, ordered call sequences, platform adapters, claims
├── xc-cfg  ← Route configs, DEX, bridge builders (NTT, Snowbridge, Basejump),
│             transfer validations
└── xc-core ← Core types, chain & asset definitions, bridge primitives
```

Two packages sit **beside** this stack, not under it: [[wiki/xc-swap\|xc-swap]] (NEAR intent swaps) and [[wiki/xc-scan\|xc-scan]] — `xc` no longer depends on `xc-scan`; the `wormhole.scan` / `wormhole.transfer` clients on `XcCtx` are `WormholeScan` / `WormholeTransfer` from [[wiki/xc-sdk\|xc-sdk]].

## Transfer Flow

1. `createXcContext()` builds `HydrationConfigService` from `assetsMap` / `chainsMap` / `routesMap`, a `Wallet` with `validations`, and the Wormhole scan/transfer clients
2. `wallet.transfer(asset, srcAddr, srcChain, dstAddr, dstChain)` finds routes and creates platform adapters
3. `transfer.estimateFee(amount)` calculates source + destination fees with optional fee swaps
4. `transfer.buildCalls(amount)` generates the **ordered** platform-specific call sequence (prerequisites first, transfer last)
5. Platform signer signs and submits each call
6. `wormhole.scan` / `wormhole.transfer` ([[wiki/xc-sdk\|xc-sdk]]) track the journey and expose the manual redeem

## DEX registration + aliasing

```ts
// packages/xc/src/dex.ts
// chains that share another chain's runtime and reuse its dex
const dexAlias: Record<string, string> = {
  assethub_cex: 'assethub',
};
```

`registerDexes()` bootstraps the `hydration` DEX (`HydrationDex`, reusing `opts.poolCtx` or spinning up its own `createSdkContext`) and the `assethub` DEX, then registers the *same* DEX instance on every alias chain. This is what lets `assethub_cex` ([[wiki/xc-cfg\|xc-cfg]]) do fee conversion without a duplicate pool context.

## Multi-Bridge Route Selection

Routes are tagged and selected with the optional `tag` argument of `ConfigBuilder.build()`. Current tags: `Basejump`, `Wormhole`, `Ntt`, `NttExecutor`, `Relayer`, `Snowbridge`, `SnowbridgeV1`.

## Supported Protocols & Bridges

- **[[wiki/xcm\|xcm]]** — Polkadot cross-consensus messaging
- **[[wiki/wormhole\|wormhole]] NTT** — Native Token Transfers, direct Hydration ↔ Ethereum / Base / Solana / Sui (**replaced MRL**, which routed through Moonbeam)
- **[[wiki/snowbridge\|snowbridge]]** — trustless Polkadot↔Ethereum, V1 and V2 side by side
- **Basejump** — EVM bridge on Base

## Supported Platforms

Substrate (Polkadot + Kusama parachains), EVM (Ethereum, Base, Hydration EVM), Solana, Sui.

## Reference docs (`packages/xc/docs/`)

| Doc | Covers |
|---|---|
| `overview.md` | XC stack architecture, the route DSL, xc-core / xc-cfg / xc-sdk layouts, navigation cheatsheet, conventions & gotchas |
| `ntt.md` | the definitive NTT reference — model, registry, transfer flow, signer paths & binding, rate limits, tracking/claim, executor vs self-redeem, per-chain support matrix, open items |
| `chain-native-balances.md` | declarative balance/min model, per-platform typing, how to add a chain |
| `unidirectional-routes.md` | phased spec for one-way routes, chain-level balance registry, the `reversible` flag |

Treat `ntt.md` and `unidirectional-routes.md` as **specs that partly lead the code** — e.g. `ntt.md` describes `ContractConfig.prior`/`.follow` slots and commented-out Sui routes, neither of which matches HEAD.

## Dependencies

Peers: `@galacticcouncil/xc-cfg >=2.1.0`, `xc-core >=2.1.0`, `xc-sdk >=2.1.0`. Also imports `createSdkContext` from [[wiki/sdk-next\|sdk-next]] for the fallback Hydration DEX bootstrap.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
