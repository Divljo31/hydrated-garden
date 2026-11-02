---
{"dg-publish":true,"permalink":"/wiki/pallet-ice/","title":"pallet-ice","tags":["ice","intents","solver","settlement","ocw","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-ice","repo":"hydration-node","paths":["pallets/ice/src/lib.rs","pallets/ice/support/src/lib.rs","pallets/ice/runtime-api/src/lib.rs","pallets/ice/amm-simulator/src/lib.rs","ice/ice-solver/src/lib.rs","ice/route-findr/SPEC.md","node/src/ice_solver_worker.rs","runtime/hydradx/src/assets.rs","runtime/hydradx/src/ice_simulator_provider.rs"],"symbols":["Pallet","Config","Solution","ResolvedIntent","PoolTrade","SolverMode","ExecutionPolicy","RoutingTarget","RoutingState","ProtocolFee","CurrentSolverMode","SolverRouting","submit_solution","set_protocol_fee","set_solver_mode","update_routing","settlement_enabled","solver_input","liquidity_state","IceSolverApi","LiquidityStateApi","HydrationSimulatorConfig","SmartRouteFinder","MatchedFee","IceFee","IceFeeReceiver"],"traits_impl":["ValidateUnsigned"],"depends_on":["pallet-intent","pallet-route-executor","pallet-omnipool","pallet-stableswap","pallet-xyk","hydration-borrow","pallet-dispatcher"],"runtime_index":99,"tags":["ice","intents","solver","settlement","ocw","runtime","rust","substrate"],"last_updated":"2026-11-02"}}
---


# pallet-ice

**TL;DR:** Settlement layer of [[wiki/ice\|ice]]. A node-side solver (`node/src/ice_solver_worker.rs`) reads `solver_input()` each best block, solves natively (`ice/ice-solver`), and submits `submit_solution` as a **local-only unsigned** extrinsic. The pallet validates and executes the batch: intent-to-intent matched volume plus residual trades through [[wiki/pallet-route-executor\|pallet-route-executor]]. Governance controls a matched-volume protocol fee, the solver mode (`V4` / `Passthrough` / `Disabled` = kill switch) and which venues the solver may route through. Runtime index = 99.

## Flow

1. Users submit intents to [[wiki/pallet-intent\|pallet-intent]] (funds reserved).
2. Each new best block, the node's ICE worker calls `IceSolverApi::solver_input()` → valid intents, SCALE-encoded simulator snapshot (Omnipool, Stableswap, Aave wraps, Uniswap v3, XYK), ED map, DCA admission floors, fee, mode.
3. The solver (generation selected by `CurrentSolverMode`) matches opposite intents, routes the remainder (`route-findr` BFS over pool edges), and builds `Solution { resolved_intents, trades: Vec<PoolTrade>, score, … }`.
4. `submit_solution` (unsigned, `longevity(1)`, `propagate(false)`) executes in the next block. Overlapping solves are dropped (busy guard).

## Config (runtime)

| Item | Runtime |
|---|---|
| `PalletId` | `ice_ice#` (holding pot) |
| `MatchedFee` | `IceFee` = 0.02% (`Permill::from_parts(200)`) — on matched volume only |
| `FeeReceiver` | `ice_fee#` pallet account |
| `AuthorityOrigin` | `Root \| TechCommitteeMajority` |
| `Simulator` | `HydrationSimulatorConfig` — simulators `(Omnipool, StableSwap, Aave, UniswapV3, Xyk)`, route discovery `SmartRouteFinder` (`route_findr::get_routes` over `pool_edges`), price denominator HDX |
| `ExtraGasSupport` | `Dispatcher` |

## Storage

| Name | Kind | Value |
|---|---|---|
| `ProtocolFee` | Value | `Permill`, default `MatchedFee` |
| `CurrentSolverMode` | Value | `SolverMode` — absent = `V4` |
| `SolverRouting` | Map | `RoutingTarget → RoutingState (Included \| Excluded)` |

`RoutingTarget`: `OmnipoolAsset(id)`, `StableswapPool(id)`, `XykPool(a,b)`, `AaveWrap(reserve, aToken)`, `UniswapV3Pool(H160)`, batch forms `AaveWraps`, `XykPools`, `UniswapV3Pools`. Omnipool and Stableswap are **opt-out** (absent = included); every other venue is **opt-in**. Exclusion wins over any batch that lists the target. Targets are normalized (pairs ordered, batches sorted/deduped).

## Extrinsics

| # | Name | Origin | Description |
|---|---|---|---|
| 0 | `submit_solution(Solution)` | none (OCW, `Local`/`InBlock`) | Execute a batch per the active mode |
| 1 | `set_protocol_fee(Permill)` | Authority | Matched-volume fee |
| 2 | `set_solver_mode(SolverMode)` | Authority | Switch solver / execution policy |
| 3 | `update_routing(target, Option<RoutingState>)` | Authority | Include / exclude / clear a venue for the solver |

Events: `SolutionExecuted`, `ProtocolFeeSet`, `SolverModeSet`, `RoutingUpdated`. Errors: `InvalidSolution`, `IntentNotFound`, `IntentOwnerNotFound`, `LimitViolation`, `PriceInconsistency`, `DuplicateIntent`, `InvalidRoute`, `ScoreMismatch`, `UnsupportedIntentKind`, `ArithmeticOverflow`, `AssetNotFound`, `InvalidAmount`, `FeeMismatch`.

## Solver modes

| `SolverMode` | Execution policy | Behaviour |
|---|---|---|
| `V4` (default) | Strict | Whole-batch: claimed amounts and `score` are binding; any failure aborts the batch |
| `Passthrough` | Passthrough | Emergency drain: each intent executed independently as a plain router trade, bound by its stored limit; failing intents skipped; claimed amounts advisory |
| `Disabled` | none | Kill switch — no solution accepted; [[wiki/pallet-intent\|pallet-intent]] refuses new intents (`SettlementDisabled`) |

## Fee invariant

Matched-volume fee is charged only on the part of each intent's `amount_out` that was matched intent-to-intent (AMM-routed volume pays pool fees only). The user always receives the resolved net `amount_out`. After execution, per touched asset, the holding-pot residual must cover the expected fee (`FeeMismatch` otherwise); the fee is swept to `FeeReceiver`.

## Runtime APIs

`IceSolverApi::solver_input() -> Option<SolverInput { intents, state, existential_deposits, min_amount_out, fee, mode }>`; `LiquidityStateApi::liquidity_state() -> LiquidityState { state, existential_deposits }` (also used by [[wiki/pepl-worker\|pepl-worker]] for liquidation routing).

## Gotchas

- Only the block author's own node can submit solutions (unsigned, non-propagating). A malicious collator could submit an arbitrary-but-valid solution; validation (limits, prices, fee invariant) is what protects users.
- `set_solver_mode(Disabled)` is the ICE kill switch (TC majority). `Passthrough` lets users exit at their own limits without trusting the batch solver.
- Opt-in venues (Aave wraps, XYK, Uniswap v3) are invisible to the solver until `update_routing(…, Some(Included))`; to stop routing through a compromised Omnipool asset or stablepool use `Excluded`.
- `amount_in` / `amount_out` must exceed the asset ED (validated in `validate_unsigned` and `submit_solution`).
- Weight = per-intent settlement + price derivation, plus router weight per trade; DCA-heavy batches are the worst case.
- Off-chain crates: `ice/ice-solver` (v4 + passthrough solvers, flow graph, ring detection, splits), `ice/route-findr` (route discovery, spec in `SPEC.md`), `ice/solver-bench` (bench, fuzz, replay).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/ice\|ice]], [[wiki/pallet-intent\|pallet-intent]], [[wiki/pallet-lazy-executor\|pallet-lazy-executor]], [[wiki/pallet-route-executor\|pallet-route-executor]]
