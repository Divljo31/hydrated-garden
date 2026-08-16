---
{"dg-publish":true,"permalink":"/wiki/index/","title":"Wiki Index","dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"index","title":"Wiki Index","last_updated":"2026-08-15"}}
---


# Index

Master catalog of all pages in the wiki. Updated on every ingest.

## Start here

- [[wiki/overview\|overview]] — 30-second briefing: what's indexed, protocol pillars, per-repo summary
- [[wiki/routing\|routing]] — task → pages cheat sheet ("I need to X → read Y, Z")
- [[wiki/hydration\|hydration]] — the protocol itself, if you need the ground truth before routing

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]] — Foundational protocol overview: products, architecture, governance, tokenomics, cross-chain design
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]] — Deep technical reference: AMM model, swap math, LP mechanics, fees, risk controls, on-chain data model
- [[wiki/source-sdk-codebase\|source-sdk-codebase]] — Galactic SDK monorepo: trading SDK, cross-chain transfers, WASM math, tooling
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]] — Hydration parachain node: 42 pallets, runtime, precompiles, math, traits, `pepl-worker/`, `ai_skills/`
- [[wiki/source-papi-docs\|source-papi-docs]] — papi.how documentation (Vocs site, 36 markdown pages)
- [[wiki/source-polkadot-api-codebase\|source-polkadot-api-codebase]] — polkadot-api monorepo: 24 packages (client, descriptors, cli, smoldot, signers, ink, substrate-bindings)
- [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]] — Hydration frontend monorepo: React 19 + Vite 8 (Rolldown) + Tanstack Router, apps/main (13 modules) + 7 workspace packages

## Entities

### Protocol & Products
- [[wiki/hydration\|hydration]] — The protocol itself: largest DeFi on Polkadot, vertically integrated trading + lending + stablecoin + liquid staking
- [[omnipool\|omnipool]] — Flagship single-pool AMM with hub token routing
- [[wiki/hollar\|hollar]] — Native over-collateralized stablecoin ($1 peg), built on GHO architecture
- [[wiki/gigahdx\|gigahdx]] — HDX liquid staking: HDX → internal stHDX (asset 670) → GIGAHDX aTokens (asset 67) on the Aave fork, exchange-rate yield, non-transferable via lock-manager precompile
- [[wiki/hydration-borrow\|hydration-borrow]] — Aave v3 fork for lending/borrowing
- [[wiki/lrna\|lrna]] — Internal hub token (H2O) powering Omnipool routing
- [[wiki/hdx\|hdx]] — Native governance and incentive token

### Strategy Tokens
- [[wiki/gdot\|gdot]] — DOT yield-bundling strategy token (vDOT + aDOT + incentives)
- [[wiki/geth\|geth]] — ETH yield-bundling strategy token (wstETH + aETH + fees)
- [[wiki/gsol\|gsol]] — SOL yield-bundling strategy token

### polkadot-api (papi) — runtime-layer foundation
- [[wiki/papi\|papi]] — Top-level entry: light-client-first TS API, metadata-driven types, promise + observable duality
- [[wiki/papi-getting-started\|papi-getting-started]] — Install, `papi add`, first client
- [[wiki/papi-client\|papi-client]] — `PolkadotClient` interface reference (getChainSpecData, getMetadata, finalizedBlock$, getTypedApi)
- [[wiki/papi-typed-api\|papi-typed-api]] — TypedApi: query / tx / event / apis / constants / view (metadata-driven, type-safe)
- [[wiki/papi-providers\|papi-providers]] — JsonRpcProvider: smoldot, WebSocket, enhancers
- [[wiki/papi-signers\|papi-signers]] — PolkadotSigner interface, browser extensions, raw signers
- [[wiki/papi-codegen\|papi-codegen]] — `papi` CLI: add, generate, whitelist, `.papi/` output
- [[wiki/papi-types\|papi-types]] — Runtime types: SS58String, HexString, Binary, Enum, Option, Result, BigInt
- [[wiki/papi-typed-codecs\|papi-typed-codecs]] — SCALE codec helpers (Bin, Vector, Struct, Enum, Tuple)
- [[wiki/papi-ink\|papi-ink]] — ink! contract integration (getInkClient, messages, deploy)
- [[wiki/papi-sdks\|papi-sdks]] — Plugin SDKs: Ink, Accounts, Multisig, Staking, Statement, Governance
- [[wiki/papi-offline\|papi-offline]] — Offline tx signing (airgap, serverless)
- [[wiki/papi-unsafe-static\|papi-unsafe-static]] — UnsafeApi (dynamic) vs StaticApis (sync snapshot)
- [[wiki/papi-recipes\|papi-recipes]] — Recipes: simple-transfer, multi-chain, upgrade, metadata-caching

### SDK Packages (`galacticcouncil/sdk`) — 17 published packages, all on papi v2 as of August 2026
- [[wiki/sdk-next\|sdk-next]] — v1.6.0 trade router (Omni / Stableswap / XYK / LBP / HSM / Aave), smart order router, DCA/TWAP, tx building; historical reads (`at`), `SnapshotPoolCtxProvider`, `StakingClient`, ICE intent builders, `src/indexer/`, money-market oracles, `StableSwapPeg`, EVM log parsers
- [[wiki/sdk-common\|sdk-common]] — Shared utilities (big numbers, XCM, EVM, substrate RPC via `createWsClient`)
- [[wiki/sdk-descriptors\|sdk-descriptors]] — Hydration papi type-safe metadata descriptors (v2.6.0, `hydrationIce` chain + `wasm/ice/ice.wasm`; GigaHdx / GigaHdxRewards / FeeProcessor present in metadata but NOT whitelisted)
- [[wiki/xc-package\|xc-package]] — Cross-chain context factory (batteries-included entry point); tag-based multi-bridge route selection
- [[wiki/xc-sdk\|xc-sdk]] — Wallet interface for multi-platform transfers (Substrate, EVM, Solana, Sui)
- [[wiki/xc-swap\|xc-swap]] — NEAR Intent Routing cross-chain swaps: any Hydration asset → NEAR-ecosystem asset in one EVM tx via `IntentEmitter.swapAndBridge` + 1Click API; sibling of [[wiki/xc-package\|xc-package]], bypasses the transfer stack
- [[wiki/xc-cfg\|xc-cfg]] — Pre-built route configs, DEX integrations, bridge builders (NTT, Snowbridge V1 + V2, Basejump); transfer-validation framework with Hydration circuit-breaker deposit/withdraw limits
- [[wiki/xc-core\|xc-core]] — Core types, chain & asset definitions, bridge primitives (`Tag.Ntt` / `Tag.NttExecutor` / `Tag.SnowbridgeV1`, `*_wh` keys); Hydration is Wormhole chain id 73
- [[wiki/xc-scan\|xc-scan]] — Cross-chain transaction scanning and journey tracking
- [[wiki/route-suggester\|route-suggester]] — Rust crate for high-performance DEX route discovery

### Runtime & Node (`hydration-node`)
- [[wiki/hydration-runtime\|hydration-runtime]] — `construct_runtime!`, pallet wiring, fee config, XCM config, spec_version 439, 42 pallets (new: `GigaHdx = 86`, `GigaHdxRewards = 87`, `FeeProcessor = 207`; trade-fee split rewired so 50% leaves the pool)
- [[wiki/hydration-precompiles\|hydration-precompiles]] — EVM precompiles: call-permit (gasless), flash-loan (HSM mint), lock-manager `0x…0806` (`getLockedBalance`, makes GIGAHDX non-transferable)
- [[wiki/pepl-worker\|pepl-worker]] — PEPL v2 node-side liquidation worker (`pepl-worker/`): multi-money-market, oracle fast path
- [[wiki/note-ai-skills\|note-ai-skills]] — Upstream `ai_skills/`: `hydration_cl0wdit` security-audit skill + `circuit-breaker-incident` triage skill, symlinked from `.claude/` and `.codex/`

### Pallets — AMM core
- [[wiki/pallet-omnipool\|pallet-omnipool]] — Single-pool AMM with LRNA hub asset
- [[wiki/pallet-stableswap\|pallet-stableswap]] — Curve-style pools (up to 5 assets); storage v2 virtual share issuance (`ShareIssuance`, `mint_shares`/`burn_shares`) — do NOT derive share supply from `total_issuance`
- [[wiki/pallet-xyk\|pallet-xyk]] — Constant-product two-asset pools
- [[wiki/pallet-lbp\|pallet-lbp]] — Liquidity bootstrapping pool
- [[wiki/pallet-route-executor\|pallet-route-executor]] — Multi-hop trade router

### Pallets — Risk & infrastructure
- [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] — Per-block trade/liquidity/egress volume limits; `Config::InTradeContext` covers the whole router window
- [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]] — Volume-based fee adjustment
- [[wiki/pallet-dynamic-evm-fee\|pallet-dynamic-evm-fee]] — EVM base-fee-per-gas oracle
- [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] — EMA price oracle store
- [[wiki/pallet-asset-registry\|pallet-asset-registry]] — Asset metadata & location registry
- [[wiki/pallet-xcm-rate-limiter\|pallet-xcm-rate-limiter]] — Defers XCM above per-asset rate threshold
- [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]] — Pay fees in any accepted asset
- [[wiki/pallet-transaction-pause\|pallet-transaction-pause]] — Pause arbitrary extrinsics

### Pallets — Finance & orchestration
- [[wiki/pallet-dca\|pallet-dca]] — Dollar-cost averaging schedules (buy schedules retired — `Error::NoLongerSupported`; existing ones still execute)
- [[wiki/pallet-staking\|pallet-staking]] — HDX staking + democracy points (legacy; superseded on the frontend by [[wiki/gigahdx\|gigahdx]])
- [[wiki/pallet-gigahdx\|pallet-gigahdx]] — Runtime index 86: `giga_stake` / `giga_unstake` / `migrate`, `ghdxlock` lock id, 28-day cooldown, exchange-rate yield, gigapot `PalletId(*b"gigahdx!")`
- [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] — Runtime index 87: referendum-linked reward accumulator, per-track Permill split pro rata by `staked_vote × conviction`, `claim_rewards` compounds
- [[wiki/pallet-fee-processor\|pallet-fee-processor]] — Runtime index 207: central trade-fee splitter (`process_trade_fee`), `on_idle` conversion of non-HDX slices to HDX, `HeldFees` ED buffering
- [[wiki/pallet-hsm\|pallet-hsm]] — Hollar Stability Mechanism
- [[wiki/pallet-otc\|pallet-otc]] — Peer-to-peer orders
- [[wiki/pallet-otc-settlements\|pallet-otc-settlements]] — OTC ↔ router arbitrage worker
- [[wiki/pallet-referrals\|pallet-referrals]] — Referral codes + tiered rewards
- [[wiki/pallet-bonds\|pallet-bonds]] — Fungible bonds with maturity
- [[wiki/pallet-liquidation\|pallet-liquidation]] — MM collateral liquidations via flash loan; adds unsigned `liquidate_with_pool` and protocol-funded `liquidate_gigahdx`
- [[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]] — Core yield-farming engine
- [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]] — LM wrapper for Omnipool NFT positions
- [[wiki/pallet-xyk-liquidity-mining\|pallet-xyk-liquidity-mining]] — LM wrapper for XYK LP shares

### Pallets — Currency & accounts
- [[wiki/pallet-currencies\|pallet-currencies]] — Multi-currency adapter
- [[wiki/pallet-evm-accounts\|pallet-evm-accounts]] — EVM ↔ Substrate account mapping; NTT minter registry (`NttMinters`, `set_ntt_minter`, `NttEmergencyOrigin`)
- [[wiki/pallet-duster\|pallet-duster]] — Dust removal + whitelist
- [[wiki/pallet-collator-rewards\|pallet-collator-rewards]] — Collator payout
- [[wiki/pallet-collator-rotation\|pallet-collator-rotation]] — SessionManager wrapper that benches one collator each odd session (runtime index 58)
- [[wiki/pallet-democracy\|pallet-democracy]] — Legacy democracy (pre-OpenGov bridge)
- [[wiki/pallet-claims\|pallet-claims]] — HDX claims via ETH signatures
- [[wiki/pallet-dispenser\|pallet-dispenser]] — One-time faucet for ETH accounts
- [[wiki/pallet-nft\|pallet-nft]] — NFT pallet (LP positions, staking)

### Pallets — Infrastructure / misc
- [[wiki/pallet-broadcast\|pallet-broadcast]] — Shared Swapped event bus
- [[wiki/pallet-dispatcher\|pallet-dispatcher]] — Proxy-style dispatch with custom origins
- [[wiki/pallet-genesis-history\|pallet-genesis-history]] — Chain-migration lineage marker
- [[wiki/pallet-parameters\|pallet-parameters]] — Genesis-set runtime flags (`IsTestnet`, `RelayParentOffsetOverride`)
- [[wiki/pallet-relaychain-info\|pallet-relaychain-info]] — Relay metadata exposure
- [[wiki/pallet-signet\|pallet-signet]] — Authenticated off-chain data signing

### Frontend (`hydration-ui`)
- [[wiki/hydration-ui\|hydration-ui]] — Top-level entry: Yarn + Turbo monorepo, `apps/main` + 7 packages
- [[wiki/hydration-ui-main-app\|hydration-ui-main-app]] — `apps/main` — App.tsx, routing, providers, Zustand stores, modules, workers
- [[wiki/hydration-ui-api\|hydration-ui-api]] — `apps/main/src/api` — React Query domain hooks (30+ domains, incl. `multisig`, `gigaStake`, `gigaApr`, `grafana`, NTT + cross-chain circuit-breaker limits)
- [[wiki/hydration-ui-modules\|hydration-ui-modules]] — 13 product modules: trade (incl. `TradeOrdersHistory`), liquidity, borrow, xcm, wallet, staking (`/staking` now [[wiki/gigahdx\|gigahdx]]; legacy at `/staking-old`), strategies (BIL RWA vault, Hollar bonds), onramp (CEX + bank wizard), governance (stub), stats, transactions, submit-transaction, layout
- [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] — Transaction signing / review / status flow (incl. `ReviewMultisig`, `ReviewMultiTransaction`, JSON-view inspector)
- [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] — `packages/web3-connect` wallet abstraction (PJS, WalletConnect, EVM permit, Solana, Near, Zcash) + persisted address book / contacts
- [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] — `packages/ui` — Radix + Emotion + style-dictionary tokens, Storybook 10.3; `BannerTop`, `TradingViewChart`, `CBreaker*` limit components
- [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] — `packages/money-market` — Aave v3 UI hooks for [[wiki/hydration-borrow\|hydration-borrow]]; three markets `CustomMarket.{hydration_v3, gigahdx_v3, bil_v3}`, `MoneyMarketProvider` now takes `market` not `env` (breaking)
- [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] — `packages/indexer` — three GraphQL clients: indexer, squid, multix (snowbridge client deleted); farms queries; charts migrating squid → Grafana SQL (`api/grafana/`)
- [[wiki/hydration-ui-utils\|hydration-ui-utils]] — `packages/utils` — shared helpers including `basejumpscan`
- [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] — Cross-cutting: React 19, Tanstack Router/Query, Emotion, viem, papi `^2.1.7`, Vite 8 (Rolldown) + MDX, zod 4, no wagmi; full integration picture

### Ecosystem
- [[wiki/polkadot\|polkadot]] — Relay chain providing shared security
- [[wiki/asset-hub\|asset-hub]] — Polkadot system chain for asset issuance

### Auditors & Security
- [[wiki/runtime-verification\|runtime-verification]] — Rust implementation auditor (Sept 2022)
- [[wiki/blockscience\|blockscience]] — Economic/math model auditor (March 2022)
- [[wiki/code4rena\|code4rena]] — Competitive audit of full node (Feb 2024)
- [[wiki/immunefi\|immunefi]] — Bug bounty platform

## Concepts

### Omnipool Mechanics
- [[wiki/impermanent-loss\|impermanent-loss]] — IL formula, Omnipool vs. XYK comparison, worked examples
- [[wiki/nft-lp-positions\|nft-lp-positions]] — NFT-based LP accounting (shares, entry price, partial withdrawals)
- [[wiki/dynamic-fees\|dynamic-fees]] — Volatility-adjusted trading and withdrawal fees
- [[wiki/tradability-flags\|tradability-flags]] — Per-asset operation permissions (SELL/BUY/ADD/REMOVE)

### Risk Controls
- [[wiki/circuit-breaker\|circuit-breaker]] — Per-block trade volume limits (50% max)
- [[wiki/price-barrier\|price-barrier]] — Spot/oracle price deviation check (1% threshold)
- [[wiki/ema-oracle\|ema-oracle]] — Exponential moving average oracle for time-weighted pricing
- [[wiki/protocol-owned-liquidity\|protocol-owned-liquidity]] — POL as liquidity floor and IL mitigation

### Governance & Tokenomics
- [[wiki/opengov\|opengov]] — On-chain governance framework (referenda, token listings, parameter changes); referendum completion drives [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] payouts
- [[wiki/bonding-curve\|bonding-curve]] — HDX staking reward distribution model (legacy [[wiki/pallet-staking\|pallet-staking]] path)
- [[wiki/gigahdx\|gigahdx]] — Liquid staking as the current HDX governance-and-yield primitive (conviction-weighted rewards, 28-day cooldown)
- [[wiki/referrals\|referrals]] — Fee-sharing referral system (5% slice, now routed by [[wiki/pallet-fee-processor\|pallet-fee-processor]])

### Trading Tools & Routing
- [[wiki/smart-order-router\|smart-order-router]] — BFS-based multi-hop trade routing across all pool types
- [[wiki/dca\|dca]] — Dollar-Cost Averaging / Split Trade automation
- [[wiki/otc-trading\|otc-trading]] — Peer-to-peer OTC trades with Omnipool price alignment
- [[wiki/lbp\|lbp]] — Liquidity Bootstrapping Pools for token launches
- [[wiki/ice\|ice]] — Intent Composing Engine (upcoming intent-based trading)
- [[wiki/stableswap\|stableswap]] — Stablecoin-optimized AMM pools
- [[wiki/xyk-pools\|xyk-pools]] — Classic constant-product pools (permissionless listing)

### Cross-Chain
- [[wiki/xcm\|xcm]] — Polkadot's native cross-chain messaging
- [[wiki/snowbridge\|snowbridge]] — Trustless Polkadot↔Ethereum bridge; V1 and V2 ship side by side (`Tag.SnowbridgeV1` = legacy cheaper route)
- [[wiki/wormhole\|wormhole]] — NTT bridge: direct Hydration ↔ Ethereum / Base / Solana / Sui, Hydration is Wormhole chain id 73 (MRL retired, Moonbeam removed)
- [[wiki/xc-swap\|xc-swap]] — NEAR Intent Routing swaps (IntentEmitter + 1Click), outside the XCM/bridge transfer stack
- [[wiki/pallet-frontier\|pallet-frontier]] — EVM compatibility layer (MetaMask, Solidity tooling)

## Runbooks

- [[wiki/runbook-run-collator\|runbook-run-collator]] — Run a Hydration collator: machine specs, runtime gate (`MaxCandidates = 0`, Invulnerables via `GeneralAdmin`), session keys, rewards (455 HDX/session), onboarding sequence
- [[wiki/runbook-request-judgement\|runbook-request-judgement]] — Request an identity judgement: `pallet_identity` `setIdentity` → `requestJudgement(reg_index, max_fee)` flow, 2 mainnet registrars (HydraSik=0, stakenode=1), NOT a Technical Committee action, registrars managed by Root/GeneralAdmin

## Analyses

*No analyses yet. Ask a question and we can file the answer here if it's worth keeping.*
