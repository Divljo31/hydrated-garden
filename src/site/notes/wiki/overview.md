---
{"dg-publish":true,"permalink":"/wiki/overview/","title":"Wiki Overview","tags":["gardenEntry"],"dg-note-properties":{"type":"overview","title":"Wiki Overview","last_updated":"2026-04-20"}}
---


# Overview

This wiki covers the [[wiki/hydration\|hydration]] protocol — the largest DeFi platform on [[wiki/polkadot\|polkadot]], built as a purpose-built Layer-1 parachain using Substrate.

## The Big Picture

Hydration's core thesis is **vertical integration of DeFi primitives**. Rather than deploying isolated smart contracts for trading, lending, and stablecoins, Hydration builds all three as native runtime pallets on a dedicated blockchain. This enables composability that contract-based systems cannot match — for example, using an [[omnipool\|omnipool]] LP position as collateral in [[wiki/hydration-borrow\|hydration-borrow]], or paying transaction fees in any Omnipool asset.

The three pillars are:

1. **Trading** — anchored by the [[omnipool\|omnipool]], a single-pool AMM where every asset pairs with [[wiki/lrna\|lrna]] (the internal hub token). Supplemented by [[wiki/stableswap\|stableswap]] pools, [[wiki/xyk-pools\|xyk-pools]], [[wiki/dca\|dca]], [[wiki/otc-trading\|otc-trading]], [[wiki/lbp\|lbp]], and the upcoming [[wiki/ice\|ice]] intent engine.

2. **Lending/Borrowing** — [[wiki/hydration-borrow\|hydration-borrow]], an Aave v3 fork running on [[wiki/pallet-frontier\|pallet-frontier]]'s EVM layer, with native integration to the Omnipool and HOLLAR.

3. **Stable Value** — [[wiki/hollar\|hollar]], an over-collateralized stablecoin built on GHO's architecture, with an asymmetric stability module (HSM) and partial automated liquidations.

## The Omnipool

The [[omnipool\|omnipool]] is the protocol's flagship innovation. By consolidating all liquidity into one pool with [[wiki/lrna\|lrna]] as the routing hub, it eliminates the fragmentation of traditional pair-based DEXes. Key characteristics: single-sided LP deposits, [[wiki/nft-lp-positions\|NFT-based LP positions]], [[wiki/dynamic-fees\|dynamic fees]], and a layered security architecture including [[wiki/circuit-breaker\|circuit breakers]], [[wiki/price-barrier\|price barriers]], [[wiki/ema-oracle\|EMA oracles]], and [[wiki/tradability-flags\|per-asset operation controls]].

The [[wiki/impermanent-loss\|impermanent-loss]] profile is distinctive: because LRNA acts as a weighted price index of the entire pool, market-correlated assets experience lower IL than in a traditional stablecoin pair. [[wiki/protocol-owned-liquidity\|POL]] accumulates when LPs withdraw at a loss, acting as a permanent liquidity floor.

## The SDK

The Galactic SDK (`galacticcouncil/sdk`) is the developer toolkit for building on Hydration. It's a TypeScript monorepo with 16 published packages:

- **[[wiki/sdk-next\|sdk-next]]** is the trading SDK — it provides a [[wiki/smart-order-router\|smart-order-router]] that finds optimal multi-hop paths across all pool types using BFS, plus DCA/TWAP scheduling and transaction building. Pool math runs as WASM modules compiled from Rust for deterministic, high-performance calculations.

- **The XC stack** ([[wiki/xc-package\|xc-package]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xc-scan\|xc-scan]]) handles cross-chain transfers across Substrate, EVM, Solana, and Sui chains. Routes are pre-configured per chain pair and support [[wiki/xcm\|xcm]], [[wiki/wormhole\|wormhole]], [[wiki/snowbridge\|snowbridge]], and (as of Apr 2026) **Basejump** for multi-bridge routing. Tag-based route selection lets callers pick between multiple bridges for the same asset pair.

- **[[wiki/sdk-common\|sdk-common]]** and **[[wiki/sdk-descriptors\|sdk-descriptors]]** provide shared utilities and type-safe chain metadata.

- **[[wiki/route-suggester\|route-suggester]]** is a Rust crate for high-performance route discovery, using the same BFS algorithm as the TypeScript router.

## The Node (`hydration-node`)

The parachain runtime lives in `galacticcouncil/hydration-node`. It's a Substrate node built on a fork of Polkadot SDK (`polkadot-stable2503-11-patch2`) with **38 custom pallets** wired together in `runtime/hydradx/src/lib.rs` (current spec_version 411, Rust 1.84.1).

The pallet set maps directly onto the three pillars:

- **AMM core** — [[wiki/pallet-omnipool\|pallet-omnipool]], [[wiki/pallet-stableswap\|pallet-stableswap]], [[wiki/pallet-xyk\|pallet-xyk]], [[wiki/pallet-lbp\|pallet-lbp]], plus [[wiki/pallet-route-executor\|pallet-route-executor]] for multi-hop routing.
- **Risk & infrastructure** — [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]], [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]], [[wiki/pallet-dynamic-evm-fee\|pallet-dynamic-evm-fee]], [[wiki/pallet-ema-oracle\|pallet-ema-oracle]], [[wiki/pallet-asset-registry\|pallet-asset-registry]], [[wiki/pallet-xcm-rate-limiter\|pallet-xcm-rate-limiter]], [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]], [[wiki/pallet-transaction-pause\|pallet-transaction-pause]].
- **Finance & orchestration** — [[wiki/pallet-dca\|pallet-dca]], [[wiki/pallet-staking\|pallet-staking]], [[wiki/pallet-hsm\|pallet-hsm]] (HOLLAR stability), [[wiki/pallet-otc\|pallet-otc]] / [[wiki/pallet-otc-settlements\|pallet-otc-settlements]], [[wiki/pallet-referrals\|pallet-referrals]], [[wiki/pallet-bonds\|pallet-bonds]], [[wiki/pallet-liquidation\|pallet-liquidation]], and three liquidity-mining pallets ([[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]], [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]], [[wiki/pallet-xyk-liquidity-mining\|pallet-xyk-liquidity-mining]]).
- **Currency, accounts & misc** — [[wiki/pallet-currencies\|pallet-currencies]], [[wiki/pallet-evm-accounts\|pallet-evm-accounts]], [[wiki/pallet-duster\|pallet-duster]], [[wiki/pallet-collator-rewards\|pallet-collator-rewards]], [[wiki/pallet-democracy\|pallet-democracy]], [[wiki/pallet-claims\|pallet-claims]], [[wiki/pallet-dispenser\|pallet-dispenser]], [[wiki/pallet-nft\|pallet-nft]], [[wiki/pallet-broadcast\|pallet-broadcast]], [[wiki/pallet-dispatcher\|pallet-dispatcher]], [[wiki/pallet-genesis-history\|pallet-genesis-history]], [[wiki/pallet-parameters\|pallet-parameters]], [[wiki/pallet-relaychain-info\|pallet-relaychain-info]], [[wiki/pallet-signet\|pallet-signet]].

EVM compatibility comes from [[wiki/pallet-frontier\|pallet-frontier]] plus runtime [[wiki/hydration-precompiles\|hydration-precompiles]] (call-permit for gasless EVM, flash-loan for HSM mint). Pool math is shared with the SDK via the `math/` crate; cross-pallet abstractions live in `traits/` (`OmnipoolHooks`, `AMM`, `RouteProvider`, `PriceOracle`). See [[wiki/hydration-runtime\|hydration-runtime]] for the full wiring and [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]] for the file-tree map.

## The Runtime API Layer (`polkadot-api` / papi)

Almost everything in the SDK that touches the chain goes through [[wiki/papi\|papi]] (polkadot-api), a TypeScript suite for interacting with Polkadot-based chains. It's light-client-first (built on smoldot), metadata-driven (types generated from runtime metadata via `papi add` / `papi generate`), and exposes both promise and observable APIs.

Entry point is the [[papi-client|`PolkadotClient`]] constructed from a [[wiki/papi-providers\|JsonRpcProvider]]. From there, `client.getTypedApi(descriptors)` returns a [[wiki/papi-typed-api\|TypedApi]] with `query` / `tx` / `event` / `apis` / `constants` sub-APIs — all fully typed against the specific chain's metadata. [[wiki/sdk-descriptors\|sdk-descriptors]] is the Hydration-specific descriptor bundle generated from a whitelist. For special cases there's also the [[wiki/papi-unsafe-static\|UnsafeApi and StaticApis]] and the [[wiki/papi-offline\|Offline API]] for airgap signing.

Auxiliary areas covered in the wiki: [[wiki/papi-signers\|papi-signers]] (PolkadotSigner interface, extensions, raw), [[wiki/papi-codegen\|papi-codegen]] (CLI + descriptor generation), [[wiki/papi-types\|papi-types]] (SS58String, HexString, Binary, Enum, Option, Result), [[wiki/papi-typed-codecs\|papi-typed-codecs]] (SCALE helpers), [[wiki/papi-ink\|papi-ink]] (ink! contracts), [[wiki/papi-sdks\|papi-sdks]] (Ink/Accounts/Multisig/Staking/Statement/Governance plugin SDKs), and [[wiki/papi-recipes\|papi-recipes]] (reference patterns).

## The Frontend (`hydration-ui`)

The web app at [hydration.net](https://hydration.net) lives in [[wiki/hydration-ui\|hydration-ui]] — a Yarn + Turbo monorepo with one primary app ([[hydration-ui-main-app|`apps/main`]]) and seven workspace packages. Built on **React 19 + Vite + Tanstack Router** (file-based routes) + **Tanstack Query** + **Emotion** (CSS-in-JS), with **viem** for EVM, **comlink** for workers, **sonner** for toasts, and i18next for translations.

The app is organized into [[wiki/hydration-ui-modules\|product modules]] — `trade`, `liquidity`, `borrow`, `xcm`, `wallet`, `staking`, `stats`, `transactions`, `submit-transaction`, `layout` — each mapping onto a top-level route. All chain data flows through [[wiki/hydration-ui-api\|hydration-ui-api]] (30+ per-domain React Query hooks including a new `multisig` domain) which sits on top of [[wiki/sdk-next\|sdk-next]], [[wiki/xc-sdk\|xc-sdk]], and [[wiki/papi\|papi]] (pinned to v1.23.3 via workspace resolutions, with `@polkadot-api/tx-utils` for extrinsic encoding/decoding and multisig handling). Transaction signing runs through [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] using signers supplied by [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] (PJS extensions, WalletConnect, EVM wallets, Solana); the review flow now includes a dedicated `ReviewMultisig` path, and the layout's NotificationCenter surfaces pending multisig approvals.

Supporting packages: [[hydration-ui-design-system|`@galacticcouncil/ui`]] is the Radix-based component library with style-dictionary-generated theme tokens and a Storybook; [[hydration-ui-money-market|`@galacticcouncil/money-market`]] wraps `@aave/contract-helpers` for the [[wiki/hydration-borrow\|hydration-borrow]] UI; [[hydration-ui-indexer|`@galacticcouncil/indexer`]] provides three codegen'd GraphQL clients (indexer, squid, snowbridge) for historical data; [[wiki/hydration-ui-utils\|hydration-ui-utils]] carries small shared helpers. See [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] for the full stack walkthrough.

## Governance and Tokenomics

All protocol decisions flow through [[wiki/opengov\|opengov]] — Polkadot's on-chain referendum system. [[wiki/hdx\|hdx]] holders vote on everything from asset listings to treasury deployment. The team relinquished control at launch. Staking uses a [[wiki/bonding-curve\|bonding-curve]] that rewards governance participation, and a [[wiki/referrals\|referral system]] shares fees with user referrers.

## Cross-Chain

As a [[wiki/polkadot\|polkadot]] parachain, Hydration inherits shared security and uses [[wiki/xcm\|xcm]] for native cross-chain messaging. Connections beyond Polkadot use [[wiki/snowbridge\|snowbridge]] (trustless Ethereum bridge) and [[wiki/wormhole\|wormhole]]. The XC SDK supports transfers across Polkadot, EVM chains (Ethereum, Base, Arbitrum), Solana, and Sui.

## Wiki Status

**Sources ingested:** 7 (protocol overview, Omnipool deep dive, SDK codebase, hydration-node codebase, papi docs, polkadot-api codebase, hydration-ui codebase)
**Pages:** ~117 across entities, pallets, concepts, API references, UI packages, and source summaries
**Last refresh:** 2026-04-20 — re-cloned `raw/sdk/`, `raw/hydration-node/`, `raw/hydration-ui/` and ran a diff-driven sync. See `log.md` at the vault root for details.
**Known gaps:** [[wiki/hydration-borrow\|hydration-borrow]] risk parameters and Aave-fork specifics, individual asset listings and current on-chain parameters
