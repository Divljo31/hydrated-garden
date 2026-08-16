---
{"dg-publish":true,"permalink":"/wiki/pallet-parameters/","title":"pallet-parameters","tags":["parameters","runtime","flags","testnet","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-parameters","repo":"hydration-node","paths":["pallets/parameters/src/lib.rs"],"symbols":["Pallet","Config","IsTestnet","RelayParentOffsetOverride","GenesisConfig","is_testnet","relay_parent_offset_override","set_testnet_flag","set_relay_parent_offset_override"],"traits_impl":[],"depends_on":[],"runtime_index":83,"tags":["parameters","runtime","flags","testnet","rust","substrate"],"last_updated":"2026-05-14"}}
---


# pallet-parameters

**TL;DR:** Hydration-specific (NOT the FRAME `pallet_parameters`) storage for two genesis-set boolean flags: `IsTestnet` and `RelayParentOffsetOverride`. Used by adapters / runtime code to branch behaviour on testnet versus mainnet, and to override relay-parent-offset enforcement in test environments. Runtime index = 83. **No extrinsics, no events, no errors** — purely a state holder.

## Role

Two flags that need to be readable at runtime without rebuilding the WASM:

- **`IsTestnet`** — distinguishes testnet from mainnet at runtime. Consulted by various adapters (e.g. relaxing assertions, gating mock behaviour).
- **`RelayParentOffsetOverride`** — overrides the normal relay-parent offset rule. Used in fork-test environments where the relay-chain block number cannot be guaranteed to advance in the expected way.

Both values are seeded at genesis (`GenesisConfig`) and never change on a running chain except in tests, where `set_testnet_flag` / `set_relay_parent_offset_override` are exposed under `#[cfg(feature = "std")]`.

## Config trait

```rust
// pallets/parameters/src/lib.rs
pub trait Config: frame_system::Config {}
```

The entire Config is empty — the pallet is just a typed storage shell scoped to a runtime.

## Storage

| Name | Kind | Default |
|---|---|---|
| `IsTestnet` | `StorageValue<_, bool, ValueQuery>` | `false` |
| `RelayParentOffsetOverride` | `StorageValue<_, bool, ValueQuery>` | `false` |

## Genesis config

```rust
pub struct GenesisConfig<T: Config> {
    pub is_testnet: bool,
    pub relay_parent_offset_override: bool,
    pub _phantom: PhantomData<T>,
}
```

Both default to `false`, so mainnet builds do not need explicit configuration. Testnets / chopsticks / zombienet configurations set these in their chain specs.

## Events / Errors / Extrinsics

None — the pallet has no `#[pallet::event]`, `#[pallet::error]`, or `#[pallet::call]`. State is set only at genesis and via the test-only setters.

## Helper API

```rust
#[cfg(feature = "std")]
impl<T: Config> Pallet<T> {
    pub fn set_testnet_flag(is_testnet: bool);
    pub fn set_relay_parent_offset_override(override_enabled: bool);
}
```

Used only from test mocks.

## Integration

- **Traits implemented:** none external
- **Traits consumed:** none
- **Pallets depended on:** none. Read directly via `pallet_parameters::IsTestnet::<Runtime>::get()` from runtime adapters and helpers.

## Gotchas

- **Not the FRAME `pallet_parameters`.** Do not confuse with `frame_support::parameter_types!` / the upstream `pallet_parameters` (which would let governance update typed key/value pairs). Hydration does **not** use that pallet today; all runtime knobs are either `pallet::constant` types or dedicated pallets (e.g. [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]], [[wiki/pallet-dynamic-evm-fee\|pallet-dynamic-evm-fee]]).
- **Empty Config trait.** Adding new flags requires a runtime upgrade (new storage value + genesis migration). It is not a generic parameters store.
- **Genesis-only mutability in production.** No extrinsic exposes a write path; `set_testnet_flag` and `set_relay_parent_offset_override` are gated behind `#[cfg(feature = "std")]` so they exist only in the host binary, not in the WASM runtime.
- A previous wiki revision documented this pallet as a generic `AggregatedKeyValue` store with a `set_parameter` extrinsic — that was inaccurate. Disregard those references.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hydration-runtime\|hydration-runtime]]
