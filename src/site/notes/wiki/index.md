---
{"dg-publish":true,"permalink":"/wiki/index/","title":"Wiki Index","dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"index","title":"Wiki Index","last_updated":"2026-11-02"}}
---


# Index

Master catalog of all pages in the wiki. Updated on every ingest.

## Start here

- [[wiki/overview\|overview]] — 30-second briefing: what's indexed, protocol pillars, per-repo summary
- [[wiki/routing\|routing]] — task → pages cheat sheet ("I need to X → read Y, Z")
- [[wiki/hydration\|hydration]] — the protocol itself, if you need the ground truth before routing

## Sources

- [[wiki/source-onchain-monitoring-landscape\|source-onchain-monitoring-landscape]] — Nov 2026 vendor research (Blockaid, Hypernative, Hexagate, Phalcon, Forta, Venn, Tenderly, OZ Monitor…): pipeline, intervention modes, Hydration relevance
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
- [[wiki/sdk-next\|sdk-next]] — v2.5.0 trade router (Omni / Stableswap / XYK / LBP / HSM / Aave ×3 markets / [[wiki/uniswap-v3\|uniswap-v3]]) on an event-driven `PoolSync` + `EventBus` engine; SOR, DCA (TWAP buy removed, `getOpenBudgetDcaOrder`), `MmRouting`, multicall EVM reads, 2 s `ChainParams`; `'finalized'` = static read
- [[wiki/sdk-common\|sdk-common]] — Shared utilities (big numbers, XCM, EVM, substrate RPC via `createWsClient`)
- [[wiki/sdk-descriptors\|sdk-descriptors]] — Hydration papi descriptors v2.9.0; single whitelist `.papi/whitelist.ts`; ICE, Intent, GigaHdx(Rewards), Parameters whitelisted — FeeProcessor + LazyExecutor still NOT; `Democracy` still whitelisted though removed from runtime
- [[wiki/xc-package\|xc-package]] — Cross-chain context factory (batteries-included entry point); tag-based multi-bridge route selection
- [[wiki/xc-sdk\|xc-sdk]] — v2.7.0 wallet interface for multi-platform transfers (Substrate, EVM, Solana, Sui, NEAR, Zcash); multichain balances, `validate(fee, amount)`
- [[wiki/xc-swap\|xc-swap]] — v0.9.0 any Hydration asset → NEAR-Intents asset in one EVM tx: `IntentEmitter.placeOrder` → WETH → NTT → Ethereum `IntentReceiver` → 1Click; drives the UI Market tab; on-chain half in [[wiki/whm-intents\|whm-intents]]
- [[wiki/xc-cfg\|xc-cfg]] — v2.9.1 route configs + bridge builders (NTT registry, Snowbridge V1+V2, Basejump, Omni); chains incl. HyperEVM, Robinhood, NEAR, Zcash; validations (circuit-breaker limits, custody, Omni, EVM value)
- [[wiki/xc-core\|xc-core]] — v2.8.0 core types, chain & asset defs, bridge primitives (`Tag.Ntt` / `NttExecutor` / `SnowbridgeV1`); `NearChain`, `ZecChain`, `NearClient`, `UtxoEmitter` ABI; Hydration = Wormhole chain 73
- [[wiki/xc-scan\|xc-scan]] — Cross-chain transaction scanning and journey tracking
- [[wiki/route-suggester\|route-suggester]] — Rust crate for high-performance DEX route discovery
- [[wiki/uniswap-v3\|uniswap-v3]] — Uniswap v3 on Hydration EVM: factory from `Parameters.UniswapV3Factory`, curated HOLLAR pools (aDOT, atBTC, aPAXG, GETH, GSOL), client-side tick math, trusted in SOR

### Runtime & Node (`hydration-node`)
- [[wiki/hydration-runtime\|hydration-runtime]] — `construct_runtime!`, pallet wiring, fee + XCM config; spec_version 458, 44 pallets, **2 s blocks**; ICE pallets `LazyExecutor = 97`, `Intent = 98`, `Ice = 99`; Democracy removed (`RemoveDemocracy` migration); XCM suspension (`PausableRouter`)
- [[wiki/hydration-precompiles\|hydration-precompiles]] — EVM precompiles: call-permit (gasless), flash-loan (HSM mint), lock-manager `0x…0806`; `is_precompile` now covers CallPermit + FlashLoanReceiver
- [[wiki/pepl-worker\|pepl-worker]] — PEPL v2 node-side liquidation worker: multi-money-market, DIA oracle fast path, off-chain liquidation routing (`routing.rs`)
- [[wiki/note-ai-skills\|note-ai-skills]] — Upstream `ai_skills/`: `hydration_cl0wdit` security-audit skill + `circuit-breaker-incident` triage skill, symlinked from `.claude/` and `.codex/`

### Pallets — AMM core
- [[wiki/pallet-omnipool\|pallet-omnipool]] — Single-pool AMM with LRNA hub asset
- [[wiki/pallet-stableswap\|pallet-stableswap]] — Curve-style pools (up to 5 assets); storage v2 virtual share issuance (`ShareIssuance`, `mint_shares`/`burn_shares`) — do NOT derive share supply from `total_issuance`
- [[wiki/pallet-xyk\|pallet-xyk]] — Constant-product two-asset pools
- [[wiki/pallet-lbp\|pallet-lbp]] — Liquidity bootstrapping pool
- [[wiki/pallet-route-executor\|pallet-route-executor]] — Multi-hop trade router
- [[wiki/pallet-intent\|pallet-intent]] — ICE intent storage/lifecycle: swap / partial / match-first / DCA intents; funds reserved as `ICE_int#`; OCW expiry cleanup; DCA migration target (index 98)
- [[wiki/pallet-ice\|pallet-ice]] — Solver-batch settlement; 0.02% matched-volume fee; `SolverMode` V4 / Passthrough / Disabled kill switch; per-venue routing include/exclude; `IceSolverApi` / `LiquidityStateApi` (index 99)
- [[wiki/pallet-lazy-executor\|pallet-lazy-executor]] — FIFO queue of post-resolution EVM forwards (`ForwardAction`, 512-byte data, 2M gas) dispatched by OCW unsigned `dispatch_top` (index 97)

### Pallets — Risk & infrastructure
- [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] — Per-block trade/liquidity limits, XCM deposit fuse (asset lockdown, covers NTT mints), global withdraw lockdown + egress limits, per-XCM-message deposit/egress netting
- [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]] — Volume-based fee adjustment
- [[wiki/pallet-dynamic-evm-fee\|pallet-dynamic-evm-fee]] — EVM base-fee-per-gas oracle
- [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] — EMA price oracle store; v3 direction-independent smoothing, 2 s-block periods; external pushers de-authorisable only by Root / EconomicParameters
- [[wiki/pallet-asset-registry\|pallet-asset-registry]] — Asset metadata & location registry (`register_external` removed)
- [[wiki/pallet-xcm-rate-limiter\|pallet-xcm-rate-limiter]] — Defers XCM above per-asset rate threshold
- [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]] — Pay fees in an `AcceptedCurrencies` asset or HDX (swap-to-DOT fallback removed); runtime index 203
- [[wiki/pallet-transaction-pause\|pallet-transaction-pause]] — Pause arbitrary extrinsics

### Pallets — Finance & orchestration
- [[wiki/pallet-dca\|pallet-dca]] — Legacy DCA schedules being migrated into ICE intents (`MigrationEnabled`, `force_cancel_schedules`, `CancelReason`); buy schedules no longer creatable
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
- [[wiki/pallet-claims\|pallet-claims]] — HDX claims via ETH signatures
- [[wiki/pallet-dispenser\|pallet-dispenser]] — One-time faucet for ETH accounts
- [[wiki/pallet-nft\|pallet-nft]] — NFT pallet (LP positions, staking)

### Pallets — Infrastructure / misc
- [[wiki/pallet-broadcast\|pallet-broadcast]] — Shared Swapped event bus
- [[wiki/pallet-dispatcher\|pallet-dispatcher]] — Proxy-style dispatch with custom origins
- [[wiki/pallet-genesis-history\|pallet-genesis-history]] — Chain-migration lineage marker
- [[wiki/pallet-parameters\|pallet-parameters]] — On-chain runtime params: `TwoSecBlocksSince`, Aave gas limits, Uniswap v3 addresses, **`suspend_xcm`** (any single TC member, Operational; resume needs TC 2/3; master only — not live at spec 455)
- [[wiki/pallet-relaychain-info\|pallet-relaychain-info]] — Relay metadata exposure
- [[wiki/pallet-signet\|pallet-signet]] — Authenticated off-chain data signing

### Frontend (`hydration-ui`)
- [[wiki/hydration-ui\|hydration-ui]] — Top-level entry: Yarn + Turbo monorepo @770f868, `apps/main` + 7 packages, 14 modules; IPFS deploy
- [[wiki/hydration-ui-main-app\|hydration-ui-main-app]] — `apps/main` — App.tsx, routing, providers, Zustand stores, modules, workers
- [[wiki/hydration-ui-api\|hydration-ui-api]] — `apps/main/src/api` — React Query domains incl. `balances/`, `portfolio/`, `gamma/`, `intents`, `neckwork`, `scan`; `isReady` gate (`isApiLoaded` gone)
- [[wiki/hydration-ui-modules\|hydration-ui-modules]] — 14 product modules: trade (Swap, Market = [[wiki/xc-swap\|xc-swap]], Limit = ICE intents, orders), liquidity (incl. V3 vaults), borrow, xcm (scan-based history), portfolio (replaces wallet; `/wallet*` redirects), tutorials, staking ([[wiki/gigahdx\|gigahdx]]), strategies, onramp, stats (fees only), transactions, submit-transaction, layout; governance unrouted
- [[wiki/hydration-ui-submit-tx\|hydration-ui-submit-tx]] — Transaction signing / review / status flow: multisig, NEAR / Zcash signer paths, intent + XcSwap meta, `TransactionClosedError`
- [[wiki/hydration-ui-web3-connect\|hydration-ui-web3-connect]] — `packages/web3-connect` — Modal v2, `Web3ConnectSession`; PJS, WalletConnect, EVM permit, Solana, NEAR, Zcash
- [[wiki/hydration-ui-design-system\|hydration-ui-design-system]] — `packages/ui` — 77 Radix + Emotion components, `@tanstack/charts`, emoji/identicon avatars (`Combobox`, `SliderTabs` removed)
- [[wiki/hydration-ui-money-market\|hydration-ui-money-market]] — `packages/money-market` — Aave v3 UI hooks for [[wiki/hydration-borrow\|hydration-borrow]]; three markets (hydration, gigahdx, bil); provider takes a Neckwork client
- [[wiki/hydration-ui-indexer\|hydration-ui-indexer]] — `packages/indexer` — Neckwork REST (openapi-fetch) + `scan` + GraphQL indexer (farms only) + multix; squid retired
- [[wiki/hydration-ui-utils\|hydration-ui-utils]] — `packages/utils` — shared helpers: intentscan, transferscan, nearblocks, zcashexplorer, `ZecAddr`
- [[wiki/hydration-ui-tech-stack\|hydration-ui-tech-stack]] — React 19, Tanstack Router/Query, Emotion, viem, papi 2.2.2, sdk-next 2.5.0, Node 26, Vitest in CI, IPFS workflow

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
- [[wiki/note-emergency-levers\|note-emergency-levers]] — Every pause/limit lever (tradability, CB lockdown + global withdraw lockdown, tx-pause, emergency-admin dispatch, NTT minter revoke, MM/HSM) with scope, reversibility and fastest origin; MM + HSM have no TC fast path
- [[wiki/note-dependency-registry\|note-dependency-registry]] — Assets grouped by upstream dependency (Snowbridge ~19, Wormhole ~11, Asset Hub, Bifrost, Interlay, parachains) + oracles (DIA feeds, EMA via Chainlink-adapter precompile) + registry schema and chain-snapshot generator
- [[wiki/note-oracle-inventory\|note-oracle-inventory]] — Every oracle Hydration consumes (EMA internal sources, Chainlink-adapter precompile, DIA, WHM relay, Bifrost/external pushers, AaveOracle, stableswap peg sources) + 32 failure modes (staleness, manipulation, representation gap, key compromise, liquidation coupling) + chain reads to resolve
- [[wiki/note-collateral-map\|note-collateral-map]] — Every money-market collateral (main 24 reserves, GIGAHDX, BIL) with route, configured price source, LTV/LT, HOLLAR facilitators + pool pricing, tBTC dependency chain worked hop by hop, "weird movement" signals
- [[wiki/note-risk-register\|note-risk-register]] — Protocol-wide risk register (bridges, issuers, oracles, money market, HOLLAR/HSM, AMMs, ICE, governance/keys, EVM, infra, social) — each risk with today's control, gap and the signal to monitor
- [[wiki/note-live-chain-check\|note-live-chain-check]] — Live read 2026-11-02 (block 16.54M, spec 455): 32 main-market reserves, live oracle sources + owners + staleness, ACL roles (TC has none on main/BIL), deposit fuses, HOLLAR facilitators, pegs; `suspend_xcm` not live
- [[wiki/note-monitoring-plan\|note-monitoring-plan]] — What to watch so we don't get caught in someone else's incident: 6 layers (upstream, arrival, pricing, exposure, control plane, response readiness), ~40 signals with severity, alert format, routing, what fires today, rollout order
- [[wiki/note-security-hardening-plan\|note-security-hardening-plan]] — Draft 4-phase plan (verify → fast paths + paging → watchtower → runtime/bridge upgrades); gaps: NTT hubs pauser = 4/7 owner, single verifier (threshold 1), fuse auto-expiry + funds-unlocker, no custody/admin/governance watch; patterns from Wormhole Governor/Accountant, CCIP, Snowbridge RFC-0166/0167, GateSeal, Aave Guardian
- [[wiki/note-dependency-aware-monitoring\|note-dependency-aware-monitoring]] — Design draft: registry + internal/external watchers + correlator + pre-encoded playbooks; build order and open decisions

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
- [[wiki/ice\|ice]] — Intent Composing Engine — **live**: [[wiki/pallet-intent\|pallet-intent]] stores intents, solvers settle batches via [[wiki/pallet-ice\|pallet-ice]], [[wiki/pallet-lazy-executor\|pallet-lazy-executor]] runs post-settlement EVM forwards; UI Limit tab
- [[wiki/stableswap\|stableswap]] — Stablecoin-optimized AMM pools
- [[wiki/xyk-pools\|xyk-pools]] — Classic constant-product pools (permissionless listing)

### Cross-Chain
- [[wiki/xcm\|xcm]] — Polkadot's native cross-chain messaging
- [[wiki/snowbridge\|snowbridge]] — Trustless Polkadot↔Ethereum bridge; V1 + V2 side by side (`Tag.SnowbridgeV1` legacy); UI drives it via XC with env kill switches
- [[wiki/wormhole\|wormhole]] — Hydration's Wormhole core + NTT routes (Ethereum, Base, Solana, Sui, HyperEVM, Robinhood, NEAR, ZEC; HDX/HOLLAR lock out to Robinhood); Hydration = chain 73; WHM messaging stack: [[wiki/source-whm-codebase\|source-whm-codebase]]
- [[wiki/xc-swap\|xc-swap]] — v0.9.0 any Hydration asset → NEAR-Intents asset in one EVM tx: `IntentEmitter.placeOrder` → WETH → NTT → Ethereum `IntentReceiver` → 1Click; drives the UI Market tab; on-chain half in [[wiki/whm-intents\|whm-intents]]
- [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]] — HyperEVM (wormhole 47 / EVM 999) is wired: HYPE via our own locking manager `0xdcDd08BF…` (not Wormhole's canonical group); feUSD still open; Ethereum NTT Safe signers in [[wiki/whm-cross-chain-governance\|whm-cross-chain-governance]]
- [[wiki/pallet-frontier\|pallet-frontier]] — EVM compatibility layer (MetaMask, Solidity tooling)

### WHM (`galacticcouncil/whm`) — Wormhole messaging
- [[wiki/source-whm-codebase\|source-whm-codebase]] — WHM monorepo: Foundry UUPS contracts, Solana oracle-emitter, NEAR ntt-manager / utxo-withdraw, 5 agents, prod + archive deployments; Hydration ids 73 / 222222 / 2034
- [[wiki/whm-oracle-relay\|whm-oracle-relay]] — Kamino Scope (PRIME, SOL), Jito stake-pool rate, wstETH `stEthPerToken`, apyUSD `convertToAssets`, Chainlink SPY → renounced `OracleReceiver` → `setPrice` on Hydration oracles; HDX EMA → Robinhood; `maxPriceAge` 300 s
- [[wiki/whm-basejump\|whm-basejump]] — Direct instant Ethereum→Hydration USDC (asset 21) route: payout from pre-funded `BasejumpLanding` `0x70e9…f976`, NTT settles behind; 0.1 USDC fee; live since ref #404
- [[wiki/whm-intents\|whm-intents]] — `IntentEmitter.placeOrder`: sell → WETH → NTT → Ethereum `IntentReceiver` → NEAR 1Click; relayer allowlist; v2 (ICE-driven) draft
- [[wiki/whm-cross-chain-governance\|whm-cross-chain-governance]] — OpenGov Root → `0xaa7e…aa7e2` → `GovernanceDispatcher` → `GovernanceExecutor` on Ethereum / Robinhood / Base / HyperEVM; 24 h delay; TC Safe veto / fast-track / 30-day recovery
- [[wiki/whm-near-corridors\|whm-near-corridors]] — Custom NEAR NTT (NEAR, ZEC; locking, Sputnik DAO owner); ZEC UTXO deposit/withdraw via Omni + NEAR MPC; `ZecUtxoEmitter`
- [[wiki/whm-agents\|whm-agents]] — broadcaster / relayer / scan / intent / utxo agents (no protocol authority); Hydration-side NTT managers + transceivers table
- [[wiki/whm-contracts\|whm-contracts]] — Prod address book per chain with owners (= upgrade authority); archived deployments marked dead

## Runbooks

- [[wiki/runbook-run-collator\|runbook-run-collator]] — Run a Hydration collator: machine specs, runtime gate (`MaxCandidates = 0`, Invulnerables via `GeneralAdmin`), session keys, rewards (455 HDX/session), onboarding sequence
- [[wiki/runbook-request-judgement\|runbook-request-judgement]] — Request an identity judgement: `pallet_identity` `setIdentity` → `requestJudgement(reg_index, max_fee)` flow, 2 mainnet registrars (HydraSik=0, stakenode=1), NOT a Technical Committee action, registrars managed by Root/GeneralAdmin

- [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]] — Pre-decided pauses per failure class: §A bridge compromise / unbacked mint, §B oracle failure, §C stable depeg, §D LST depeg, §E origin-chain halt; severity ladder S0–S3, per-asset card template, drills

## Analyses

*No analyses yet. Ask a question and we can file the answer here if it's worth keeping.*
