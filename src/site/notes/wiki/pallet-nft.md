---
{"dg-publish":true,"permalink":"/wiki/pallet-nft/","title":"pallet-nft","tags":["nft","uniques","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-nft","repo":"hydration-node","paths":["pallets/nft/src/lib.rs","pallets/nft/src/types.rs"],"symbols":["Pallet","Config","create_collection","mint","transfer","burn","destroy_collection","Collections","Items","CollectionInfo","ItemInfo","NftPermission","NftCollectionId","NftItemId"],"traits_impl":[],"depends_on":[],"runtime_index":null,"tags":["nft","uniques","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-nft

**TL;DR:** Thin wrapper over `pallet_uniques` that adds Hydration-specific collection typing + permission system. NOT individually wired in `construct_runtime!` — exposed through `pallet_uniques` (index 32). Provides the LP-position NFT infrastructure used by [[wiki/pallet-omnipool\|pallet-omnipool]], [[wiki/pallet-staking\|pallet-staking]], and the LM pallets.

## Role

Uniques is generic; Hydration needs typed collections (Omnipool LP, staking position, LM deposit, marketplace) each with different permissions (who may mint/burn/transfer). This pallet stores the collection type and permission metadata on top of uniques storage.

## Config trait (excerpt)

```rust
// pallets/nft/src/lib.rs
pub trait Config: frame_system::Config + pallet_uniques::Config {
    type WeightInfo: WeightInfo;
    type NftCollectionId: Member + Parameter + Default + Copy + HasCompact
        + AtLeast32BitUnsigned + Into<Self::CollectionId> + From<Self::CollectionId> + ...;
    type NftItemId: Member + Parameter + Default + Copy + HasCompact
        + AtLeast32BitUnsigned + Into<Self::ItemId> + From<Self::ItemId> + ...;
    type CollectionType: Member + Parameter + Default + Copy + MaxEncodedLen;
    type Permissions: NftPermission<Self::CollectionType>;
    #[pallet::constant] type ReserveCollectionIdUpTo: Get<Self::NftCollectionId>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Collections` | StorageMap | `NftCollectionId → CollectionInfo<CollectionType, Metadata>` |
| `Items` | StorageDoubleMap | `(NftCollectionId, NftItemId) → ItemInfo<Metadata>` |

(Item ownership, approvals, etc. live in `pallet_uniques`.)

## Events

`CollectionCreated`, `ItemMinted`, `ItemTransferred`, `ItemBurned`, `CollectionDestroyed`.

## Errors

`NoAvailableItemId`, `NoAvailableCollectionId`, `TokenCollectionNotEmpty`, `CollectionUnknown`, `ItemUnknown`, `NotPermitted`, `IdReserved`.

## Extrinsics

| Name | Description |
|------|-------------|
| `create_collection` | Create NFT collection with `CollectionType` + metadata |
| `mint` | Mint item in collection (collection owner only) |
| `transfer` | Transfer item |
| `burn` | Burn item |
| `destroy_collection` | Destroy empty collection |

## Hooks

None.

## Integration

- **Traits implemented:** none external; delegates to `pallet_uniques`
- **Traits consumed:** `NftPermission<CollectionType>` (runtime decides which collection types allow which operations)
- **Pallets depended on:** `pallet_uniques`

## Key types

```rust
// pallets/nft/src/types.rs
pub struct CollectionInfo<CollectionType, Metadata> {
    pub collection_type: CollectionType,
    pub metadata: Metadata,
}

pub struct ItemInfo<Metadata> {
    pub metadata: Metadata,
}

pub trait NftPermission<CollectionType> {
    fn can_create(t: &CollectionType) -> bool;
    fn can_mint(t: &CollectionType) -> bool;
    fn can_transfer(t: &CollectionType) -> bool;
    fn can_burn(t: &CollectionType) -> bool;
    fn can_destroy(t: &CollectionType) -> bool;
    fn has_deposit(t: &CollectionType) -> bool;
}
```

## Gotchas

- Collection IDs `0..=ReserveCollectionIdUpTo` are reserved for runtime-registered collections (Omnipool LP, staking, LM) — user-created collections must start above this cap.
- `CollectionType` is the runtime-defined enum gating permissions — in Hydration the variants are Omnipool LP (non-transferable), staking position (non-transferable), LM deposit (non-transferable), marketplace (transferable).
- No ERC-721-style approvals — pure ownership model.
- Destroy succeeds only when collection is empty (all items burned/transferred first).
- Metadata size is bounded by `pallet_uniques::Config::StringLimit`.
- Items are bound to their collection — no cross-collection moves.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/nft-lp-positions\|nft-lp-positions]]
