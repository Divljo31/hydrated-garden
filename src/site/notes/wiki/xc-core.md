---
{"dg-publish":true,"permalink":"/wiki/xc-core/","title":"xc-core","tags":["sdk","cross-chain","types","definitions"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"product","title":"xc-core","tags":["sdk","cross-chain","types","definitions"],"source_count":1,"last_updated":"2026-04-20"}}
---


# xc-core

**TL;DR:** `@galacticcouncil/xc-core` provides foundational types and interfaces for cross-chain transfers — chain models, asset definitions, and route configuration — depended on by all other XC packages.

## Chain Model

Chains are modeled by ecosystem and type:

**Ecosystems:** Ethereum, Polkadot, Kusama, Solana, Sui

**Chain types:** Parachain, EvmParachain, EvmChain, SolanaChain, SuiChain

Each chain maps assets with chain-specific IDs and decimals, can have a DEX attached (via `registerDex()`), tracks minimum deposit amounts, and provides type-checking methods (`isEvm()`, `isSubstrate()`, etc.).

EVM chains can optionally define bridge integrations: `basejump`, `snowbridge`, or `wormhole`.

## Asset Model

Assets have a unique `key` (e.g., `'eth'`, `'dai'`) and an `originSymbol` (e.g., `'ETH'`). `AssetAmount` extends this with amount (bigint), decimals, and origin chain reference.

## Route Configuration

Routes are organized hierarchically: `ChainRoutes` → `AssetRoute[]`. Each route defines:
- **Source:** asset, balance query builder, fee config, minimum amount
- **Destination:** chain, asset, destination fee config
- **Transact** (optional): intermediate chain for multi-step transfers
- **Execution builders:** ExtrinsicConfigBuilder (Substrate), ContractConfigBuilder (EVM), ProgramConfigBuilder (Solana), MoveConfigBuilder (Sui)
- **Tags** (optional): bridge selection tags (e.g., `Basejump`, `Wormhole`, `Snowbridge`)

Routes are keyed by: `${sourceAsset.key}-${destChain.key}-${destAsset.key}`. When multiple routes exist for the same key, `build()` accepts an optional `tag` parameter to select a specific bridge.

## Bridge Primitives

**Basejump** — EVM native bridge for Wormhole integration. Accessed via `Basejump.fromChain(evmChain)` to get bridge contract address.

**Snowbridge** — Trustless Polkadot↔Ethereum bridge.

**Wormhole** — Multi-chain bridge with circle-backed USDC support.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
