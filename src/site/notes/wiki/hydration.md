---
{"dg-publish":true,"permalink":"/wiki/hydration/","title":"Hydration","tags":["protocol","polkadot","defi","parachain"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"project","title":"Hydration","tags":["protocol","polkadot","defi","parachain"],"source_count":3,"last_updated":"2026-08-15"}}
---


# Hydration

**TL;DR:** Hydration (formerly HydraDX) is the largest DeFi protocol on Polkadot by TVL (~$330M+), a purpose-built Layer-1 parachain consolidating trading (via [[omnipool\|omnipool]]), lending/borrowing ([[wiki/hydration-borrow\|hydration-borrow]]), stable value ([[wiki/hollar\|hollar]]) and — since Aug 2026 — liquid staking ([[wiki/gigahdx\|gigahdx]]), all integrated at the runtime level.

## Mission

Consolidate the foundational pillars of DeFi into a single, vertically integrated environment (three at launch, four since GigaHDX):

1. **Trading** — via the [[omnipool\|omnipool]] AMM and ancillary tools ([[wiki/dca\|dca]], [[wiki/otc-trading\|otc-trading]], [[wiki/lbp\|lbp]], [[wiki/ice\|ice]])
2. **Lending/Borrowing** — via [[wiki/hydration-borrow\|hydration-borrow]] (Aave v3 fork; three markets since Aug 2026)
3. **Stable Value** — via [[wiki/hollar\|hollar]], the native over-collateralized stablecoin
4. **Liquid Staking** — via [[wiki/gigahdx\|gigahdx]] (Aug 2026): stake [[wiki/hdx\|hdx]] → GIGAHDX, a yield-bearing aToken on Hydration's own Aave fork, usable as money-market collateral. It takes 40% of every Omnipool asset fee through [[wiki/pallet-fee-processor\|pallet-fee-processor]] and replaces the legacy NFT staking as the default `/staking` product

The design philosophy is vertical integration: the pillars interact natively at the runtime level, enabling composability that smart-contract-based systems cannot replicate — for example, using an Omnipool LP position as collateral for borrowing.

## Technical Foundation

Hydration is a Polkadot parachain, inheriting shared security from the relay chain. The runtime is composed of Substrate pallets (Rust modules), not EVM smart contracts. This gives deeper integration and stronger security guarantees than contract-based protocols. EVM compatibility is available via [[wiki/pallet-frontier\|pallet-frontier]] for Solidity tooling and MetaMask support.

Cross-chain interoperability uses [[wiki/xcm\|xcm]], with bridges to Ethereum via [[wiki/snowbridge\|snowbridge]] (trustless, V1 + V2) and [[wiki/wormhole\|wormhole]] **NTT** — direct to Ethereum / Base / Solana / Sui since Aug 2026 (the Moonbeam MRL hop is gone). [[wiki/xc-swap\|xc-swap]] adds NEAR Intent Routing outside the transfer stack.

## Governance

Fully on-chain via [[wiki/opengov\|opengov]]. All [[wiki/hdx\|hdx]] holders can vote on protocol changes, [[omnipool\|omnipool]] listings, and treasury management. No multisigs or council intermediaries — the founding team relinquished control at mainnet launch. The Technical Committee retains only emergency intervention powers (e.g., pausing asset trading via [[wiki/tradability-flags\|tradability-flags]]).

## Treasury

One of the most diversified on-chain treasuries in Polkadot. Seeded with ~22.5M DAI from the March 2021 LBP. Holds BTC, ETH, DOT, stablecoins, and tokenized gold. Actively deployed across Hydration's own products. All decisions via OpenGov referenda.

## Codebase

| Component | Repository | Wiki Pages |
|-----------|-----------|------------|
| Runtime | `galacticcouncil/hydration-node` (Substrate/Rust) | [[wiki/hydration-runtime\|hydration-runtime]], [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]] |
| AAVE v3 | `galacticcouncil/aave-v3-deploy` (EVM/Solidity) | — |
| UI | `galacticcouncil/hydration-ui` (Frontend) | [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-main-app\|hydration-ui-main-app]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] |
| SDK | `galacticcouncil/sdk` (TypeScript) | [[wiki/sdk-next\|sdk-next]], [[wiki/xc-package\|xc-package]], [[wiki/sdk-common\|sdk-common]], [[wiki/sdk-descriptors\|sdk-descriptors]], [[wiki/route-suggester\|route-suggester]] |

The SDK is a monorepo with 17 published packages. [[wiki/sdk-next\|sdk-next]] provides the [[wiki/smart-order-router\|smart-order-router]] for trading, and the XC stack ([[wiki/xc-package\|xc-package]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xc-scan\|xc-scan]], [[wiki/xc-swap\|xc-swap]]) handles cross-chain transfers across Substrate, EVM, Solana, Sui and NEAR. The runtime is at **spec version 439**, 42 pallets.

## Security

Active auditing program, Immunefi bug bounty, runtime-level risk controls ([[wiki/circuit-breaker\|circuit-breaker]], [[wiki/price-barrier\|price-barrier]], [[wiki/dynamic-fees\|dynamic-fees]], liquidity caps). Audited by [[wiki/runtime-verification\|runtime-verification]], [[wiki/blockscience\|blockscience]], and [[wiki/code4rena\|code4rena]].

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
