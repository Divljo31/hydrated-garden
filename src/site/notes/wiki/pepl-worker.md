---
{"dg-publish":true,"permalink":"/wiki/pepl-worker/","title":"pepl-worker","tags":["liquidation","money-market","aave","offchain-worker","node","gigahdx","rust"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"pepl-worker","repo":"hydration-node","paths":["pepl-worker/Cargo.toml","pepl-worker/src/lib.rs","pepl-worker/support/Cargo.toml","pepl-worker/support/src/lib.rs","pepl-worker/support/src/types.rs","pepl-worker/support/src/math.rs","node/src/service.rs","node/src/rpc.rs","node/src/cli.rs","integration-tests/src/pepl.rs"],"key_exports":["LiquidationTask","LiquidationTaskConfig","LiquidationWorkerCli","LiquidationDecision","WorkerStatus","MmInstance","InstanceKind","InstanceSource","run","decide_liquidation","liquidation_call","dry_run_liquidation","apply_oracle_updates_and_decide","process_events_multi","borrower_cache","Hydration","MoneyMarket","Borrower","LiquidationOption"],"symbols":["LiquidationTask","LiquidationTaskConfig","LiquidationWorkerCli","LiquidationDecision","WorkerStatus","MmInstance","InstanceKind","InstanceSource","run","decide_liquidation","liquidation_call","dry_run_liquidation","DryRun","encode_liquidation_opaque","apply_oracle_updates_and_decide","ensure_instance_for_pool","find_instance_by_pool","find_instance_by_pap","MAX_SCAN_DRY_RUNS_PER_BLOCK","SUBMIT_COOLDOWN_BLOCKS","WORKER_LIVENESS_WINDOW","Hydration","MoneyMarket","Borrower","Reserve","ReserveData","UserConfiguration","EModeCategory","LiquidationOption","LiquidationAmounts","select_best_liquidation_option","RuntimeApiProvider","RuntimeClient","ray_mul","percent_mul","wad_div"],"depends_on":["pallet-liquidation","pallet-gigahdx","hydration-runtime"],"tags":["liquidation","money-market","aave","offchain-worker","node","gigahdx","rust"],"last_updated":"2026-08-15"}}
---


# pepl-worker

**TL;DR:** PEPL **v2** — the node-side liquidation worker for Hydration's Aave-fork money markets, shipped as two standalone workspace crates (`pepl-worker`, `pepl-worker-support`) instead of the v1 module embedded in `node/src/liquidation_worker.rs`. Multi-money-market aware (regular Aave pools **and** the [[wiki/gigahdx\|gigahdx]] market), submits unsigned `liquidate_with_pool` calls into the tx pool, and carries a same-block DIA-oracle fast path. Default since spec 439; v1 survives behind `--liquidation-worker-v1`.

> The repo never expands the acronym "PEPL" — it is the internal codename for this worker (branch `feat/pepl-v2`, PR #1501, the HEAD merge of the 2026-08-15 sync). Log target is `liquidation-worker`; default log prefix is `pepl-worker`.

## Crate layout

| Crate | Path | Role |
|---|---|---|
| `pepl-worker` v0.1.0 | `pepl-worker/` | the task: block/mempool streams, market registry, scan loop, submission |
| `pepl-worker-support` | `pepl-worker/support/` | pure Aave-model layer: EVM call encoding, `MoneyMarket` / `Borrower` types, RAY/WAD math |

`pepl-worker/support/src/tests/` and `pepl-worker/src/tests/` hold the unit tests; end-to-end and v1↔v2 parity tests live in `integration-tests/src/pepl.rs`.

## Relationship to v1

| | v1 (`node/src/liquidation_worker.rs` + `liquidation-worker-support/`) | v2 (`pepl-worker/`) |
|---|---|---|
| Markets | single, one PAP | registry of up to `MAX_MM_INSTANCES = 16`, incl. gigahdx |
| Discovery | omniwatch only | omniwatch **∪** on-chain `BORROW` logs **∪** on-disk cache **∪** chain bootstrap |
| Extrinsic | `liquidate` | `liquidate_with_pool` with `unsigned_priority` |
| Throughput control | worker-side block budget (`--weight-reserve`) | tx pool orders by `unsigned_priority`; scan capped only by `MAX_SCAN_DRY_RUNS_PER_BLOCK = 256` |
| Dedup | `liquidated_users` set + `tx_waitlist` | `submit_cooldown` map, `SUBMIT_COOLDOWN_BLOCKS = 4` |
| Restart | starts empty | borrower set persisted to `<base-path>/pepl/borrowers.json` |

v1 is still compiled (`#![allow(dead_code)]`) and reachable via `--liquidation-worker-v1`; `impl From<&pepl_worker::LiquidationWorkerCli> for LiquidationWorkerConfig` lets the v2 flags boot it. `--weight-reserve` and `--pap-contract` / `--oracle-update-signer` are accepted as v1 aliases — `--weight-reserve` logs a warning and is ignored by v2.

## Node wiring (node/src/service.rs)

```rust
// node/src/service.rs — start_node_impl
let cache_path = liquidation_worker_config.resolve_borrower_cache_path(&base_path_for_pepl);
let mut worker_cfg: pepl_worker::LiquidationTaskConfig = liquidation_worker_config.into();
worker_cfg.borrower_cache_path = cache_path;

let task = LiquidationTask::new(RuntimeClient::new(client.clone()), transaction_pool.clone(), worker_cfg);
pepl_status = Some(task.status.clone());
task_manager.spawn_handle().spawn("pepl-worker-runner", "pepl-worker",
    pepl_worker::run(task, client.clone()));
```

Enabled by default on validator nodes, off on non-validators; `--liquidation-worker <bool>` flips either way. `task.status: Arc<WorkerStatus>` is cloned into `rpc::FullDeps.pepl_status` and served by the `liquidation_*` RPCs (see [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]).

## CLI flags (`LiquidationWorkerCli`)

| Flag | Notes |
|---|---|
| `--liquidation-worker <bool>` | enable/disable |
| `--pool-address-provider <addr>` (alias `--pap-contract`) | repeatable; pins extra markets on top of auto-discovery |
| `--mm-pool-denylist <addr>` | pools that must never be instantiated |
| `--mm-discovery <bool>` | omniwatch instance discovery, default on |
| `--borrower-cache-path <path>` | default `<node-base-path>/pepl/borrowers.json`; `""` disables persistence |
| `--runtime-api-caller <addr>` | must hold WETH to pay for runtime-API EVM calls |
| `--oracle-signer <addr>…` (alias `--oracle-update-signer`) | DIA oracle update signers watched in the mempool |
| `--oracle-update-call-address <addr>…` | DIA oracle contract addresses |
| `--target-hf <u128>` | default `TARGET_HF = 1.001e18` |
| `--omniwatch-url <url>` | default `https://omniwatch.play.hydration.cloud/api/borrowers/by-health` |
| `--liquidations-per-block <u8>` | oracle-fast-path submissions per block, default 20; does **not** bound the scan |
| `--weight-reserve <u8>` | v1 only; ignored with a warning |
| `--liquidation-worker-v1` | run the legacy worker instead |

## Market registry

```rust
// pepl-worker/src/lib.rs
pub enum InstanceKind { Generic, GigaHdx }
pub enum InstanceSource { Chain, Config, Discovered }
```

- `Generic` → `pallet_liquidation::liquidate_with_pool` against the borrowing contract.
- `GigaHdx` → decisions must carry the **aToken** asset id as collateral, not the underlying (the pallet routes on the GIGAHDX aToken). `MmInstance.atoken_map` (underlying → aToken) is refreshed per block; `map_decision_collateral` applies it.
- Chain bootstrap reads two storage prefixes directly: `Liquidation::BorrowingContract` and `GigaHdx::GigaHdxPoolContract` (`pepl-worker/src/lib.rs → mod storage_key`).
- The registry only grows (keep-until-restart); only the operator denylist prevents creation. Unresolvable pools get exponential backoff (`POOL_RESOLVE_RETRY_BASE_BLOCKS = 4` → `MAX_POOL_RESOLVE_BACKOFF_BLOCKS = 600`).

## Coverage model

Borrower sets are a **union** that may only shrink on on-chain evidence:

1. **omniwatch seed** — `by-health` endpoint, `OMNIWATCH_FETCH_ATTEMPTS = 5`, backoff 3 s, per-attempt timeout 10 s; re-fetched every `OMNIWATCH_REFETCH_EVERY_N_BLOCKS = 100` (or every `10` until the first successful fetch). Always fetched in the background — never awaited inline.
2. **on-chain `BORROW` events** — topic `0xb3d0…dce0`, scanned from `System::Events` per block; unscanned blocks stay queued (`MAX_PENDING_EVENT_BLOCKS = 256`).
3. **on-disk cache** — `borrower_cache::{load, save}`, atomic tmp+rename, a *floor* unioned with the live fetch (never a fallback), never overwritten with an empty set.

A borrower is pruned only after `ZERO_DEBT_READS_BEFORE_PRUNE = 3` consecutive zero-debt reads.

## Decision path

```rust
// pepl-worker/src/lib.rs — pure, no I/O, unit-testable without a node
pub fn decide_liquidation(
    cfg: &LiquidationTaskConfig,
    money_market: &MoneyMarket,
    borrower: &Borrower,
) -> Option<LiquidationDecision>;

pub struct LiquidationDecision {
    pub collateral_asset: AssetId,
    pub debt_asset: AssetId,
    pub user: EvmAddress,
    pub debt_to_cover: Balance,
    pub priority: u64,      // total collateral / 1e8 — collateral at risk
}
```

Skips when collateral `< min_collateral` (`MIN_COLLATERAL_BASE = 1e8`, i.e. 1.0 base currency), when HF ≥ `ONE_HF = 1e18`, or when `calc_best_liquidation_option_for` finds no option. Then:

1. `liquidation_call(decision, pool)` → `RuntimeCall::Liquidation(liquidate_with_pool { …, unsigned_priority: Some(priority) })` — see [[wiki/pallet-liquidation\|pallet-liquidation]].
2. `dry_run_liquidation` via `xcm_runtime_apis::dry_run::DryRunApi` — `DryRun::{WouldSucceed, WouldFail, Unknown}`. `Unknown` never blocks a submission (a broken dry-run API must not silently stop liquidations). This catches `NotProfitable`, which `decide_liquidation` cannot model — it never simulates the collateral sale.
3. `encode_liquidation_opaque` → `fp_self_contained::UncheckedExtrinsic::new_bare` → `transaction_pool.submit_one(TransactionSource::Local)`.

## Oracle fast path

`run()` watches the tx pool's import notifications for pending DIA oracle-update txs from `oracle_signer` to `oracle_update_call`, then calls:

```rust
pub fn apply_oracle_updates_and_decide(
    cfg: &LiquidationTaskConfig,
    money_market: &mut MoneyMarket,      // the instance's cached market, not a re-fetch
    updates: &[(String, U256)],          // lowercased symbol → new price
    borrowers: &[Borrower],
    base_prices: &HashMap<String, U256>, // cross-market fallback denominator
) -> Vec<LiquidationDecision>;
```

- Reprices **direct** reserves (`symbol == base`) and **derived** ones (`symbol.contains(base)`, e.g. `gDOT`/`vDOT` for a `DOT` update) by scaling `new_base / old_base`.
- Both the reserve price and each borrower's already-converted base-currency collateral/debt amounts are scaled — changing the price alone would not move the health factor.
- Decides from `MmInstance.cached_mm` / `cached_borrowers` rather than re-fetching (~200 ms of runtime-API EVM calls is too slow to beat the block carrying the oracle tx). A cache older than `MAX_FAST_PATH_CACHE_AGE_BLOCKS = 3` is refused.
- `base_prices` exists because the gigahdx market holds stHDX but no plain HDX — without it every HDX update was a no-op there.

## `pepl-worker-support`

| Item | File | Role |
|---|---|---|
| `Function` (selector enum) | `support/src/lib.rs` | `getPool()`, `getReservesData(address)`, `getPriceOracle()`, `getAssetPrice(address)`, … |
| `Hydration` | `support/src/lib.rs` | `{caller, pool_address_provider, log_prefix}` + `fetch_money_market`, `fetch_pool`, `fetch_price_oracle`, `fetch_reserves_list`, `fetch_borrower` |
| `traits::{RuntimeClient, RuntimeApiProvider, RuntimeApiErr}` | `support/src/lib.rs` | node abstraction: `storage`, `call`, `address_to_asset`, `minimum_balance`, `timestamp` |
| `MoneyMarket`, `Reserve`, `ReserveData`, `UserConfiguration`, `EModeCategory` | `support/src/types.rs` | Aave-V3 model; `MoneyMarket.poisoned` records reserves that failed to load so one bad reserve skips only the borrowers holding it, not all liquidations |
| `Borrower`, `calc_health_factor` | `support/src/types.rs` | per-borrower reserve amounts already converted to base currency |
| `LiquidationOption`, `LiquidationAmounts`, `select_best_liquidation_option` | `support/src/types.rs` | picks the option closest to `target_health_factor` |
| `ApiProvider`, `RuntimeClient` (concrete) | `support/src/types.rs` | adapters over the node client / runtime APIs |
| `ray_mul`, `percent_mul`, `wad_div`, `pow10_u256/u512` | `support/src/math.rs` | Aave fixed-point primitives |

Runtime APIs consumed: `EthereumRuntimeRPCApi` (EVM `call`), `Erc20MappingApi` (asset ↔ EVM address), `CurrenciesApi` (minimum balance), `DryRunApi`. No new runtime API was added for the worker.

## Hard-coded mainnet addresses (`pepl-worker/src/lib.rs → mod contracts`)

| Constant | Address |
|---|---|
| `POOL_ADDRESS_PROVIDER` | `0xf3ba4d1b50f78301bdd7eaea9b67822a15fca691` |
| `RUNTIME_API_CALLER` | `0x33a5e905fB83FcFB62B0Dd1595DfBc06792E054e` (must hold WETH) |
| `ORACLE_SIGNER` | `0x33a5e905…`, `0xff0c6240…` |
| `ORACLE_UPDATE_CALL` | `0xdee629af…`, `0x48ae7803…` |

## Gotchas

- `liquidations_per_block` bounds only the **oracle fast path**. The per-block scan submits on find; its backstop is `MAX_SCAN_DRY_RUNS_PER_BLOCK = 256`, set above what a block can execute (~250–600 `liquidate` calls fit the normal PoV budget), so it only ever trims work that could not have landed anyway.
- A malformed `--omniwatch-url` does **not** panic — `LiquidationTask::new` runs inside `start_node_impl`, so it degrades to no seeding and relies on `BORROW` discovery + the disk cache.
- Decisions for a `GigaHdx` instance must carry the aToken asset id; sending the underlying makes the pallet reject the call.
- `WorkerStatus` accessors degrade to "unknown" rather than blocking/panicking on a poisoned lock, because monitoring depends on them. `liquidation_isRunning` reports false after `WORKER_LIVENESS_WINDOW = 120 s` without a completed scan.
- `liquidation_maxTransactionsPerBlock` keeps its v1 name but no longer bounds every submission path — v1 returned a weight-derived global cap.
- Log target is `liquidation-worker` for **both** workers; distinguish them by the `pepl-worker` / `pepl-worker/mm-xxxxxxxx` log prefix.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hydration-runtime\|hydration-runtime]]
- [[wiki/pallet-liquidation\|pallet-liquidation]]
