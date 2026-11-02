---
{"dg-publish":true,"permalink":"/wiki/source-hydration-node-codebase/","title":"hydration-node codebase","tags":["hydration","runtime","substrate","polkadot","rust"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"source","title":"hydration-node codebase","source_kind":"repo_clone","raw_path":"raw/hydration-node/","upstream":"https://github.com/galacticcouncil/hydration-node","cloned_at":"2026-04-13","last_refreshed":"2026-11-02","last_commit":"a35840f (2026-10-28) — spec_version 458; 2 s blocks; ICE intent stack live (pallet-lazy-executor=97, pallet-intent=98, pallet-ice=99, node ice_solver_worker, ice/ solver crates); Uniswap v3 router venue; XCM suspension in pallet-parameters; gov v1 (pallet-democracy) removed; fee-payment swap fallback removed; DCA→ICE migration path; direction-independent EMA smoothing; PEPL off-chain liquidation routing","produces_pages":["pallet-omnipool","pallet-stableswap","pallet-xyk","pallet-lbp","pallet-route-executor","pallet-circuit-breaker","pallet-dynamic-fees","pallet-dynamic-evm-fee","pallet-ema-oracle","pallet-asset-registry","pallet-xcm-rate-limiter","pallet-transaction-multi-payment","pallet-transaction-pause","pallet-dca","pallet-staking","pallet-hsm","pallet-otc","pallet-otc-settlements","pallet-referrals","pallet-fee-processor","pallet-gigahdx","pallet-gigahdx-rewards","pallet-bonds","pallet-liquidation","pallet-liquidity-mining","pallet-omnipool-liquidity-mining","pallet-xyk-liquidity-mining","pallet-currencies","pallet-evm-accounts","pallet-duster","pallet-collator-rewards","pallet-collator-rotation","pallet-claims","pallet-dispenser","pallet-nft","pallet-broadcast","pallet-dispatcher","pallet-genesis-history","pallet-parameters","pallet-relaychain-info","pallet-signet","pallet-ice","pallet-intent","pallet-lazy-executor","ice","hydration-runtime","hydration-precompiles","gigahdx","pepl-worker","note-ai-skills"],"tags":["hydration","runtime","substrate","polkadot","rust"],"last_updated":"2026-11-02"}}
---


# hydration-node codebase

**TL;DR:** Substrate parachain powering [[wiki/hydration\|hydration]]. 44 pallet crates under `pallets/`, runtime at `runtime/hydradx/` (spec_version 458, **2 s blocks**), Polkadot SDK fork `polkadot-stable2506-11-patch`, toolchain Rust 1.88.0 → wasm32. Major subsystems: Omnipool/Stableswap/XYK/LBP AMMs + router (now incl. a Uniswap v3 EVM venue), the [[wiki/ice\|ice]] intent stack ([[wiki/pallet-intent\|pallet-intent]], [[wiki/pallet-ice\|pallet-ice]], [[wiki/pallet-lazy-executor\|pallet-lazy-executor]], node-side solver), [[wiki/gigahdx\|gigahdx]] staking, [[wiki/pallet-fee-processor\|pallet-fee-processor]], [[wiki/pepl-worker\|pepl-worker]] liquidations, off-chain synthetic EVM logs. Gov v1 is removed.

## Upstream
- Repo: https://github.com/galacticcouncil/hydration-node
- Branch cloned: default (master)
- Local path: `raw/hydration-node/`
- Vault copy: 1426 files (958 `.rs`); integration-test chain snapshots excluded (see `raw/hydration-node/.vault-clone-info`)
- HEAD: `a35840f` (2026-10-28), "Merge pull request #1561 … chore/remove-migrations"
- Previous vault sync: `cc1bc97` (2026-08-15), spec 439

## Top-level layout

```
raw/hydration-node/
├── pallets/                  # 44 custom pallets
├── ice/                      # off-chain ICE crates: ice-solver, route-findr, solver-bench
├── runtime/
│   ├── hydradx/              # Main runtime (construct_runtime!)
│   └── adapters/             # Runtime adapter crates
├── node/                     # Collator binary (CLI, RPC, service, synthetic_logs)
├── pepl-worker/              # NEW — PEPL v2 liquidation worker (crate + support subcrate)
├── liquidation-worker-support/  # v1 liquidator helpers (still built; v1 is opt-in)
├── integration-tests/        # Full-runtime integration tests (src/ + chain-state snapshots)
├── precompiles/              # EVM precompiles (call-permit, flash-loan, lock-manager, utils)
├── math/                     # hydra-dx-math library
├── traits/                   # hydradx-traits shared trait definitions
├── primitives/               # Core types
├── runtime-mock/             # Shared test mocking utilities
├── scraper/                  # Offchain workers for pool scraping
├── ai_skills/                # NEW — repo-local agent skills (see [[wiki/note-ai-skills\|note-ai-skills]])
├── launch-configs/           # Zombienet, Chopsticks, fork configs
├── scripts/                  # Benchmarking, ops and TC-proposal scripts
├── infrastructure/           # Terraform + node bootstrap shell scripts
├── docs/                     # CONTRIBUTING / STYLE_GUIDE / CODE_OF_CONDUCT
├── CLAUDE.md                 # Upstream project instructions
├── AGENTS.md                 # NEW — Codex entrypoint, defers to CLAUDE.md
├── Cargo.toml                # Workspace manifest
├── Makefile                  # build / test / clippy / format / docker-amd64
├── Dockerfile{,.binary,.with-builder}   # .binary / .with-builder are new
├── rustfmt.toml              # Tabs, line width 120
└── rust-toolchain            # Rust 1.88.0 (note: no .toml extension)
```

New top-level entries since `7722ff4`: `ai_skills/`, `pepl-worker/`, `AGENTS.md`, `Dockerfile.binary`, `Dockerfile.with-builder`, `.codex/`. `.claude/skills/hydration_cl0wdit` and `.codex/skills/hydration_cl0wdit` are now **symlinks** into `ai_skills/`.

## Build & test (from `raw/hydration-node/CLAUDE.md`)

```sh
make build             # release build
make test              # cargo test --locked
make test-release      # cargo test --release --locked
make clippy            # clippy with -D warnings
make format            # cargo fmt
make build-benchmarks  # build with runtime-benchmarks feature
make test-benchmarks   # test with runtime-benchmarks feature
make docker-amd64      # NEW — package a pre-built linux/amd64 binary via Dockerfile.binary (no compile)
```

Single pallet test: `cargo test -p pallet-omnipool --locked`
All cargo commands use `--config net.git-fetch-with-cli=true`.

## Pallet inventory (44)

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

### ICE (3)
- [[wiki/pallet-intent\|pallet-intent]] — intent storage/lifecycle: swaps, partial, match-first, DCA intents (index 98)
- [[wiki/pallet-ice\|pallet-ice]] — solver-batch settlement, matched-volume fee, solver mode kill switch, venue routing (index 99)
- [[wiki/pallet-lazy-executor\|pallet-lazy-executor]] — FIFO queue of post-resolution EVM forwards, OCW-dispatched (index 97)

### Finance & orchestration (14)
- [[wiki/pallet-dca\|pallet-dca]] — legacy DCA schedules (sell-only); migration path into ICE DCA intents
- [[wiki/pallet-staking\|pallet-staking]] — legacy HDX staking + conviction-voting points (superseded by gigahdx)
- [[wiki/pallet-gigahdx\|pallet-gigahdx]] — stHDX / GIGAHDX staking against the AAVE-fork money market (index 86)
- [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] — per-OpenGov-track voting rewards funded from trade fees (index 87)
- [[wiki/pallet-fee-processor\|pallet-fee-processor]] — central trade-fee splitter + HDX conversion (index 207)
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
- [[wiki/pallet-collator-rotation\|pallet-collator-rotation]] — SessionManager wrapper that benches one collator each odd session
- [[wiki/pallet-claims\|pallet-claims]] — HDX claims (ETH signatures)
- [[wiki/pallet-dispenser\|pallet-dispenser]] — one-time faucet (ETH accounts)
- [[wiki/pallet-nft\|pallet-nft]] — NFT pallet (LP positions, staking)

### Infrastructure / misc (6)
- [[wiki/pallet-broadcast\|pallet-broadcast]] — shared Swapped event bus
- [[wiki/pallet-dispatcher\|pallet-dispatcher]] — proxy-style dispatch with custom origins
- [[wiki/pallet-genesis-history\|pallet-genesis-history]] — chain-migration lineage marker
- [[wiki/pallet-parameters\|pallet-parameters]] — runtime parameters + XCM kill switch (`suspend_xcm`), Uniswap v3 addresses, `TwoSecBlocksSince`
- [[wiki/pallet-relaychain-info\|pallet-relaychain-info]] — relay metadata exposure
- [[wiki/pallet-signet\|pallet-signet]] — authenticated off-chain data signing

## Runtime

See [[wiki/hydration-runtime\|hydration-runtime]]. Main entry: `runtime/hydradx/src/lib.rs` (construct_runtime!, recursion_limit = 512, spec_version 458). No Hyperbridge / ISMP / token-gateway and no gov v1 pallets are wired.

## Node (`node/`, crate `hydradx` v15.7.2)

| Module | Role |
|---|---|
| `node/src/service.rs` | `new_partial` / `start_node_impl`; spawns the liquidation and ICE solver workers, wires the synthetic storage override; Aura `authoring_duration` 2000 ms |
| `node/src/ice_solver_worker.rs` | ICE solver worker: per best block reads `IceSolverApi::solver_input`, solves natively, submits unsigned `ICE.submit_solution`. Flags `--ice-solver-worker`, `--ice-solver-split`, `--ice-solver-tolerance`; default on for validators — see [[wiki/pallet-ice\|pallet-ice]] |
| `node/src/cli.rs` | `Cli.liquidation_worker_config` is now `pepl_worker::LiquidationWorkerCli` (was the local v1 config struct) |
| `node/src/rpc.rs` | `FullDeps.pepl_status: Option<Arc<pepl_worker::WorkerStatus>>`; hosts the `liquidation_*` RPC module and swaps in a synth-aware `eth_getLogs` |
| `node/src/liquidation_worker.rs` | **v1** worker, retained behind `--liquidation-worker-v1`; `#![allow(dead_code)]`, plus a `From<&pepl_worker::LiquidationWorkerCli>` so old flags still boot it |
| `node/src/synthetic_logs/` | Off-chain indexing of substrate activity as EVM logs (synth tx hashes stable across reorgs) |

### `liquidation_*` RPCs (node/src/rpc.rs → `mod liquidation`)

`liquidation_getBorrowers` (→ `Vec<CoveredBorrower{pool, borrower}>`), `liquidation_isRunning`, `liquidation_maxTransactionsPerBlock`, `liquidation_lastScannedBlock`. All read [[wiki/pepl-worker\|pepl-worker]]'s `WorkerStatus`; they return empty/false when no v2 worker runs.

### `node/src/synthetic_logs/` (~2.6k LOC + SCALE test fixtures)

| File | Role |
|---|---|
| `metadata_events.rs` | decodes `System::Events` against the **chain's own runtime metadata**, by name not variant index, so a node built on a different polkadot-sdk still reads events; measures unrepresentable records instead of losing their neighbours |
| `compat_events.rs` | picks a reader per block; falls back to hand-written layouts + a salvage scan when metadata is unavailable |
| `storage_override.rs` | `SyntheticStorageOverride` wraps Frontier's `StorageOverride`, appending synthetic txs/statuses/receipts (block header stays canonical) |
| `eth_filter.rs` | replacement `eth_getLogs` that surfaces synth logs without corrupting canonical block hashes |
| `mapping_sync.rs` | vendored mapping-sync worker that also indexes synth tx hashes so `eth_getTransactionByHash` / `_receipt` resolve |
| `test_data/*.scale` | recorded mainnet event blobs + trimmed metadata_433 used by the decoder tests |

Translation primitives themselves live in the runtime crate (`runtime/hydradx/src/evm/synthetic_logs.rs`, `evm/event_logs.rs`) — see [[wiki/hydration-runtime\|hydration-runtime]].

Other node changes: `new_partial` treats the upstream default `runtime_cache_size == 2` as unset and uses `8` (historical-state RPC/indexer load thrashes a cache of 2); consensus now spawns on `spawn_essential_handle()`; mainnet bootnode ids refreshed in `node/res/hydradx.json` (helikon nodes dropped, shellfish added).

## Precompiles

See [[wiki/hydration-precompiles\|hydration-precompiles]]. Located under `precompiles/`: `call-permit` (gasless EVM permit), `flash-loan` (HSM flash mint), **`lock-manager`** (`pallet-evm-precompile-lock-manager`, address `0x…0806`, backs `LockableAToken.sol` for [[wiki/gigahdx\|gigahdx]]), and `utils`.

## Math / traits / primitives

- `math/` → hydra-dx-math (Omnipool, Stableswap, XYK, LBP, DCA math primitives)
- `traits/` → hydradx-traits (shared trait abstractions like `OmnipoolHooks`, `RouteProvider`, `PriceOracle`, `AMM`). Notable modules: `traits/src/fee_processor.rs` (`FeeReceiver`, `FeeDestination`, `Convert`), `traits/src/gigahdx.rs` (`MoneyMarketOperations`, `Seize`, `ClearConflictingVotes`), `traits/src/amm.rs` (`SimulatorConfig`, `RouteDiscovery`, simulator traits for ICE), `traits/src/lazy_executor.rs` (`ForwardAction`, `Source`, `Mutate`); `traits/src/router.rs` `PoolType` includes `UniswapV3(fee)`
- `primitives/` → hydradx-primitives (BlockNumber=u32, AssetId=u32, Balance=u128, Amount=i128, Price=FixedU128, CollectionId=u128, ItemId=u128, EvmAddress=H160, Hash=H256). New constant `GIGAHDX_SOURCE = *b"gigahdxs"` in `primitives/src/constants.rs`

## Off-chain workers

- [[wiki/pepl-worker\|pepl-worker]] (`pepl-worker/`, `pepl-worker/support/`) — PEPL v2 liquidation worker (default); off-chain collateral→debt routing via `LiquidityStateApi`
- ICE solver: `node/src/ice_solver_worker.rs` + `ice/ice-solver` (v4 + passthrough), `ice/route-findr`, `ice/solver-bench` (bench/fuzz/replay) — see [[wiki/pallet-ice\|pallet-ice]]
- `liquidation-worker-support/` (v1.2.0) — helper crate for the legacy v1 worker in `node/src/liquidation_worker.rs`, reachable via `--liquidation-worker-v1`
- `scraper/` — pool-state scraping for integration-test snapshots

## Integration tests

`integration-tests/src/` includes `fee_processor.rs`, `gigahdx.rs`, `gigahdx_rewards.rs`, `pepl.rs` (v1-vs-v2 decision parity + `decide_liquidation` scenarios), `ice/` (intent/solver end-to-end), `uniswap_v3_router.rs`, and an `integration-tests/README.md` describing the scraper-snapshot → replay-test debugging loop. Binary chain-state snapshots under `integration-tests/` (`dca-snapshot/`, `snapshots/`, `omnipool-snapshot/`, `evm-snapshot/`) are **excluded from `raw/`** — never cite those paths; `integration-tests/src/` is present.

## Agent skills in-repo

`ai_skills/` holds repo-local skills usable by Claude Code and Codex alike (`hydration_cl0wdit` security audit, `circuit-breaker-incident` lockdown triage). See [[wiki/note-ai-skills\|note-ai-skills]].

## Code style rules (from upstream CLAUDE.md)

- Tabs for indentation (hard_tabs = true), max line width 120
- No `unwrap()` in production — require explicit proof `; qed` comments
- No unsafe unless specifically permitted
- Indent depth > 5 is a smell
- Conventional commits: `<type>(<scope>)<breaking>: <subject>` (types: feat, fix, refactor, perf, test, docs, style, ci, build)
- Bump crate SemVer on changes; bump runtime `spec_version` for breaking changes
- **Comment policy (added 2026-08):** default to no comment. Never restate the code. Module headers = one paragraph. Skip obvious field/error docs. Extrinsic Description = 1–2 lines, no `Error` enumeration, no step lists. Keep only comments that warn about something a reader would otherwise miss (hidden invariants, why a defensive branch exists, why an unusual construct is correct)

## Operator runbooks (referenced from upstream CLAUDE.md)

- `scripts/mint-limit/README.md` — TC proposals to set XCM mint limits and lift circuit-breaker lockdowns
- `scripts/dca-monitor/README.md` — verify DCA fixes on a Chopsticks fork before a runtime upgrade
- `scripts/onchain-routes/README.md` — TC proposals to register/update on-chain router routes
- `integration-tests/README.md` — debug prod issues via scraper snapshots + integration tests

## CI

1. `cargo fmt --check`
2. `clippy --release --all-targets` (warnings = errors)
3. `test --release`
4. Benchmark build check
5. Semantic PR title validation
6. Version bump validation
7. `weight-diff` (new, `.github/workflows/weight-diff.yml`) — diffs generated pallet weights against `master` and comments on the PR (driven by `scripts/weight-diff/`)

## Dependencies

- Polkadot SDK fork: `galacticcouncil/polkadot-sdk` branch `polkadot-stable2506-11-patch` (was `polkadot-stable2503-11-patch2`)
- ORML fork: `galacticcouncil/open-runtime-module-library` branch `polkadot-stable2506` (was `bifrost-io/…` `polkadot-stable2503`)
- Toolchain: Rust `1.88.0` (was 1.84.1), file is `rust-toolchain` (no extension)
- Codec: `parity-scale-codec 3.7`
- New workspace deps: `frame-metadata 20.0` (node reads chain metadata to decode events), `alloy-primitives` / `alloy-sol-types` (runtime), `static_assertions` (runtime), `lru` / `rlp` / `futures-timer` (node)
- Workspace-level dep management in root `Cargo.toml`

## Sources

- [[wiki/source-hydration-general-context\|source-hydration-general-context]]
- [[wiki/source-omnipool-deep-context\|source-omnipool-deep-context]]
- [[wiki/hydration\|hydration]]
