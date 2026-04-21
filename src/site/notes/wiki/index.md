---
{"dg-publish":true,"permalink":"/wiki/index/","title":"Wiki Index","dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"index","title":"Wiki Index","last_updated":"2026-04-20"}}
---


# Index

Master catalog of all pages in the wiki. Updated on every ingest.

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]] — Foundational protocol overview: products, architecture, governance, tokenomics, cross-chain design
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]] — Deep technical reference: AMM model, swap math, LP mechanics, fees, risk controls, on-chain data model
- [[wiki/source-sdk-codebase\|source-sdk-codebase]] — Galactic SDK monorepo: trading SDK, cross-chain transfers, WASM math, tooling
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]] — Hydration parachain node: 38 pallets, runtime, precompiles, math, traits
- [[wiki/source-papi-docs\|source-papi-docs]] — papi.how documentation (Vocs site, 36 markdown pages)
- [[wiki/source-polkadot-api-codebase\|source-polkadot-api-codebase]] — polkadot-api monorepo: 24 packages (client, descriptors, cli, smoldot, signers, ink, substrate-bindings)
- [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]] — Hydration frontend monorepo: React 19 + Vite + Tanstack Router, apps/main + 7 workspace packages

## Entities

### Protocol & Products
- [[wiki/hydration\|hydration]] — The protocol itself: largest DeFi on Polkadot, vertically integrated trading + lending + stablecoin
- [[omnipool\|omnipool]] — Flagship single-pool AMM with hub token routing
- [[wiki/hollar\|hollar]] — Native over-collateralized stablecoin ($1 peg), built on GHO architecture
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

### SDK Packages (`galacticcouncil/sdk`)
- [[wiki/sdk-next\|sdk-next]] — Trade router, pool queries, smart order router, DCA/TWAP, tx building
- [[wiki/sdk-common\|sdk-common]] — Shared utilities (big numbers, XCM, EVM, substrate RPC)
- [[wiki/sdk-descriptors\|sdk-descriptors]] — Hydration papi type-safe metadata descriptors
- [[wiki/xc-package\|xc-package]] — Cross-chain context factory (batteries-included entry point); tag-based multi-bridge route selection
- [[wiki/xc-sdk\|xc-sdk]] — Wallet interface for multi-platform transfers (Substrate, EVM, Solana, Sui)
- [[wiki/xc-cfg\|xc-cfg]] — Pre-built route configs, DEX integrations, bridge builders (Basejump, Wormhole, Snowbridge)
- [[wiki/xc-core\|xc-core]] — Core types, chain & asset definitions, bridge primitives (basejump, snowbridge, wormhole)
- [[wiki/xc-scan\|xc-scan]] — Cross-chain transaction scanning and journey tracking
- [[wiki/route-suggester\|route-suggester]] — Rust crate for high-performance DEX route discovery

### Runtime & Node (`hydration-node`)
- [[wiki/hydration-runtime\|hydration-runtime]] — `construct_runtime!`, pallet wiring, fee config, XCM config, spec_version 411
- [[wiki/hydration-precompiles\|hydration-precompiles]] — EVM precompiles: call-permit (gasless), flash-loan (HSM mint)

### Pallets — AMM core
- [[wiki/pallet-omnipool\|pallet-omnipool]] — Single-pool AMM with LRNA hub asset
- [[wiki/pallet-stableswap\|pallet-stableswap]] — Curve-style pools (up to 5 assets)
- [[wiki/pallet-xyk\|pallet-xyk]] — Constant-product two-asset pools
- [[wiki/pallet-lbp\|pallet-lbp]] — Liquidity bootstrapping pool
- [[wiki/pallet-route-executor\|pallet-route-executor]] — Multi-hop trade router

### Pallets — Risk & infrastructure
- [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] — Per-block trade/liquidity/egress volume limits
- [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]] — Volume-based fee adjustment
- [[wiki/pallet-dynamic-evm-fee\|pallet-dynamic-evm-fee]] — EVM base-fee-per-gas oracle
- [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] — EMA price oracle store
- [[wiki/pallet-asset-registry\|pallet-asset-registry]] — Asset metadata & location registry
- [[wiki/pallet-xcm-rate-limiter\|pallet-xcm-rate-limiter]] — Defers XCM above per-asset rate threshold
- [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]] — Pay fees in any accepted asset
- [[wiki/pallet-transaction-pause\|pallet-transaction-pause]] — Pause arbitrary extrinsics

### Pallets — Finance & orchestration
- [[wiki/pallet-dca\|pallet-dca]] — Dollar-cost averaging schedules
- [[wiki/pallet-staking\|pallet-staking]] — HDX staking + democracy points
- [[wiki/pallet-hsm\|pallet-hsm]] — Hollar Stability Mechanism
- [[wiki/pallet-otc\|pallet-otc]] — Peer-to-peer orders
- [[wiki/pallet-otc-settlements\|pallet-otc-settlements]] — OTC ↔ router arbitrage worker
- [[wiki/pallet-referrals\|pallet-referrals]] — Referral codes + tiered rewards
- [[wiki/pallet-bonds\|pallet-bonds]] — Fungible bonds with maturity
- [[wiki/pallet-liquidation\|pallet-liquidation]] — MM collateral liquidations via flash loan
- [[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]] — Core yield-farming engine
- [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]] — LM wrapper for Omnipool NFT positions
- [[wiki/pallet-xyk-liquidity-mining\|pallet-xyk-liquidity-mining]] — LM wrapper for XYK LP shares

### Pallets — Currency & accounts
- [[wiki/pallet-currencies\|pallet-currencies]] — Multi-currency adapter
- [[wiki/pallet-evm-accounts\|pallet-evm-accounts]] — EVM ↔ Substrate account mapping
- [[wiki/pallet-duster\|pallet-duster]] — Dust removal + whitelist
- [[wiki/pallet-collator-rewards\|pallet-collator-rewards]] — Collator payout
- [[wiki/pallet-democracy\|pallet-democracy]] — Legacy democracy (pre-OpenGov bridge)
- [[wiki/pallet-claims\|pallet-claims]] — HDX claims via ETH signatures
- [[wiki/pallet-dispenser\|pallet-dispenser]] — One-time faucet for ETH accounts
- [[wiki/pallet-nft\|pallet-nft]] — NFT pallet (LP positions, staking)

### Pallets — Infrastructure / misc
- [[wiki/pallet-broadcast\|pallet-broadcast]] — Shared Swapped event bus
- [[wiki/pallet-dispatcher\|pallet-dispatcher]] — Proxy-style dispatch with custom origins
- [[wiki/pallet-genesis-history\|pallet-genesis-history]] — Chain-migration lineage marker
- [[wiki/pallet-parameters\|pallet-parameters]] — On-chain runtime parameters
- [[wiki/pallet-relaychain-info\|pallet-relaychain-info]] — Relay metadata exposure
- [[wiki/pallet-signet\|pallet-signet]] — Authenticated off-chain data signing

### Frontend (`hydration-ui`)
- [[wiki/hydration-ui\|hydration-ui]] — Top-level entry: Yarn + Turbo monorepo, `apps/main` + 7 packages
- [[wiki/hydration-ui-main-app\|hydration-ui-main-app]] — `apps/main` — App.tsx, routing, providers, Zustand stores, modules, workers
- [[wiki/hydration-ui-api\|hydration-ui-api]] — `apps/main/src/api` — React Query domain hooks (30+ domains, incl. new `multisig`)
- [[wiki/hydration-ui-modules\|hydration-ui-modules]] — Product modules: trade, liquidity, borrow, xcm, wallet, staking, stats, transactions, submit-transaction, layout
- [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] — Transaction signing / review / status flow (incl. `ReviewMultisig`)
- [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] — `packages/web3-connect` wallet abstraction (PJS, WalletConnect, EVM, Solana)
- [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] — `packages/ui` — Radix + Emotion + style-dictionary tokens, Storybook
- [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] — `packages/money-market` — Aave v3 UI hooks for [[wiki/hydration-borrow\|hydration-borrow]]
- [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] — `packages/indexer` — three GraphQL clients: indexer, squid, snowbridge
- [[wiki/hydration-ui-utils\|hydration-ui-utils]] — `packages/utils` — shared helpers
- [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] — Cross-cutting: React 19, Tanstack Router/Query, Emotion, viem, papi 1.23.3, full integration picture

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
- [[wiki/opengov\|opengov]] — On-chain governance framework (referenda, token listings, parameter changes)
- [[wiki/bonding-curve\|bonding-curve]] — HDX staking reward distribution model
- [[wiki/referrals\|referrals]] — Fee-sharing referral system

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
- [[wiki/snowbridge\|snowbridge]] — Trustless Polkadot↔Ethereum bridge
- [[wiki/wormhole\|wormhole]] — Multi-chain bridge for broader connectivity
- [[wiki/pallet-frontier\|pallet-frontier]] — EVM compatibility layer (MetaMask, Solidity tooling)

## Analyses

*No analyses yet. Ask a question and we can file the answer here if it's worth keeping.*
