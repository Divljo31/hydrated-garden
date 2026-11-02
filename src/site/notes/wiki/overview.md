---
{"dg-publish":true,"permalink":"/wiki/overview/","title":"Wiki Overview","tags":["gardenEntry"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"overview","title":"Wiki Overview","last_updated":"2026-11-02"}}
---


# Overview

This wiki covers the [[wiki/hydration\|hydration]] protocol — the largest DeFi platform on [[wiki/polkadot\|polkadot]], built as a purpose-built Layer-1 parachain using Substrate.

## The Big Picture

Hydration's core thesis is **vertical integration of DeFi primitives**. Rather than deploying isolated smart contracts for trading, lending, and stablecoins, Hydration builds all three as native runtime pallets on a dedicated blockchain. This enables composability that contract-based systems cannot match — for example, using an [[omnipool\|omnipool]] LP position as collateral in [[wiki/hydration-borrow\|hydration-borrow]], or paying transaction fees in any Omnipool asset.

The pillars are:

1. **Trading** — anchored by the [[omnipool\|omnipool]], a single-pool AMM where every asset pairs with [[wiki/lrna\|lrna]] (the internal hub token). Supplemented by [[wiki/stableswap\|stableswap]] pools, [[wiki/xyk-pools\|xyk-pools]], [[wiki/dca\|dca]], [[wiki/otc-trading\|otc-trading]], [[wiki/lbp\|lbp]], the live [[wiki/ice\|ice]] intent engine (legacy DCA is migrating into it), and curated [[wiki/uniswap-v3\|uniswap-v3]] pools on the EVM side.

2. **Lending/Borrowing** — [[wiki/hydration-borrow\|hydration-borrow]], an Aave v3 fork running on [[wiki/pallet-frontier\|pallet-frontier]]'s EVM layer, with native integration to the Omnipool and HOLLAR.

3. **Stable Value** — [[wiki/hollar\|hollar]], an over-collateralized stablecoin built on GHO's architecture, with an asymmetric stability module (HSM) and partial automated liquidations.

4. **Liquid Staking** — [[wiki/gigahdx\|gigahdx]] (August 2026, the headline change of this cycle). It is vertical integration made literal: [[wiki/pallet-gigahdx\|pallet-gigahdx]] locks HDX in the staker's own account under lock id `ghdxlock`, mints internal stHDX (asset `670`), and supplies it to the lending fork, which mints non-transferable **GIGAHDX** aTokens (asset `67`) to the staker's EVM address. Yield is a pure exchange rate rather than per-position accounting, funded by a 15% cut of trade fees into the "gigapot"; [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] adds a further 25% cut paid out per referendum, split by `staked_vote × conviction`. Unstaking has a 28-day cooldown. The legacy [[wiki/pallet-staking\|pallet-staking]] path still exists but the UI has moved on.

Trade fees now bind these pillars together: [[wiki/pallet-fee-processor\|pallet-fee-processor]] is the single splitter for every Omnipool asset fee — gigahdx 15% + gigahdx-rewards 25% + staking 5% + referrals 5%, so **50% of each fee leaves the pool**.

## The Omnipool

The [[omnipool\|omnipool]] is the protocol's flagship innovation. By consolidating all liquidity into one pool with [[wiki/lrna\|lrna]] as the routing hub, it eliminates the fragmentation of traditional pair-based DEXes. Key characteristics: single-sided LP deposits, [[wiki/nft-lp-positions\|NFT-based LP positions]], [[wiki/dynamic-fees\|dynamic fees]], and a layered security architecture including [[wiki/circuit-breaker\|circuit breakers]], [[wiki/price-barrier\|price barriers]], [[wiki/ema-oracle\|EMA oracles]], and [[wiki/tradability-flags\|per-asset operation controls]].

The [[wiki/impermanent-loss\|impermanent-loss]] profile is distinctive: because LRNA acts as a weighted price index of the entire pool, market-correlated assets experience lower IL than in a traditional stablecoin pair. [[wiki/protocol-owned-liquidity\|POL]] accumulates when LPs withdraw at a loss, acting as a permanent liquidity floor.

## The SDK

The Galactic SDK (`galacticcouncil/sdk`, synced at `d16ff85`) is the developer toolkit for building on Hydration — a TypeScript monorepo of published packages, all on **papi v2**:

- **[[wiki/sdk-next\|sdk-next]]** (v2.5.0) is the trading SDK. Pools live under `pool/amm/*` and are kept current by one event-driven sync engine (`PoolSync` + `EventBus`); `'finalized'` reads are static snapshots, only `'best'` subscribes. The [[wiki/smart-order-router\|smart-order-router]] routes across **Omnipool, Stableswap, XYK, LBP, HSM, Aave (three markets: main, BIL, GIGAHDX) and [[wiki/uniswap-v3\|uniswap-v3]]**. Also: DCA (TWAP buy removed), historical reads (`createSdkContext({ at })`), `SnapshotPoolCtxProvider`, `StakingClient` + `RewardClaimSimulator`, `MmRouting`, multicall EVM reads, [[wiki/ice\|ice]] intent builders. Pool math runs as WASM compiled from Rust.

- **The XC stack** ([[wiki/xc-package\|xc-package]], [[wiki/xc-sdk\|xc-sdk]], [[wiki/xc-cfg\|xc-cfg]], [[wiki/xc-core\|xc-core]], [[wiki/xc-scan\|xc-scan]]) handles transfers across Substrate, EVM, Solana, Sui, **NEAR and Zcash**. Bridges: [[wiki/wormhole\|wormhole]] **NTT** (Ethereum, Base, Solana, Sui, HyperEVM, Robinhood, NEAR, ZEC), [[wiki/snowbridge\|snowbridge]] V1 + V2, Basejump, and the Omni corridor. `xc-cfg` pre-flights Hydration circuit-breaker deposit / withdraw limits plus custody, Omni and EVM-value validations.

- **[[wiki/xc-swap\|xc-swap]]** (v0.9.0) buys any NEAR-Intents asset with any Hydration asset in one Hydration EVM tx — `IntentEmitter.placeOrder` → WETH → NTT → Ethereum `IntentReceiver` → 1Click. On-chain half lives in WHM ([[wiki/whm-intents\|whm-intents]]).

- **[[wiki/sdk-common\|sdk-common]]** and **[[wiki/sdk-descriptors\|sdk-descriptors]]** (v2.9.0) provide shared utilities and type-safe chain metadata. ICE, Intent, GigaHdx(Rewards) and Parameters are whitelisted; FeeProcessor and LazyExecutor are not.

- **[[wiki/route-suggester\|route-suggester]]** is a Rust crate for high-performance route discovery, using the same BFS algorithm as the TypeScript router.

## The Node (`hydration-node`)

The parachain runtime lives in `galacticcouncil/hydration-node` (synced at `a35840f`). Substrate node on a Polkadot SDK fork (`polkadot-stable2506-11-patch`), Rust 1.88.0, **44 custom pallets** wired in `runtime/hydradx/src/lib.rs`, **spec_version 458**, **2-second blocks** (every block-denominated parameter is 3× shorter in wall-clock time than before `TwoSecBlocksSince`).

Top-level non-runtime components: **`pepl-worker/`** ([[wiki/pepl-worker\|pepl-worker]]) — PEPL v2 liquidation worker with DIA oracle fast path and off-chain routing — and **`ai_skills/`** ([[wiki/note-ai-skills\|note-ai-skills]]) — `hydration_cl0wdit` security audit + `circuit-breaker-incident` triage.

The pallet set maps onto the pillars:

- **AMM core** — [[wiki/pallet-omnipool\|pallet-omnipool]], [[wiki/pallet-stableswap\|pallet-stableswap]], [[wiki/pallet-xyk\|pallet-xyk]], [[wiki/pallet-lbp\|pallet-lbp]], [[wiki/pallet-route-executor\|pallet-route-executor]] (now also routes through Uniswap v3).
- **Intents (ICE)** — [[wiki/pallet-intent\|pallet-intent]] (index 98), [[wiki/pallet-ice\|pallet-ice]] (99, solver-batch settlement, `SolverMode` kill switch), [[wiki/pallet-lazy-executor\|pallet-lazy-executor]] (97, post-settlement EVM forwards). Legacy [[wiki/pallet-dca\|pallet-dca]] schedules are being migrated into ICE intents.
- **Risk & infrastructure** — [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] (incl. XCM deposit fuse, global withdraw lockdown, per-message netting), [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]], [[wiki/pallet-dynamic-evm-fee\|pallet-dynamic-evm-fee]], [[wiki/pallet-ema-oracle\|pallet-ema-oracle]], [[wiki/pallet-asset-registry\|pallet-asset-registry]], [[wiki/pallet-xcm-rate-limiter\|pallet-xcm-rate-limiter]], [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]], [[wiki/pallet-transaction-pause\|pallet-transaction-pause]], [[wiki/pallet-parameters\|pallet-parameters]] (`suspend_xcm` — any single TC member). Incident-response view: [[wiki/note-emergency-levers\|note-emergency-levers]].
- **Liquid staking & fees** — [[wiki/pallet-gigahdx\|pallet-gigahdx]], [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]], [[wiki/pallet-fee-processor\|pallet-fee-processor]].
- **Finance & orchestration** — [[wiki/pallet-staking\|pallet-staking]], [[wiki/pallet-hsm\|pallet-hsm]] (HOLLAR stability), [[wiki/pallet-otc\|pallet-otc]] / [[wiki/pallet-otc-settlements\|pallet-otc-settlements]], [[wiki/pallet-referrals\|pallet-referrals]], [[wiki/pallet-bonds\|pallet-bonds]], [[wiki/pallet-liquidation\|pallet-liquidation]], and three liquidity-mining pallets ([[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]], [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]], [[wiki/pallet-xyk-liquidity-mining\|pallet-xyk-liquidity-mining]]).
- **Currency, accounts & misc** — [[wiki/pallet-currencies\|pallet-currencies]], [[wiki/pallet-evm-accounts\|pallet-evm-accounts]], [[wiki/pallet-duster\|pallet-duster]], [[wiki/pallet-collator-rewards\|pallet-collator-rewards]], [[wiki/pallet-collator-rotation\|pallet-collator-rotation]], [[wiki/pallet-claims\|pallet-claims]], [[wiki/pallet-dispenser\|pallet-dispenser]], [[wiki/pallet-nft\|pallet-nft]], [[wiki/pallet-broadcast\|pallet-broadcast]], [[wiki/pallet-dispatcher\|pallet-dispatcher]], [[wiki/pallet-genesis-history\|pallet-genesis-history]], [[wiki/pallet-relaychain-info\|pallet-relaychain-info]], [[wiki/pallet-signet\|pallet-signet]].

EVM compatibility comes from [[wiki/pallet-frontier\|pallet-frontier]] plus runtime [[wiki/hydration-precompiles\|hydration-precompiles]] (call-permit, flash-loan for HSM mint, lock-manager `0x…0806`, Chainlink-adapter oracle addresses). Pool math is shared with the SDK via the `math/` crate; cross-pallet abstractions live in `traits/`. See [[wiki/hydration-runtime\|hydration-runtime]] and [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]].

## WHM (`galacticcouncil/whm`)

The Wormhole messaging monorepo (synced at `36ae52b`): Foundry UUPS contracts, a Solana oracle-emitter, NEAR NTT / UTXO contracts and five off-chain agents. It carries the [[wiki/whm-oracle-relay\|whm-oracle-relay]] (Kamino / Jito / wstETH / apyUSD / SPY prices into Hydration's MM oracles — renounced, no pause lever), [[wiki/whm-basejump\|whm-basejump]] (instant Ethereum → Hydration USDC), [[wiki/whm-intents\|whm-intents]] (the on-chain side of [[wiki/xc-swap\|xc-swap]]), [[wiki/whm-near-corridors\|whm-near-corridors]] (NEAR, ZEC), and [[wiki/whm-cross-chain-governance\|whm-cross-chain-governance]] (OpenGov Root → `GovernanceExecutor` on Ethereum / Robinhood / Base / HyperEVM). Address book and owners: [[wiki/whm-contracts\|whm-contracts]]. Entry: [[wiki/source-whm-codebase\|source-whm-codebase]].

## The Runtime API Layer (`polkadot-api` / papi)

Almost everything in the SDK that touches the chain goes through [[wiki/papi\|papi]] (polkadot-api), a TypeScript suite for interacting with Polkadot-based chains. It's light-client-first (built on smoldot), metadata-driven (types generated from runtime metadata via `papi add` / `papi generate`), and exposes both promise and observable APIs.

Entry point is the [[papi-client|`PolkadotClient`]] constructed from a [[wiki/papi-providers\|JsonRpcProvider]]. From there, `client.getTypedApi(descriptors)` returns a [[wiki/papi-typed-api\|TypedApi]] with `query` / `tx` / `event` / `apis` / `constants` sub-APIs — all fully typed against the specific chain's metadata. [[wiki/sdk-descriptors\|sdk-descriptors]] is the Hydration-specific descriptor bundle generated from a whitelist. For special cases there's also the [[wiki/papi-unsafe-static\|UnsafeApi and StaticApis]] and the [[wiki/papi-offline\|Offline API]] for airgap signing.

Auxiliary areas covered in the wiki: [[wiki/papi-signers\|papi-signers]] (PolkadotSigner interface, extensions, raw), [[wiki/papi-codegen\|papi-codegen]] (CLI + descriptor generation), [[wiki/papi-types\|papi-types]] (SS58String, HexString, Binary, Enum, Option, Result), [[wiki/papi-typed-codecs\|papi-typed-codecs]] (SCALE helpers), [[wiki/papi-ink\|papi-ink]] (ink! contracts), [[wiki/papi-sdks\|papi-sdks]] (Ink/Accounts/Multisig/Staking/Statement/Governance plugin SDKs), and [[wiki/papi-recipes\|papi-recipes]] (reference patterns).

## The Frontend (`hydration-ui`)

The web app at [hydration.net](https://hydration.net) lives in [[wiki/hydration-ui\|hydration-ui]] (synced at `770f868`) — a Yarn + Turbo monorepo with one app ([[hydration-ui-main-app|`apps/main`]]) and seven workspace packages, also deployed to IPFS. Stack: **React 19 + Vite + Tanstack Router + Tanstack Query + Emotion**, viem, papi 2.2.2, Node 26, Vitest in CI — see [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]].

The app has 14 [[wiki/hydration-ui-modules\|product modules]]: `trade` (Swap; **Market** tab on [[wiki/xc-swap\|xc-swap]]; **Limit** tab on [[wiki/ice\|ice]] intents; order history), `liquidity` (incl. Uniswap v3 vaults), `borrow`, `xcm` (scan-based history), `portfolio` (multichain balances + tracked wallets — replaces the deleted `wallet` module; `/wallet*` routes redirect), `tutorials`, `staking` ([[wiki/gigahdx\|gigahdx]]), `strategies`, `onramp`, `stats` (fees view only), `transactions`, `submit-transaction`, `layout`. `governance/` exists but is unrouted.

Chain data flows through [[wiki/hydration-ui-api\|hydration-ui-api]] on top of [[wiki/sdk-next\|sdk-next]] 2.5.0, the xc-* packages and [[wiki/papi\|papi]]. Signing runs through [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] with signers from [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] (PJS, WalletConnect, EVM permit, Solana, NEAR, Zcash). History and explorer data come from **Neckwork** REST plus `scan` via [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] (squid retired; GraphQL indexer kept for farms; multix for multisig). Supporting packages: [[hydration-ui-design-system|`@galacticcouncil/ui`]], [[hydration-ui-money-market|`@galacticcouncil/money-market`]] (three Aave markets), [[wiki/hydration-ui-utils\|hydration-ui-utils]].

## Governance and Tokenomics

All protocol decisions flow through [[wiki/opengov\|opengov]] — the referendum tracks plus a Technical Committee with emergency powers (gov v1 / Democracy is gone from the runtime). [[wiki/hdx\|hdx]] holders vote on everything from asset listings to treasury deployment. [[wiki/gigahdx\|gigahdx]] is the staking path, paying conviction-weighted rewards per completed referendum via [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]; legacy staking uses a [[wiki/bonding-curve\|bonding-curve]]. A [[wiki/referrals\|referral system]] shares fees with referrers. Cross-chain governance of WHM contracts: [[wiki/whm-cross-chain-governance\|whm-cross-chain-governance]].

## Cross-Chain

As a [[wiki/polkadot\|polkadot]] parachain, Hydration uses [[wiki/xcm\|xcm]] for native messaging (TC can `suspend_xcm` in an emergency once spec 458 ships; the live chain runs spec 455 — see [[wiki/note-live-chain-check\|note-live-chain-check]]). Beyond Polkadot: [[wiki/snowbridge\|snowbridge]] (V1 + V2) and [[wiki/wormhole\|wormhole]] **NTT** — Ethereum, Base, Solana, Sui, HyperEVM ([[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]]), Robinhood, NEAR and ZEC, with Hydration acting as the locking hub for HDX / HOLLAR towards Robinhood. [[wiki/whm-basejump\|whm-basejump]] gives instant Ethereum → Hydration USDC; [[wiki/xc-swap\|xc-swap]] adds NEAR-Intents swaps as a single-tx path.

## Security / incident response

[[wiki/note-security-hardening-plan\|note-security-hardening-plan]] (phased hardening plan), [[wiki/note-risk-register\|note-risk-register]] (every protocol risk with the signal to watch), [[wiki/note-live-chain-check\|note-live-chain-check]] (live state snapshot), [[wiki/note-monitoring-plan\|note-monitoring-plan]] (what to watch, by layer), [[wiki/note-collateral-map\|note-collateral-map]] (money-market collateral and its dependency chains), [[wiki/note-dependency-aware-monitoring\|note-dependency-aware-monitoring]] (design draft), [[wiki/note-dependency-registry\|note-dependency-registry]] (assets and oracles by upstream dependency), [[wiki/note-emergency-levers\|note-emergency-levers]] (what can be paused, by whom, how fast), [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]] (pre-decided responses), backed by [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]].

## Wiki Status

**Sources ingested:** 8 (protocol overview, Omnipool deep dive, SDK codebase, hydration-node codebase, hydration-ui codebase, WHM codebase, papi docs, polkadot-api codebase) + 1 web-research source (monitoring vendors)
**Last refresh:** 2026-11-02 — `raw/` re-cloned from `master`: hydration-node `cc1bc97` → `a35840f`, sdk `c57d172` → `d16ff85`, hydration-ui `b469e8a` → `770f868`, whm new at `36ae52b`. Deprecated content removed (Democracy, squid, wallet module, retired-bridge narratives, "upcoming" ICE). See `log.md`.
**Known gaps:** [[wiki/hydration-borrow\|hydration-borrow]] risk parameters and Aave-fork specifics; current on-chain parameters (registry venues need a chain snapshot); owners of the six WHM-fed Hydration oracle contracts.
**Known hazard:** the SDK's `StableSwapClient` derives share supply from `Tokens.TotalIssuance(shareAssetId)` while [[wiki/pallet-stableswap\|pallet-stableswap]] v2 uses `Stableswap.ShareIssuance` — still unfixed at `d16ff85`.
