---
{"dg-publish":true,"permalink":"/wiki/pallet-genesis-history/","title":"pallet-genesis-history","tags":["genesis","provenance","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-genesis-history","repo":"hydration-node","paths":["pallets/genesis-history/src/lib.rs"],"symbols":["Pallet","Config","PreviousChain","Chain"],"traits_impl":[],"depends_on":[],"runtime_index":55,"tags":["genesis","provenance","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-genesis-history

**TL;DR:** Records the previous chain's name + genesis hash so post-migration chains (Hydration is a rename/relaunch of HydraDX) can prove provenance on-chain. No extrinsics, no hooks — purely a `StorageValue` populated at genesis. Runtime index = 55.

## Role

When Hydration launched as a re-genesis of the prior HydraDX chain, this pallet captures "what chain did we come from" in on-chain storage so any verifier (bridges, indexers, auditors) can read it from state.

## Config trait (excerpt)

```rust
// pallets/genesis-history/src/lib.rs
pub trait Config: frame_system::Config {}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `PreviousChain` | StorageValue | `Chain { name: Vec<u8>, genesis_hash: Vec<u8> }` |

## Events

None.

## Errors

None.

## Extrinsics

None.

## Hooks

None (no runtime logic).

## Integration

- **Traits implemented:** none external
- **Traits consumed:** none
- **Pallets depended on:** none

## Key type

```rust
// pallets/genesis-history/src/lib.rs
pub struct Chain {
    pub name: Vec<u8>,
    pub genesis_hash: Vec<u8>,
}

#[pallet::genesis_config]
pub struct GenesisConfig { pub previous_chain: Chain }
// `build` writes previous_chain into the PreviousChain storage value.
```

## Gotchas

- Storage value is set at genesis and never modified — no extrinsic exists to change it.
- `Chain.name` and `Chain.genesis_hash` are unbounded `Vec<u8>` — deliberately not `BoundedVec` since it's only ever written once at genesis.
- Consumers just read the value; any migrations should keep this pallet intact to preserve chain history.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
