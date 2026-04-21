---
{"dg-publish":true,"permalink":"/wiki/pallet-asset-registry/","title":"pallet-asset-registry","tags":["registry","assets","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-asset-registry","repo":"hydration-node","paths":["pallets/asset-registry/src/lib.rs","pallets/asset-registry/src/types.rs","pallets/asset-registry/src/traits.rs"],"symbols":["Pallet","Config","AssetDetails","AssetType","AssetMetadata","AssetLocation","register","update","set_metadata","set_location","Assets","AssetIds","AssetMetadataMap","AssetLocations","BannedAssets"],"traits_impl":["Inspect","Create","Mutate","RegistryInspect"],"depends_on":[],"runtime_index":51,"tags":["registry","assets","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-asset-registry

**TL;DR:** Canonical asset metadata registry. Maps `AssetId` (u32) → `AssetDetails` (name, symbol, decimals, type, existential deposit, XCM location). Also manages Ethereum-origin-asset mappings and a banned-assets set. Runtime index = 51.

## Role

Single source of truth for every registered asset on Hydration. Consumed by essentially every pallet that needs to look up asset metadata, decimals, or XCM location. Writes require governance / authority origin.

## Config trait (excerpt)

```rust
// pallets/asset-registry/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type RegistryOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type UpdateOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type Currency: MultiCurrency<Self::AccountId, CurrencyId = Self::AssetId, Balance = Self::Balance>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen + Default + AtLeast32BitUnsigned + From<u32> + Into<u32>;
    type Balance: Parameter + Member + AtLeast32BitUnsigned + Default + Copy + MaxEncodedLen;
    type AssetNativeLocation: Parameter + Member + Default + MaxEncodedLen;
    #[pallet::constant] type SequentialIdStartAt: Get<Self::AssetId>;
    #[pallet::constant] type StringLimit: Get<u32>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Assets` | StorageMap | `AssetId → AssetDetails` |
| `AssetIds` | StorageMap | `BoundedVec<u8, StringLimit> → AssetId` (name lookup) |
| `AssetMetadataMap` | StorageMap | `AssetId → AssetMetadata` |
| `AssetLocations` | StorageMap | `AssetId → AssetNativeLocation` (XCM MultiLocation) |
| `LocationAssets` | StorageMap | `AssetNativeLocation → AssetId` |
| `BannedAssets` | StorageMap | `AssetId → ()` |
| `NextAssetId` | StorageValue | `AssetId` |

## Events

`Registered`, `Updated`, `MetadataSet`, `LocationSet`, `AssetBanned`, `AssetUnbanned`.

## Errors

`AssetNotFound`, `AssetAlreadyRegistered`, `TooLong`, `AssetNotRegistered`, `LocationAlreadyRegistered`, `NoIdAvailable`, `InvalidSharedAssetItsComponents`, `Forbidden`.

## Extrinsics

| Name | Description |
|------|-------------|
| `register` | Register new asset with metadata + type + location (RegistryOrigin) |
| `update` | Update existing asset details (UpdateOrigin) |
| `set_metadata` | Change symbol/decimals (UpdateOrigin) |
| `set_location` | Set/update XCM location (UpdateOrigin) |
| `ban_asset` | Add to banned set (RegistryOrigin) |
| `unban_asset` | Remove from banned set (RegistryOrigin) |

## Hooks

None (stateless registry; no lifecycle hooks).

## Integration

- **Traits implemented:** `RegistryInspect`, `Inspect`, `Create<Balance>`, `Mutate`, `AssetKind`
- **Traits consumed:** `MultiCurrency`
- **Pallets depended on:** none directly

## AssetType variants

```rust
// pallets/asset-registry/src/types.rs
pub enum AssetType<AssetId> {
    Token,                                 // native-ish (e.g. DOT)
    XYK,                                   // xyk share token
    StableSwap,                            // stableswap share token
    Bond,                                  // pallet-bonds share
    External,                              // external (XCM) origin
    Erc20,                                  // EVM-registered ERC20
    PoolShare(AssetId, AssetId),           // (unused legacy)
}
```

## Gotchas

- `AssetId` is `u32`; 0–999 reserved for core (HDX=0, LRNA=1, USDT=10, WETH=20, etc.), 1000+ sequential.
- `SequentialIdStartAt` controls auto-ID lane for new registrations.
- Banned assets cannot be transferred, traded, or added to pools — checked via `AssetInspection`.
- `AssetLocations` is bidirectional (AssetId ↔ MultiLocation) — required by XCM adapters.
- ERC20 assets: `AssetType::Erc20` with `AssetDetails::existential_deposit` and metadata synthesized from on-chain ERC20 contract via [[wiki/pallet-evm-accounts\|pallet-evm-accounts]]' binding helper.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
