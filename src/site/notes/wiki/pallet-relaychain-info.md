---
{"dg-publish":true,"permalink":"/wiki/pallet-relaychain-info/","title":"pallet-relaychain-info","tags":["relaychain","parachain","cumulus","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-relaychain-info","repo":"hydration-node","paths":["pallets/relaychain-info/src/lib.rs"],"symbols":["Pallet","Config","OnValidationData","CurrentBlockNumbers"],"traits_impl":[],"depends_on":[],"runtime_index":201,"tags":["relaychain","parachain","cumulus","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-relaychain-info

**TL;DR:** Emits a `CurrentBlockNumbers` event every block containing the parachain and Polkadot relay chain block numbers. Used by off-chain indexers to align parachain state with relay-chain epochs. Runtime index = 201.

## Role

Cumulus gives the runtime access to validation data (relay parent number) via `cumulus_pallet_parachain_system`. This pallet hooks `on_validation_data` to emit both numbers in a single event, making indexer cross-referencing trivial.

## Config trait (excerpt)

```rust
// pallets/relaychain-info/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type RelaychainBlockNumberProvider: BlockNumberProvider;
}
```

## Storage

None.

## Events

`CurrentBlockNumbers { parachain_block_number: BlockNumber, relaychain_block_number: BlockNumber }`.

## Errors

None.

## Extrinsics

None.

## Hooks

Implements `OnValidationData` (cumulus hook) — runs once per parablock when relay data is injected:

```rust
impl<T: Config> OnValidationData for Pallet<T> {
    fn on_validation_data(data: PersistedValidationData) {
        Self::deposit_event(Event::CurrentBlockNumbers {
            parachain_block_number: <frame_system::Pallet<T>>::block_number(),
            relaychain_block_number: data.relay_parent_number,
        });
    }
}
```

## Integration

- **Traits implemented:** `OnValidationData`
- **Traits consumed:** `BlockNumberProvider` (for the relay chain number, typically `cumulus_pallet_parachain_system::RelaychainDataProvider`)
- **Pallets depended on:** `cumulus_pallet_parachain_system` (implicitly — it's what calls `on_validation_data`)

## Gotchas

- Event fires every parablock, so indexers see one `CurrentBlockNumbers` per block.
- `relaychain_block_number` here is the **relay parent** — not the latest finalized relay block, but the one this parablock was built against.
- If Hydration is ever run solo (not as a parachain), `OnValidationData` never fires and no event is emitted.
- This pallet has no storage, extrinsics, or errors — it's a pure informational emitter.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
