---
{"dg-publish":true,"permalink":"/wiki/source-hydration-node-codebase/","title":"hydration-node codebase","tags":["hydration","runtime","substrate","polkadot","rust"],"dg-note-properties":{"type":"source","title":"hydration-node codebase","source_kind":"repo_clone","raw_path":"raw/hydration-node/","upstream":"https://github.com/galacticcouncil/hydration-node","cloned_at":"2026-04-13","last_refreshed":"2026-04-20","last_commit":"ae3c944 add scripts/upgrade-runtime for governance-based runtime upgrades on forks (2026-04-18)","produces_pages":["pallet-omnipool","pallet-stableswap","pallet-xyk","pallet-lbp","pallet-route-executor","pallet-circuit-breaker","pallet-dynamic-fees","pallet-dynamic-evm-fee","pallet-ema-oracle","pallet-asset-registry","pallet-xcm-rate-limiter","pallet-transaction-multi-payment","pallet-transaction-pause","pallet-dca","pallet-staking","pallet-hsm","pallet-otc","pallet-otc-settlements","pallet-referrals","pallet-bonds","pallet-liquidation","pallet-liquidity-mining","pallet-omnipool-liquidity-mining","pallet-xyk-liquidity-mining","pallet-currencies","pallet-evm-accounts","pallet-duster","pallet-collator-rewards","pallet-democracy","pallet-claims","pallet-dispenser","pallet-nft","pallet-broadcast","pallet-dispatcher","pallet-genesis-history","pallet-parameters","pallet-relaychain-info","pallet-signet","hydration-runtime","hydration-precompiles"],"tags":["hydration","runtime","substrate","polkadot","rust"],"last_updated":"2026-04-20"}}
---


# hydration-node codebase

**TL;DR:** Substrate parachain powering [[wiki/hydration\|hydration]]. 38 pallets under `pallets/`, runtime at `runtime/hydradx/` (spec_version 411), Polkadot SDK fork `polkadot-stable2503-11-patch2`, toolchain Rust 1.84.1 → wasm32.

## Upstream
- Repo: https://github.com/galacticcouncil/hydration-node
- Branch cloned: default (main)
- Local path: `raw/hydration-node/`
- Full file count: 1117 tracked source files

## Top-level layout

```
raw/hydration-node/
├── pallets/                  # 38 custom pallets
├── runtime/
│   ├── hydradx/              # Main runtime (construct_runtime!)
│   └── adapters/             # Runtime adapter crates
├── node/                     # Collator binary (CLI, RPC, service)
├── integration-tests/        # Full-runtime integration tests
├── precompiles/              # EVM precompiles (call-permit, flash-loan)
├── math/                     # hydra-dx-math library
├── traits/                   # hydradx-traits shared trait definitions
├── primitives/               # Core types
├── runtime-mock/             # Shared test mocking utilities
├── liquidation-worker-support/  # Offchain liquidator helpers
├── scraper/                  # Offchain workers for pool scraping
├── launch-configs/           # Zombienet, Chopsticks, fork configs
├── scripts/                  # Benchmarking and deployment scripts
├── CLAUDE.md                 # Upstream project instructions
├── Cargo.toml                # Workspace manifest
├── Makefile                  # build / test / clippy / format targets
├── rustfmt.toml              # Tabs, line width 120
└── rust-toolchain.toml       # Rust 1.84.1
```

## Build & test (from `raw/hydration-node/CLAUDE.md`)

```sh
make build             # release build
make test              # cargo test --locked
make test-release      # cargo test --release --locked
make clippy            # clippy with -D warnings
make format            # cargo fmt
make build-benchmarks  # build with runtime-benchmarks feature
make test-benchmarks   # test with runtime-benchmarks feature
```

Single pallet test: `cargo test -p pallet-omnipool --locked`
All cargo commands use `--config net.git-fetch-with-cli=true`.

## Pallet inventory (38)

### AMM core (5)
- [[wiki/pallet-omnipool\|pallet-omnipool]] — single-pool AMM with hub asset LRNA
- [[wiki/pallet-stableswap\|pallet-stableswap]] — Curve-style pools (up to 5 assets)
- [[wiki/pallet-xyk\|pallet-xyk]] — constant-product two-asset pools
- [[wiki/pallet-lbp\|pallet-lbp]] — liquidity bootstrapping pool
- [[wiki/pallet-route-executor\|pallet-route-executor]] — multi-hop trade router

### Risk & infrastructure (8)
- [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] — trade/liquidity/egress limits
- [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]] — volume-based fee adjustment
- [[wiki/pallet-dynamic-evm-fee\|pallet-dynamic-evm-fee]] — EVM base-fee-per-gas oracle
- [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] — EMA price oracle store
- [[wiki/pallet-asset-registry\|pallet-asset-registry]] — asset metadata & location registry
- [[wiki/pallet-xcm-rate-limiter\|pallet-xcm-rate-limiter]] — defers XCM above per-asset rate
- [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]] — pay fees in any accepted asset
- [[wiki/pallet-transaction-pause\|pallet-transaction-pause]] — pause arbitrary extrinsics

### Finance & orchestration (11)
- [[wiki/pallet-dca\|pallet-dca]] — dollar-cost averaging schedules
- [[wiki/pallet-staking\|pallet-staking]] — HDX staking + democracy points
- [[wiki/pallet-hsm\|pallet-hsm]] — Hollar Stability Mechanism
- [[wiki/pallet-otc\|pallet-otc]] — peer-to-peer orders
- [[wiki/pallet-otc-settlements\|pallet-otc-settlements]] — OTC↔router arbitrage worker
- [[wiki/pallet-referrals\|pallet-referrals]] — referral codes + tiered rewards
- [[wiki/pallet-bonds\|pallet-bonds]] — fungible bonds with maturity
- [[wiki/pallet-liquidation\|pallet-liquidation]] — MM collateral liquidations (flash loan)
- [[wiki/pallet-liquidity-mining\|pallet-liquidity-mining]] — core yield-farming engine
- [[wiki/pallet-omnipool-liquidity-mining\|pallet-omnipool-liquidity-mining]] — LM wrapper for Omnipool NFTs
- [[wiki/pallet-xyk-liquidity-mining\|pallet-xyk-liquidity-mining]] — LM wrapper for XYK LP shares

### Currency & accounts (8)
- [[wiki/pallet-currencies\|pallet-currencies]] — multi-currency adapter
- [[wiki/pallet-evm-accounts\|pallet-evm-accounts]] — EVM↔Substrate account mapping
- [[wiki/pallet-duster\|pallet-duster]] — dust removal + whitelist
- [[wiki/pallet-collator-rewards\|pallet-collator-rewards]] — collator payout
- [[wiki/pallet-democracy\|pallet-democracy]] — legacy democracy (pre-OpenGov bridge)
- [[wiki/pallet-claims\|pallet-claims]] — HDX claims (ETH signatures)
- [[wiki/pallet-dispenser\|pallet-dispenser]] — one-time faucet (ETH accounts)
- [[wiki/pallet-nft\|pallet-nft]] — NFT pallet (LP positions, staking)

### Infrastructure / misc (6)
- [[wiki/pallet-broadcast\|pallet-broadcast]] — shared Swapped event bus
- [[wiki/pallet-dispatcher\|pallet-dispatcher]] — proxy-style dispatch with custom origins
- [[wiki/pallet-genesis-history\|pallet-genesis-history]] — chain-migration lineage marker
- [[wiki/pallet-parameters\|pallet-parameters]] — on-chain runtime parameters
- [[wiki/pallet-relaychain-info\|pallet-relaychain-info]] — relay metadata exposure
- [[wiki/pallet-signet\|pallet-signet]] — authenticated off-chain data signing

## Runtime

See [[wiki/hydration-runtime\|hydration-runtime]]. Main entry: `runtime/hydradx/src/lib.rs` (construct_runtime!, recursion_limit = 512, spec_version 411).

## Precompiles

See [[wiki/hydration-precompiles\|hydration-precompiles]]. Located under `precompiles/`. Include `call-permit` (gasless EVM permit) and `flash-loan` (HSM flash mint).

## Math / traits / primitives

- `math/` → hydra-dx-math (Omnipool, Stableswap, XYK, LBP, DCA math primitives)
- `traits/` → hydradx-traits (shared trait abstractions like `OmnipoolHooks`, `RouteProvider`, `PriceOracle`, `AMM`)
- `primitives/` → hydradx-primitives (BlockNumber=u32, AssetId=u32, Balance=u128, Amount=i128, Price=FixedU128, CollectionId=u128, ItemId=u128, EvmAddress=H160, Hash=H256)

## Code style rules (from upstream CLAUDE.md)

- Tabs for indentation (hard_tabs = true), max line width 120
- No `unwrap()` in production — require explicit proof `; qed` comments
- No unsafe unless specifically permitted
- Indent depth > 5 is a smell
- Conventional commits: `<type>(<scope>)<breaking>: <subject>` (types: feat, fix, refactor, perf, test, docs, style, ci, build)
- Bump crate SemVer on changes; bump runtime `spec_version` for breaking changes

## CI

1. `cargo fmt --check`
2. `clippy --release --all-targets` (warnings = errors)
3. `test --release`
4. Benchmark build check
5. Semantic PR title validation
6. Version bump validation

## Dependencies

- Polkadot SDK fork: `galacticcouncil/polkadot-sdk` branch `polkadot-stable2503-11-patch2`
- ORML fork: `bifrost-io/open-runtime-module-library` branch `polkadot-stable2503`
- Codec: `parity-scale-codec 3.7`
- Workspace-level dep management in root `Cargo.toml`

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/hydration\|hydration]]
