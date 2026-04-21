---
{"dg-publish":true,"permalink":"/wiki/pallet-dispatcher/","title":"pallet-dispatcher","tags":["dispatcher","batching","governance","ismp","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-dispatcher","repo":"hydration-node","paths":["pallets/dispatcher/src/lib.rs"],"symbols":["Pallet","Config","dispatch_as_treasury","dispatch_as_aave_manager","dispatch_with_extra_gas","note_ismp_responses","CLEANUP_LIMIT"],"traits_impl":[],"depends_on":[],"runtime_index":40,"tags":["dispatcher","batching","governance","ismp","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-dispatcher

**TL;DR:** Authorized-origin call dispatcher — lets specific origins (Treasury, AaveManager, EmergencyAdmin) execute calls as a pallet-controlled account, plus a gas-augmented dispatch path for heavy EVM calls, plus an ISMP response cleanup hook via `on_idle`. Runtime index = 40.

## Role

Three jobs rolled into one pallet:
1. **Dispatch-as**: Governance (or specific privileged origins) can dispatch inner calls as if signed by a designated managed account (Treasury account, Aave manager account).
2. **Extra gas**: Wrap an inner call with a declared extra gas allowance, used for EVM-heavy dispatches where standard weight estimation is too conservative.
3. **ISMP housekeeping**: `on_idle` cleans up stale ISMP responses (up to `CLEANUP_LIMIT` per block).

## Config trait (excerpt)

```rust
// pallets/dispatcher/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: ...;
    type RuntimeCall: Parameter + Dispatchable<...> + GetDispatchInfo + From<Call<Self>>;
    type TreasuryManagerOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type AaveManagerOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type DefaultAaveManagerAccount: Get<Self::AccountId>;
    type TreasuryAccount: Get<Self::AccountId>;
    type GasWeightMapping: GasWeightMapping;
    type WeightInfo: WeightInfo;
}

pub const CLEANUP_LIMIT: u32 = 100;
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `AaveManagerAccount` | StorageValue | `AccountId` (OptionQuery → falls back to DefaultAaveManagerAccount) |

## Events

`TreasuryManagerCallDispatched { result }`, `AaveManagerCallDispatched { result }`.

## Errors

None unique — inner call errors propagate.

## Extrinsics

| Name | Description |
|------|-------------|
| `dispatch_as_treasury` | `TreasuryManagerOrigin` dispatches call as `TreasuryAccount` signed origin |
| `dispatch_as_aave_manager` | `AaveManagerOrigin` dispatches call as Aave manager account signed origin |
| `note_aave_manager` | Root sets the Aave manager account in `AaveManagerAccount` |
| `dispatch_with_extra_gas` | Any signed origin; wraps inner call with `extra_gas` converted to weight via `GasWeightMapping` |

## Hooks

`on_idle(_, remaining)` — calls `pallet_ismp::Pallet::cleanup_requests(CLEANUP_LIMIT)` if enough weight; returns consumed weight.

## Integration

- **Traits implemented:** none external
- **Traits consumed:** `GasWeightMapping` (EVM gas ↔ Substrate weight conversion)
- **Pallets depended on:** `pallet_ismp` (for cleanup), otherwise none

## Key extrinsic

```rust
// pallets/dispatcher/src/lib.rs
pub fn dispatch_as_treasury(
    origin: OriginFor<T>, call: Box<<T as Config>::RuntimeCall>,
) -> DispatchResultWithPostInfo {
    T::TreasuryManagerOrigin::ensure_origin(origin)?;
    let treasury_origin = RawOrigin::Signed(T::TreasuryAccount::get()).into();
    let res = call.dispatch(treasury_origin);
    Self::deposit_event(Event::TreasuryManagerCallDispatched {
        result: res.map(|_| ()).map_err(|e| e.error),
    });
    res
}
```

## Gotchas

- `dispatch_with_extra_gas` is a gas inflation tool — be careful with weight accounting; the post-dispatch weight equals declared-extra + actual inner.
- Treasury/Aave manager accounts dispatched as `Signed` — the inner call runs with that account as the caller's AccountId; they can therefore hold balances, be referenced in asset registrations, etc.
- `AaveManagerAccount` being empty means callers get `DefaultAaveManagerAccount` — never a panic, always a fallback.
- `on_idle` ISMP cleanup only triggers if `pallet_ismp` is present in the runtime; otherwise it's a no-op (guarded by trait bound).
- No pausable / emergency-halt functionality here — see [[wiki/pallet-transaction-pause\|pallet-transaction-pause]].

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
