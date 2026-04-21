---
{"dg-publish":true,"permalink":"/wiki/xc-cfg/","title":"xc-cfg","tags":["sdk","cross-chain","configuration","routing"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"xc-cfg","tags":["sdk","cross-chain","configuration","routing"],"source_count":1,"last_updated":"2026-04-20"}}
---


# xc-cfg

**TL;DR:** `@galacticcouncil/xc-cfg` provides pre-built route configurations for multi-chain transfers and DEX integrations (HydrationDex, AssetHub) for fee asset conversion, including contract builders for [[wiki/snowbridge\|snowbridge]], [[wiki/wormhole\|wormhole]], and Basejump bridges.

## Route Configs

Pre-configured routes for chain pairs across:
- **Polkadot:** [[wiki/hydration\|hydration]], [[wiki/asset-hub\|asset-hub]], Bifrost, Moonbeam
- **Kusama:** Basilisk, AssetHub
- **EVM:** Ethereum, Base, Arbitrum
- **Solana, Sui**

Defines which assets can be transferred between which chains, including fee structures, minimum amounts, execution builders (extrinsics, contracts, programs, moves), and bridge tags for multi-bridge routes.

## Bridge Builders

Contract builders for EVM bridge execution:

### Basejump
Supports `bridgeViaWormhole()` method via Basejump contract. Address is registered on EVM chains (e.g., Base).

### Snowbridge
Trustless Polkadot↔Ethereum bridge with generic execution.

### Wormhole
Multi-chain bridge with circle-backed USDC.

See `packages/xc-cfg/src/builders/contracts/` for implementations.

## DEX Integrations

### HydrationDex
The primary DEX integration, used for fee asset conversion:
- `getQuote()` — get swap quote for asset conversion
- `getCalldata()` — generate transaction for swap
- Supports pool types: Aave, [[omnipool\|Omni]], [[wiki/stableswap\|Stable]], [[wiki/xyk-pools\|XYK]]
- Uses the [[wiki/smart-order-router\|smart-order-router]] from [[wiki/sdk-next\|sdk-next]]
- Fallback pricing via `MultiTransactionPayment` runtime storage

### AssetHub DEX
Simpler DEX for basic swaps between parachains.

## Tag-Based Route Selection

Routes can be tagged (e.g., `Tag.Basejump`, `Tag.Wormhole`) to enable multi-bridge paths for the same asset pair. `ConfigBuilder.build(assetOnDest, tag?)` accepts an optional tag to select a specific bridge route.

## Dependencies

Depends on [[wiki/xc-core\|xc-core]] for types and [[wiki/sdk-next\|sdk-next]] (peer) for the trading router.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
