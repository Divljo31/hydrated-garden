---
{"dg-publish":true,"permalink":"/wiki/hydration/","title":"Hydration","tags":["protocol","polkadot","defi","parachain"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"entity","entity_kind":"project","title":"Hydration","tags":["protocol","polkadot","defi","parachain"],"source_count":3,"last_updated":"2026-04-13"}}
---


# Hydration

**TL;DR:** Hydration (formerly HydraDX) is the largest DeFi protocol on Polkadot by TVL (~$330M+), a purpose-built Layer-1 parachain consolidating three pillars: trading (via [[omnipool\|omnipool]]), lending/borrowing ([[wiki/hydration-borrow\|hydration-borrow]]), and stable value ([[wiki/hollar\|hollar]]), all integrated at the runtime level.

## Mission

Consolidate the three foundational pillars of DeFi into a single, vertically integrated environment:

1. **Trading** — via the [[omnipool\|omnipool]] AMM and ancillary tools ([[wiki/dca\|dca]], [[wiki/otc-trading\|otc-trading]], [[wiki/lbp\|lbp]], [[wiki/ice\|ice]])
2. **Lending/Borrowing** — via [[wiki/hydration-borrow\|hydration-borrow]] (Aave v3 fork)
3. **Stable Value** — via [[wiki/hollar\|hollar]], the native over-collateralized stablecoin

The design philosophy is vertical integration: all three pillars interact natively at the runtime level, enabling composability that smart-contract-based systems cannot replicate — for example, using an Omnipool LP position as collateral for borrowing.

## Technical Foundation

Hydration is a Polkadot parachain, inheriting shared security from the relay chain. The runtime is composed of Substrate pallets (Rust modules), not EVM smart contracts. This gives deeper integration and stronger security guarantees than contract-based protocols. EVM compatibility is available via [[wiki/pallet-frontier\|pallet-frontier]] for Solidity tooling and MetaMask support.

Cross-chain interoperability uses [[wiki/xcm\|xcm]], with bridges to Ethereum via [[wiki/snowbridge\|snowbridge]] (trustless) and [[wiki/wormhole\|wormhole]].

## Governance

Fully on-chain via [[wiki/opengov\|opengov]]. All [[wiki/hdx\|hdx]] holders can vote on protocol changes, [[omnipool\|omnipool]] listings, and treasury management. No multisigs or council intermediaries — the founding team relinquished control at mainnet launch. The Technical Committee retains only emergency intervention powers (e.g., pausing asset trading via [[wiki/tradability-flags\|tradability-flags]]).

## Treasury

One of the most diversified on-chain treasuries in Polkadot. Seeded with ~22.5M DAI from the March 2021 LBP. Holds BTC, ETH, DOT, stablecoins, and tokenized gold. Actively deployed across Hydration's own products. All decisions via OpenGov referenda.

## Codebase

| Component | Repository | Wiki Pages |
|-----------|-----------|------------|
| Runtime | `galacticcouncil/hydration-node` (Substrate/Rust) | — |
| AAVE v3 | `galacticcouncil/aave-v3-deploy` (EVM/Solidity) | — |
| UI | `galacticcouncil/hydration-ui` (Frontend) | [[wiki/hydration-ui\|hydration-ui]], [[wiki/hydration-ui-main-app\|hydration-ui-main-app]], [[wiki/hydration-ui-modules\|hydration-ui-modules]], [[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] |
| SDK | `galacticcouncil/sdk` (TypeScript) | [[wiki/sdk-next\|sdk-next]], [[wiki/xc-package\|xc-package]], [[wiki/sdk-common\|sdk-common]], [[wiki/sdk-descriptors\|sdk-descriptors]], [[wiki/route-suggester\|route-suggester]] |

The SDK is a monorepo with 16 published packages. [[wiki/sdk-next\|sdk-next]] provides the [[wiki/smart-order-router\|smart-order-router]] for trading, and the XC stack ([[wiki/xc-package\|xc-package]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xc-scan\|xc-scan]]) handles cross-chain transfers across Substrate, EVM, Solana, and Sui.

## Security

Active auditing program, Immunefi bug bounty, runtime-level risk controls ([[wiki/circuit-breaker\|circuit-breaker]], [[wiki/price-barrier\|price-barrier]], [[wiki/dynamic-fees\|dynamic-fees]], liquidity caps). Audited by [[wiki/runtime-verification\|runtime-verification]], [[wiki/blockscience\|blockscience]], and [[wiki/code4rena\|code4rena]].

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
