---
{"dg-publish":true,"permalink":"/wiki/overview/","title":"Wiki Overview","tags":["gardenEntry"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"overview","title":"Wiki Overview","last_updated":"2026-08-15"}}
---


# Overview

This wiki covers the [[wiki/hydration\|hydration]] protocol — the largest DeFi platform on [[wiki/polkadot\|polkadot]], built as a purpose-built Layer-1 parachain using Substrate.

## The Big Picture

Hydration's core thesis is **vertical integration of DeFi primitives**. Rather than deploying isolated smart contracts for trading, lending, and stablecoins, Hydration builds all three as native runtime pallets on a dedicated blockchain. This enables composability that contract-based systems cannot match — for example, using an [[omnipool\|omnipool]] LP position as collateral in [[wiki/hydration-borrow\|hydration-borrow]], or paying transaction fees in any Omnipool asset.

The pillars are:

1. **Trading** — anchored by the [[omnipool\|omnipool]], a single-pool AMM where every asset pairs with [[wiki/lrna\|lrna]] (the internal hub token). Supplemented by [[wiki/stableswap\|stableswap]] pools, [[wiki/xyk-pools\|xyk-pools]], [[wiki/dca\|dca]], [[wiki/otc-trading\|otc-trading]], [[wiki/lbp\|lbp]], and the upcoming [[wiki/ice\|ice]] intent engine.

2. **Lending/Borrowing** — [[wiki/hydration-borrow\|hydration-borrow]], an Aave v3 fork running on [[wiki/pallet-frontier\|pallet-frontier]]'s EVM layer, with native integration to the Omnipool and HOLLAR.

3. **Stable Value** — [[wiki/hollar\|hollar]], an over-collateralized stablecoin built on GHO's architecture, with an asymmetric stability module (HSM) and partial automated liquidations.

4. **Liquid Staking** — [[wiki/gigahdx\|gigahdx]] (August 2026, the headline change of this cycle). It is vertical integration made literal: [[wiki/pallet-gigahdx\|pallet-gigahdx]] locks HDX in the staker's own account under lock id `ghdxlock`, mints internal stHDX (asset `670`), and supplies it to the lending fork, which mints non-transferable **GIGAHDX** aTokens (asset `67`) to the staker's EVM address. Yield is a pure exchange rate rather than per-position accounting, funded by a 15% cut of trade fees into the "gigapot"; [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] adds a further 25% cut paid out per referendum, split by `staked_vote × conviction`. Unstaking has a 28-day cooldown. The legacy [[wiki/pallet-staking\|pallet-staking]] path still exists but the UI has moved on.

Trade fees now bind these pillars together: [[wiki/pallet-fee-processor\|pallet-fee-processor]] is the single splitter for every Omnipool asset fee — gigahdx 15% + gigahdx-rewards 25% + staking 5% + referrals 5%, so **50% of each fee leaves the pool**.

## The Omnipool

The [[omnipool\|omnipool]] is the protocol's flagship innovation. By consolidating all liquidity into one pool with [[wiki/lrna\|lrna]] as the routing hub, it eliminates the fragmentation of traditional pair-based DEXes. Key characteristics: single-sided LP deposits, [[wiki/nft-lp-positions\|NFT-based LP positions]], [[wiki/dynamic-fees\|dynamic fees]], and a layered security architecture including [[wiki/circuit-breaker\|circuit breakers]], [[wiki/price-barrier\|price barriers]], [[wiki/ema-oracle\|EMA oracles]], and [[wiki/tradability-flags\|per-asset operation controls]].

The [[wiki/impermanent-loss\|impermanent-loss]] profile is distinctive: because LRNA acts as a weighted price index of the entire pool, market-correlated assets experience lower IL than in a traditional stablecoin pair. [[wiki/protocol-owned-liquidity\|POL]] accumulates when LPs withdraw at a loss, acting as a permanent liquidity floor.

## The SDK

The Galactic SDK (`galacticcouncil/sdk`) is the developer toolkit for building on Hydration. It's a TypeScript monorepo with 17 published packages:

- **[[wiki/sdk-next\|sdk-next]]** (v1.6.0) is the trading SDK — it provides a [[wiki/smart-order-router\|smart-order-router]] that finds optimal multi-hop paths across **Omnipool, Stableswap, XYK, LBP, HSM, and Aave** pools using BFS, plus DCA/TWAP scheduling and transaction building. Pool math runs as WASM modules compiled from Rust for deterministic, high-performance calculations. Supports historical reads (`createSdkContext({ at })`), a `SnapshotPoolCtxProvider`, `StakingClient` + `RewardClaimSimulator`, and now [[wiki/ice\|ice]] intent builders, `src/indexer/`, money-market oracles and `StableSwapPeg`.

- **The XC stack** ([[wiki/xc-package\|xc-package]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xc-scan\|xc-scan]]) handles cross-chain transfers across Substrate, EVM, Solana, and Sui chains. The bridge model was replaced this cycle: [[wiki/wormhole\|wormhole]] MRL is gone, superseded by **NTT** with direct Hydration ↔ Ethereum / Base / Solana / Sui routes (Hydration is Wormhole chain id 73; Moonbeam, Interlay, Crust, Ajuna and Laos were dropped, `assethub_cex` added). [[wiki/snowbridge\|snowbridge]] now ships V1 and V2 side by side. Every SDK package is on **papi v2**, and `xc-cfg` pre-flights Hydration's circuit-breaker deposit / withdraw limits (`HydrationDepositLimitValidation`, `HydrationWithdrawLimitValidation`).

- **[[wiki/xc-swap\|xc-swap]]** is the new sibling of the XC stack — NEAR Intent Routing, selling any Hydration asset into a NEAR-ecosystem asset in a single Hydration EVM tx (`IntentEmitter.swapAndBridge` + the off-chain 1Click API), bypassing the transfer stack entirely.

- **[[wiki/sdk-common\|sdk-common]]** and **[[wiki/sdk-descriptors\|sdk-descriptors]]** provide shared utilities and type-safe chain metadata. Note that descriptors v2.6.0 does **not** whitelist GigaHdx / GigaHdxRewards / FeeProcessor — there is no typed SDK API for liquid staking or fee routing yet.

- **[[wiki/route-suggester\|route-suggester]]** is a Rust crate for high-performance route discovery, using the same BFS algorithm as the TypeScript router.

## The Node (`hydration-node`)

The parachain runtime lives in `galacticcouncil/hydration-node`. It's a Substrate node built on a fork of Polkadot SDK (`polkadot-stable2506-11-patch`) with **42 custom pallets** wired together in `runtime/hydradx/src/lib.rs` (current spec_version 439, Rust 1.88.0). The August 2026 upgrade added `GigaHdx = 86`, `GigaHdxRewards = 87` and `FeeProcessor = 207` (no existing index moved), rewired trade-fee distribution through [[wiki/pallet-fee-processor\|pallet-fee-processor]], and migrated [[wiki/pallet-stableswap\|pallet-stableswap]] storage v1 → v2 (virtual share issuance — deriving share supply from `total_issuance` is now wrong).

The repo also carries two non-runtime top-level components: **`pepl-worker/`** ([[wiki/pepl-worker\|pepl-worker]]) — the PEPL v2 node-side liquidation worker, multi-money-market with an oracle fast path — and **`ai_skills/`** ([[wiki/note-ai-skills\|note-ai-skills]]) — the `hydration_cl0wdit` security-audit skill plus a `circuit-breaker-incident` triage skill, symlinked from both `.claude/` and `.codex/`.

The pallet set maps directly onto the pillars:

- **AMM core** — [[wiki/pallet-omnipool\|pallet-omnipool]], [[wiki/pallet-stableswap\|pallet-stableswap]], [[wiki/pallet-xyk\|pallet-xyk]], [[wiki/pallet-lbp\|pallet-lbp]], plus [[wiki/pallet-route-executor\|pallet-route-executor]] for multi-hop routing.
- **Risk & infrastructure** — [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]], [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]], [[wiki/pallet-dynamic-evm-fee\|pallet-dynamic-evm-fee]], [[wiki/pallet-ema-oracle\|pallet-ema-oracle]], [[wiki/pallet-asset-registry\|pallet-asset-registry]], [[wiki/pallet-xcm-rate-limiter\|pallet-xcm-rate-limiter]], [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]], [[wiki/pallet-transaction-pause\|pallet-transaction-pause]].
- **Liquid staking & fees** — [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]], [[wiki/pallet-fee-processor\|pallet-fee-processor]].
- **Finance & orchestration** — [[wiki/pallet-dca\|pallet-dca]] (buy schedules retired), [[wiki/pallet-staking\|pallet-staking]], [[wiki/pallet-hsm\|pallet-hsm]] (HOLLAR stability), [[wiki/pallet-otc\|pallet-otc]] / [[wiki/pallet-otc-settlements\|pallet-otc-settlements]], [[wiki/pallet-referrals\|pallet-referrals]], [[wiki/pallet-bonds\|pallet-bonds]], [[wiki/pallet-liquidation\|pallet-liquidation]], and three liquidity-mining pallets ([[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]], [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]], [[wiki/pallet-xyk-liquidity-mining\|pallet-xyk-liquidity-mining]]).
- **Currency, accounts & misc** — [[wiki/pallet-currencies\|pallet-currencies]], [[wiki/pallet-evm-accounts\|pallet-evm-accounts]], [[wiki/pallet-duster\|pallet-duster]], [[wiki/pallet-collator-rewards\|pallet-collator-rewards]], [[wiki/pallet-collator-rotation\|pallet-collator-rotation]], [[wiki/pallet-democracy\|pallet-democracy]], [[wiki/pallet-claims\|pallet-claims]], [[wiki/pallet-dispenser\|pallet-dispenser]], [[wiki/pallet-nft\|pallet-nft]], [[wiki/pallet-broadcast\|pallet-broadcast]], [[wiki/pallet-dispatcher\|pallet-dispatcher]], [[wiki/pallet-genesis-history\|pallet-genesis-history]], [[wiki/pallet-parameters\|pallet-parameters]], [[wiki/pallet-relaychain-info\|pallet-relaychain-info]], [[wiki/pallet-signet\|pallet-signet]].

EVM compatibility comes from [[wiki/pallet-frontier\|pallet-frontier]] plus runtime [[wiki/hydration-precompiles\|hydration-precompiles]] (call-permit for gasless EVM, flash-loan for HSM mint, and the new lock-manager precompile at `0x…0806` that makes GIGAHDX non-transferable). Pool math is shared with the SDK via the `math/` crate; cross-pallet abstractions live in `traits/` (`OmnipoolHooks`, `AMM`, `RouteProvider`, `PriceOracle`). See [[wiki/hydration-runtime\|hydration-runtime]] for the full wiring and [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]] for the file-tree map.

## The Runtime API Layer (`polkadot-api` / papi)

Almost everything in the SDK that touches the chain goes through [[wiki/papi\|papi]] (polkadot-api), a TypeScript suite for interacting with Polkadot-based chains. It's light-client-first (built on smoldot), metadata-driven (types generated from runtime metadata via `papi add` / `papi generate`), and exposes both promise and observable APIs.

Entry point is the [[papi-client|`PolkadotClient`]] constructed from a [[wiki/papi-providers\|JsonRpcProvider]]. From there, `client.getTypedApi(descriptors)` returns a [[wiki/papi-typed-api\|TypedApi]] with `query` / `tx` / `event` / `apis` / `constants` sub-APIs — all fully typed against the specific chain's metadata. [[wiki/sdk-descriptors\|sdk-descriptors]] is the Hydration-specific descriptor bundle generated from a whitelist. For special cases there's also the [[wiki/papi-unsafe-static\|UnsafeApi and StaticApis]] and the [[wiki/papi-offline\|Offline API]] for airgap signing.

Auxiliary areas covered in the wiki: [[wiki/papi-signers\|papi-signers]] (PolkadotSigner interface, extensions, raw), [[wiki/papi-codegen\|papi-codegen]] (CLI + descriptor generation), [[wiki/papi-types\|papi-types]] (SS58String, HexString, Binary, Enum, Option, Result), [[wiki/papi-typed-codecs\|papi-typed-codecs]] (SCALE helpers), [[wiki/papi-ink\|papi-ink]] (ink! contracts), [[wiki/papi-sdks\|papi-sdks]] (Ink/Accounts/Multisig/Staking/Statement/Governance plugin SDKs), and [[wiki/papi-recipes\|papi-recipes]] (reference patterns).

## The Frontend (`hydration-ui`)

The web app at [hydration.net](https://hydration.net) lives in [[wiki/hydration-ui\|hydration-ui]] — a Yarn + Turbo monorepo with one primary app ([[hydration-ui-main-app|`apps/main`]]) and seven workspace packages. Built on **React 19 + Vite 8 (Rolldown) + Tanstack Router** (file-based routes) + **Tanstack Query** + **Emotion** (CSS-in-JS), with an MDX pipeline (`@mdx-js/rollup` + `remark-gfm`), **viem** for EVM, **comlink** for workers, **sonner** for toasts, zod 4, and i18next for translations. No wagmi anywhere.

The app is organized into 13 [[wiki/hydration-ui-modules\|product modules]] — `trade`, `liquidity`, `borrow`, `xcm`, `wallet`, `staking`, `strategies`, `onramp`, `governance`, `stats`, `transactions`, `submit-transaction`, `layout` — each mapping onto a top-level route. Three are new: `strategies` (BIL RWA vault, ERC-4626 + ERC-7540 async redeem, plus Hollar stable bonds sold via OTC offers), `onramp` (Kraken / Binance / Kucoin / Coinbase / Gate.io + bank wizard, routed through the new `assethub_cex` chain), and `governance` (stub, nav entry still commented out). **`/staking` now serves the [[wiki/gigahdx\|gigahdx]] liquid-staking UI** — legacy staking is demoted to `/staking-old` — backed by `api/gigaStake.ts` + `api/gigaApr.ts`.

All chain data flows through [[wiki/hydration-ui-api\|hydration-ui-api]] (30+ per-domain React Query hooks) which sits on top of [[wiki/sdk-next\|sdk-next]] (1.6.0), the xc-* packages (2.x), and [[wiki/papi\|papi]] (`^2.1.7` via workspace resolution). Transaction signing runs through [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] using signers supplied by [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] (PJS extensions, WalletConnect, EVM wallets, Solana, plus new Near and Zcash modes and a persisted address book). The xcm module reflects the NTT migration (`XcmTag.NttExecutor` in `BRIDGE_PROVIDER_TAGS` — ordering is load-bearing; `utils/wormhole.ts` deleted) and now surfaces cross-chain circuit-breaker deposit / withdraw limits. Wallet transaction history was deleted in favour of `trade/orders/TradeOrdersHistory` plus deep links into the **Neckwork** explorer, which also replaced Subscan as the default explorer.

Supporting packages: [[hydration-ui-design-system|`@galacticcouncil/ui`]] is the Radix-based component library with style-dictionary-generated theme tokens and a Storybook (10.3); [[hydration-ui-money-market|`@galacticcouncil/money-market`]] wraps `@aave/contract-helpers` for the [[wiki/hydration-borrow\|hydration-borrow]] UI and now covers **three markets** (`CustomMarket.{hydration_v3, gigahdx_v3, bil_v3}`, with `MoneyMarketProvider` taking `market` instead of `env` — breaking); [[hydration-ui-indexer|`@galacticcouncil/indexer`]] provides **three** codegen'd GraphQL clients (indexer, squid, multix — the snowbridge client is gone), with charts migrating from squid to **Grafana SQL**; [[wiki/hydration-ui-utils\|hydration-ui-utils]] carries shared helpers including `basejumpscan`. See [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] for the full stack walkthrough.

## Governance and Tokenomics

All protocol decisions flow through [[wiki/opengov\|opengov]] — Polkadot's on-chain referendum system. [[wiki/hdx\|hdx]] holders vote on everything from asset listings to treasury deployment. The team relinquished control at launch. Legacy staking uses a [[wiki/bonding-curve\|bonding-curve]] that rewards governance participation; [[wiki/gigahdx\|gigahdx]] is the current path, paying conviction-weighted rewards per completed referendum via [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]], and a [[wiki/referrals\|referral system]] shares fees with user referrers.

## Cross-Chain

As a [[wiki/polkadot\|polkadot]] parachain, Hydration inherits shared security and uses [[wiki/xcm\|xcm]] for native cross-chain messaging. Connections beyond Polkadot use [[wiki/snowbridge\|snowbridge]] (trustless Ethereum bridge, V1 + V2) and [[wiki/wormhole\|wormhole]] **NTT** — direct Hydration ↔ Ethereum / Base / Solana / Sui, with the Moonbeam-routed MRL path retired. [[wiki/xc-swap\|xc-swap]] adds NEAR Intent Routing on top, as a separate single-tx path rather than a bridge route.

## Wiki Status

**Sources ingested:** 7 (protocol overview, Omnipool deep dive, SDK codebase, hydration-node codebase, papi docs, polkadot-api codebase, hydration-ui codebase)
**Pages:** ~130 across entities, pallets, concepts, API references, UI packages, and source summaries
**Last refresh:** 2026-08-15 — re-cloned `raw/sdk/`, `raw/hydration-node/`, `raw/hydration-ui/` and ran a diff-driven sync (hydration-node `7722ff4` → `cc1bc97`, sdk `96b4b0e` → `c57d172`, hydration-ui `90d653f` → `b469e8a`). ~40 pages updated, 7 created ([[wiki/gigahdx\|gigahdx]], [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]], [[wiki/pallet-fee-processor\|pallet-fee-processor]], [[wiki/pepl-worker\|pepl-worker]], [[wiki/note-ai-skills\|note-ai-skills]], [[wiki/xc-swap\|xc-swap]]). Headline changes: GigaHDX liquid staking and the rewired trade-fee split (node spec 419 → 439, 42 pallets); Wormhole MRL → NTT across SDK and UI; Snowbridge removed from the UI; `/staking` is now GigaHDX. See `log.md` at the vault root for details.
**Known gaps:** [[wiki/hydration-borrow\|hydration-borrow]] risk parameters and Aave-fork specifics, individual asset listings and current on-chain parameters
**Known hazard:** the SDK's `StableSwapClient` still derives share supply from `Tokens.TotalIssuance(shareAssetId)`, which is wrong under [[wiki/pallet-stableswap\|pallet-stableswap]] storage v2 — unfixed upstream at `c57d172`.
