---
{"dg-publish":true,"permalink":"/wiki/pallet-intent/","title":"pallet-intent","tags":["ice","intents","dca","trading","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-intent","repo":"hydration-node","paths":["pallets/intent/src/lib.rs","pallets/intent/src/types.rs","pallets/intent/contracts/IIntentResolutionReceiver.sol","pallets/ice/support/src/lib.rs","runtime/hydradx/src/assets.rs","runtime/hydradx/src/ice_oracle_routes.rs"],"symbols":["Pallet","Config","Intent","IntentInput","IntentData","IntentDataInput","SwapParams","SwapData","DcaParams","DcaData","Partial","OnResolved","Intents","IntentOwner","NextIncrementalId","AccountIntents","AccountIntentCount","submit_intent","remove_intent","cleanup_intent","add_intent","cancel_intent","do_add_migrated_intent","get_valid_intents","solver_intents","validate_resolve","intent_resolved","IntentMigrator","NAMED_RESERVE_ID","SettlementEnabled","MaxDcaSlippage","DcaEarlyWindow","SolverDeadlineMargin","DerivedRouteShortPrice"],"traits_impl":["IntentMigrator","ValidateUnsigned"],"depends_on":["pallet-ice","pallet-lazy-executor","pallet-asset-registry","pallet-currencies","pallet-route-executor","pallet-ema-oracle","pallet-dca"],"runtime_index":98,"tags":["ice","intents","dca","trading","runtime","rust","substrate"],"last_updated":"2026-11-02"}}
---


# pallet-intent

**TL;DR:** Storage and lifecycle of user **intents** for [[wiki/ice\|ice]]: one-shot swaps (optionally partially fillable), match-first swaps, and DCA intents. `submit_intent` reserves the input funds under named reserve `ICE_int#`; [[wiki/pallet-ice\|pallet-ice]] settles intents in solver batches; expired intents are cleaned by an OCW. Also the migration target for legacy [[wiki/pallet-dca\|pallet-dca]] schedules (`IntentMigrator`). Runtime index = 98.

## Intent kinds (`ice_support`)

| Input (`IntentDataInput`) | Stored (`IntentData`) | Semantics |
|---|---|---|
| `Swap(SwapParams { asset_in, asset_out, amount_in, amount_out, partial: bool })` | `Swap(SwapData { …, partial: Partial })` | Sell `amount_in` for ≥ `amount_out`. `Partial::Yes(filled)` tracks cumulative fill; min rate = `amount_out / amount_in` forever |
| `MatchFirstSwap { swap, match_window }` (ms) | `MatchFirstSwap { swap, match_until }` | Offered to the solver as a plain `Swap`, but **only against opposite intent flow** until `match_until` |
| `Dca(DcaParams { asset_in, asset_out, amount_in, amount_out, slippage, budget: Option, period })` | `Dca(DcaData { …, remaining_budget, last_execution_block })` | Per-period exact sell; enforced min = `max(amount_out hard floor, oracle price − slippage)`; `budget: None` = rolling |

Intent wrapper: `Intent { data, deadline: Option<Moment ms>, on_resolved: Option<OnResolved> }`. `OnResolved::Forward { contract: EvmAddress, data: BoundedVec<u8, 512> }` — after (each) resolution the output is pushed to `contract` and its `IIntentResolutionReceiver` interface is called, via [[wiki/pallet-lazy-executor\|pallet-lazy-executor]]. Append-only enum (index-stable).

## Config (runtime values)

| Item | Runtime |
|---|---|
| `Currency` | `Currencies` (named reserves) |
| `LazyExecutorHandler` | `LazyExecutor` |
| `RegistryHandler` | `AssetRegistry` |
| `HubAssetId` | `LRNA` |
| `MaxAllowedIntentDuration` | 24 h (ms) |
| `SolverDeadlineMargin` | `MILLISECS_PER_BLOCK` (an intent expiring before the next block is not offered) |
| `OraclePriceProvider` | `ice_oracle_routes::DerivedRouteShortPrice` (oracle price along the route the trade actually takes) |
| `MinDcaPeriod` | `MinimalPeriod` (15 blocks) |
| `MaxDcaSlippage` | 50% |
| `DcaEarlyWindow` | 150 blocks (≈5 min), capped at period/2 |
| `SettlementEnabled` | `IceSettlementEnabled` → `pallet_ice::settlement_enabled()` |
| `MaxIntentsPerAccount` | 100 |

## Storage

| Name | Kind | Key → Value |
|---|---|---|
| `Intents` | Map | `IntentId → Intent` |
| `IntentOwner` | Map | `IntentId → AccountId` |
| `NextIncrementalId` | Value | `u64` |
| `AccountIntents` | DoubleMap | `(AccountId, IntentId) → ()` reverse index |
| `AccountIntentCount` | Map | `AccountId → u32` (cap `MaxIntentsPerAccount`) |

## Extrinsics

| # | Name | Origin | Description |
|---|---|---|---|
| 0 | `submit_intent(IntentInput)` | Signed | Validate, reserve funds, store. Refused with `SettlementDisabled` when ICE solver mode is `Disabled` |
| 1 | `remove_intent(id)` | Signed (owner) | Cancel + unreserve |
| 2 | `cleanup_intent(id)` | Unsigned (OCW, `Local`/`InBlock` only) | Remove an expired intent, unreserve, queue its callback if any |

Events: `IntentSubmitted`, `IntentResolved`, `IntentResovedPartially` (sic), `IntentCanceled`, `IntentExpired`, `FailedToQueueCallback`, `DcaTradeExecuted`, `DcaCompleted`.

Errors: `InvalidDeadline`, `InvalidIntent`, `InvalidForwardContract`, `IntentNotFound`, `IntentExpired`, `IntentActive`, `ResolveMismatch`, `LimitViolation`, `ArithmeticOverflow`, `IntentOwnerNotFound`, `InvalidOwner`, `InsufficientReservedBalance`, `NotImplemented`, `AssetNotFound`, `InvalidDcaPeriod`, `InvalidDcaBudget`, `InvalidDcaDeadline`, `InvalidDcaSlippage`, `SettlementDisabled`, `MaxIntentsReached`, `InvalidMatchWindow`.

## Solver-facing API (called by [[wiki/pallet-ice\|pallet-ice]])

- `get_valid_intents()` / `solver_intents()` — intents still valid at `now + SolverDeadlineMargin`; DCA intents are offered when due (or within `DcaEarlyWindow` if reverse-pair flow covers them), converted to `Swap` with the effective min as `amount_out`. Returns per-intent DCA admission floors.
- `validate_resolve(intent, resolve)` — checks a proposed resolution against the stored limit / partial state.
- `intent_resolved(who, resolved)` — applies fills, emits events, removes or updates the intent, queues `OnResolved` callbacks.
- `do_add_migrated_intent(owner, DcaParams)` (`IntentMigrator`) — used by [[wiki/pallet-dca\|pallet-dca]] migration; bypasses the per-account cap, lifts `amount_out` to the output ED, tightens (does not refuse) slippage above `MaxDcaSlippage`.

## Hooks

`offchain_worker`: submits up to 10 `cleanup_intent` unsigned txs per block for expired intents (in-source note: temporary, to be replaced by an off-chain bot).

## Gotchas

- Funds are **reserved**, not transferred, until settlement — reserve id `ICE_int#`. Settlement repatriates reserved balances (for ERC-20-bound assets see [[wiki/pallet-currencies\|pallet-currencies]] receipt semantics).
- A DCA intent has no deadline, so it never expires — submission is refused while settlement is disabled so budgets can't get stuck.
- MatchFirstSwap's counter-flow condition is checked at admission only; a solution can still route it through an AMM. Safe only because just the block author's own solver can submit (`propagate(false)`, `Local`/`InBlock`).
- All deadlines are **milliseconds** (`Moment`), periods are **blocks** (2 s).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/ice\|ice]], [[wiki/pallet-ice\|pallet-ice]], [[wiki/pallet-lazy-executor\|pallet-lazy-executor]], [[wiki/pallet-dca\|pallet-dca]]
