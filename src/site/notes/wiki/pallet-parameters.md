---
{"dg-publish":true,"permalink":"/wiki/pallet-parameters/","title":"pallet-parameters","tags":["parameters","governance","dynamic-config","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-parameters","repo":"hydration-node","paths":["pallets/parameters/src/lib.rs"],"symbols":["Pallet","Config","set_parameter","Parameters","RuntimeParameters","AggregatedKeyValue"],"traits_impl":["Get"],"depends_on":[],"runtime_index":83,"tags":["parameters","governance","dynamic-config","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-parameters

**TL;DR:** Governance-editable runtime parameters. Substitutes for `pallet::constant` types that need to be changed without a runtime upgrade. Uses `frame_support::parameter_types!`-style aggregation + the `AggregatedKeyValue` trait to store typed key/value pairs in one map. Runtime index = 83.

## Role

Some runtime knobs (fees, thresholds, slashes, limits) need to change frequently via governance. Changing a `#[pallet::constant]` requires a runtime upgrade; this pallet replaces those constants with storage-backed parameters that `AdminOrigin` can update on the fly.

## Config trait (excerpt)

```rust
// pallets/parameters/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type RuntimeParameters: AggregatedKeyValue;
    type AdminOrigin: EnsureOriginWithArg<
        Self::RuntimeOrigin,
        <Self::RuntimeParameters as AggregatedKeyValue>::Key,
    >;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Parameters` | StorageMap | `<RuntimeParameters as AggregatedKeyValue>::Key → Option<<RuntimeParameters as AggregatedKeyValue>::Value>` |

## Events

`Updated { key_value: RuntimeParameters }` — emitted on every successful `set_parameter`.

## Errors

None unique; `AdminOrigin` failures surface as `BadOrigin`.

## Extrinsics

| Name | Description |
|------|-------------|
| `set_parameter` | `AdminOrigin` (per-key) sets or clears the parameter. `None` value deletes. |

## Hooks

None.

## Integration

- **Traits implemented:** indirectly provides `Get<T>` accessors for each parameter (via `define_aggregated_parameters!` macro expansion in the runtime)
- **Traits consumed:** `AggregatedKeyValue` (provided by runtime via macro), `EnsureOriginWithArg`
- **Pallets depended on:** none; governance pallets (OpenGov referenda) are the callers

## Key pattern

```rust
// runtime/hydradx/src/governance/mod.rs (pattern)
define_aggregated_parameters! {
    pub RuntimeParameters = {
        Dynamic: dynamic::Parameters = 0,
        Omnipool: omnipool::Parameters = 1,
        ...
    }
}

// usage as a Get<_>:
type MinimumPoolLiquidity = dynamic::MinimumPoolLiquidity;
// which reads Parameters::<Runtime>::get(key) under the hood
```

## Gotchas

- `AdminOrigin` is `EnsureOriginWithArg` — the origin check receives the key as argument, letting the runtime gate different keys to different origins (e.g. some parameters Root-only, some Treasury Origin).
- A parameter left `None` means "no value set" — callers using `Get<T>` must define a default; the macro typically returns `Default::default()`.
- Changes are immediate (next block) — no cooloff.
- Best practice: group related parameters under one "section" struct so their `AdminOrigin` policy can be unified.
- Governance calls go through `pallet_referenda` / OpenGov → dispatcher → `set_parameter`.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
