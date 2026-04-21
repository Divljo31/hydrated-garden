---
{"dg-publish":true,"permalink":"/wiki/pallet-transaction-pause/","title":"pallet-transaction-pause","tags":["governance","pause","risk","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-transaction-pause","repo":"hydration-node","paths":["pallets/transaction-pause/src/lib.rs"],"symbols":["Pallet","Config","PausedTransactions","pause_transaction","unpause_transaction","PausedTransactionFilter"],"traits_impl":["Contains"],"depends_on":[],"runtime_index":60,"tags":["governance","pause","risk","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-transaction-pause

**TL;DR:** Governance kill-switch. Maintains a set of `(pallet_name, function_name)` pairs that are blocked; the runtime's `BaseCallFilter` consults this set before dispatching any call. Runtime index = 60.

## Role

Emergency response tool. Allows governance (or a fast-track technical origin) to disable specific extrinsics without a full runtime upgrade when a bug/exploit is discovered.

## Config trait (excerpt)

```rust
// pallets/transaction-pause/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type RuntimeCall: Parameter + GetCallMetadata;
    type UpdateOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    #[pallet::constant] type MaxNameLen: Get<u32>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `PausedTransactions` | StorageMap | `(BoundedVec<u8, MaxNameLen>, BoundedVec<u8, MaxNameLen>) → ()` |

## Events

`TransactionPaused { pallet_name, function_name }`, `TransactionUnpaused { pallet_name, function_name }`.

## Errors

`CannotPause`, `InvalidCharacter`, `NotPaused`.

## Extrinsics

| Name | Description |
|------|-------------|
| `pause_transaction` | Add `(pallet_name, function_name)` to paused set (UpdateOrigin) |
| `unpause_transaction` | Remove from paused set (UpdateOrigin) |

## Hooks

None.

## Integration

- **Traits implemented:** `Contains<RuntimeCall>` — wired into the runtime's `BaseCallFilter`
- **Traits consumed:** `GetCallMetadata` (to extract pallet/function name from a call)
- **Pallets depended on:** none

## Key logic: filter

```rust
// pallets/transaction-pause/src/lib.rs
impl<T: Config> Contains<T::RuntimeCall> for PausedTransactionFilter<T>
where <T as frame_system::Config>::RuntimeCall: GetCallMetadata {
    fn contains(call: &T::RuntimeCall) -> bool {
        let CallMetadata { function_name, pallet_name } = call.get_call_metadata();
        PausedTransactions::<T>::contains_key((pallet_name.as_bytes().to_vec(), function_name.as_bytes().to_vec()))
    }
}
```

## Gotchas

- Cannot pause calls from `pallet-transaction-pause` itself (prevents locking out the pause mechanism).
- Matches by *string* name — pallet/function names must be exact. Renames in runtime upgrades invalidate paused entries.
- The runtime's `BaseCallFilter` is typically composed: `(Not<PausedTransactionFilter>, AllOtherFilters)` — so pausing takes effect immediately block-to-block.
- Used historically after audit findings to disable vulnerable extrinsics within a single block via OpenGov.
- Paused transactions still consume nonce/fee (pre-dispatch validates) — post-dispatch check is what rejects them.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
