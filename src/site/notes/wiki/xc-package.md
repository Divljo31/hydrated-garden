---
{"dg-publish":true,"permalink":"/wiki/xc-package/","title":"xc (Cross-Chain Package)","tags":["sdk","cross-chain","transfers","xc"],"dg-note-properties":{"type":"entity","entity_kind":"product","title":"xc (Cross-Chain Package)","tags":["sdk","cross-chain","transfers","xc"],"source_count":1,"last_updated":"2026-04-20"}}
---


# xc (Cross-Chain Package)

**TL;DR:** `@galacticcouncil/xc` is the batteries-included entry point for cross-chain transfers, a context factory wiring [[wiki/xc-core\|xc-core]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-sdk\|xc-sdk]], and [[wiki/xc-scan\|xc-scan]] into a single unified API. Supports multi-bridge route selection via tags.

## Quick Start

```ts
import { createXcContext } from '@galacticcouncil/xc';
const xc = await createXcContext();
```

Can share pool context with [[wiki/sdk-next\|sdk-next]]:
```ts
const sdk = await createSdkContext(client);
const xc = await createXcContext(sdk.ctx.pool);
```

Returns: `{ config, wallet, wormhole: { scan, transfer } }`

## Architecture

```
xc (this package) ← Start here
├── xc-sdk  ← Wallet, transfers, fee swaps
├── xc-cfg  ← Route configs, DEX, bridge builders (Basejump, Snowbridge, Wormhole)
└── xc-core ← Core types, chain & asset definitions, bridge primitives
```

## Transfer Flow

1. `createXcContext()` initializes config, wallet, and bridge clients
2. `wallet.transfer(asset, srcAddr, srcChain, dstAddr, dstChain)` finds routes and creates platform adapters
3. `transfer.estimateFee(amount)` calculates source + destination fees with optional fee swaps
4. `transfer.buildCall(amount)` generates platform-specific transaction (Extrinsic/Contract/Program/Move)
5. Platform signer signs and submits
6. [[wiki/xc-scan\|xc-scan]] tracks the journey across chains

## Multi-Bridge Route Selection

When multiple bridges exist for the same asset pair, routes are tagged (e.g., `Tag.Basejump`, `Tag.Wormhole`) and can be selected via the optional `tag` parameter in `ConfigBuilder.build()`.

## Supported Protocols & Bridges

- **[[wiki/xcm\|xcm]]** — Polkadot cross-consensus messaging
- **Basejump** — EVM bridge for Wormhole integration (new in Apr 2026)
- **[[wiki/wormhole\|wormhole]]** — Multi-chain bridge via MRL
- **[[wiki/snowbridge\|snowbridge]]** — Trustless Polkadot↔Ethereum bridge

## Supported Platforms

Substrate (Polkadot parachains), EVM (Ethereum, Base, Arbitrum), Solana, Sui.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
