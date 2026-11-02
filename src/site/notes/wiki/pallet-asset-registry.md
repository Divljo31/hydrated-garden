---
{"dg-publish":true,"permalink":"/wiki/pallet-asset-registry/","title":"pallet-asset-registry","tags":["registry","assets","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-asset-registry","repo":"hydration-node","paths":["pallets/asset-registry/src/lib.rs","pallets/asset-registry/src/types.rs"],"symbols":["Pallet","Config","AssetDetails","AssetType","register","update","ban_asset","unban_asset","Assets","NextAssetId","AssetIds","AssetLocations","LocationAssets","BannedAssets","RegisterAssetHook"],"traits_impl":["Inspect","Create","Mutate","RegistryInspect"],"depends_on":[],"runtime_index":51,"tags":["registry","assets","runtime","rust","substrate"],"last_updated":"2026-11-02"}}
---


# pallet-asset-registry

**TL;DR:** Canonical asset metadata registry. Maps `AssetId` (u32) → `AssetDetails` (name, symbol, decimals, type, existential deposit, XCM location). Also manages Ethereum-origin-asset mappings and a banned-assets set. Runtime index = 51.

## Role

Single source of truth for every registered asset on Hydration. Consumed by essentially every pallet that needs to look up asset metadata, decimals, or XCM location. Writes require governance / authority origin.

## Config trait (excerpt)

```rust
// pallets/asset-registry/src/lib.rs
pub trait Config: frame_system::Config {
    type RegistryOrigin: EnsureOrigin<Self::RuntimeOrigin>;   // runtime: Root | GeneralAdmin
    type UpdateOrigin: EnsureOrigin<Self::RuntimeOrigin>;     // runtime: Root | TechCommitteeMajority | GeneralAdmin
    type AssetId: Parameter /* + … */;
    type AssetNativeLocation: Parameter + Member + Default + MaxEncodedLen;
    type Currency: FungiblesInspect<Self::AccountId, AssetId = Self::AssetId, Balance = Balance>;
    type SequentialIdStartAt: Get<Self::AssetId>;
    type StringLimit: Get<u32> + Debug + PartialEq;
    type MinStringLimit: Get<u32> + Debug + PartialEq;
    type RegisterAssetHook: RegisterAssetHook<Self::AssetId>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Assets` | StorageMap | `AssetId → AssetDetails { name, asset_type, existential_deposit, symbol, decimals, xcm_rate_limit, is_sufficient }` |
| `NextAssetId` | StorageValue | `AssetId` |
| `AssetIds` | StorageMap | `BoundedVec<u8, StringLimit> → AssetId` (name lookup) |
| `AssetLocations` | StorageMap | `AssetId → AssetNativeLocation` (XCM location) |
| `LocationAssets` | StorageMap | `AssetNativeLocation → AssetId` |
| `BannedAssets` | StorageMap | `AssetId → ()` — transfers banned |

## Events

`Registered`, `Updated`, `LocationSet`, `AssetBanned`, `AssetUnbanned`.

## Errors

`NoIdAvailable`, `AssetNotFound`, `TooShort`, `InvalidSymbol`, `AssetNotRegistered`, `AssetAlreadyRegistered`, `InvalidSharedAssetLen`, `CannotUpdateLocation`, `NotInReservedRange`, `LocationAlreadyRegistered`, `Forbidden`, `InsufficientBalance`, `ForbiddenSufficiencyChange`, `AssetAlreadyBanned`, `AssetNotBanned`.

## Extrinsics

| # | Name | Origin | Description |
|---|------|--------|-------------|
| 0 | `register` | RegistryOrigin | Register asset: name, type, ED, symbol, decimals, location, `xcm_rate_limit`, `is_sufficient` |
| 1 | `update` | Registry, or UpdateOrigin for the non-restricted fields | Update details (incl. `xcm_rate_limit`, which drives the [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] deposit fuse) |
| 5 | `ban_asset` | UpdateOrigin | Ban transfers of an asset |
| 6 | `unban_asset` | UpdateOrigin | Lift the ban |

There is **no permissionless registration**: external assets are registered only through `register` by governance (call index 4 is retired). Metadata and location are set through `register` / `update` — there are no separate `set_metadata` / `set_location` calls.

## Hooks

None (stateless registry; no lifecycle hooks).

## Integration

- **Traits implemented:** `RegistryInspect`, `Inspect`, `Create<Balance>`, `Mutate`, `AssetKind`
- **Traits consumed:** `MultiCurrency`
- **Pallets depended on:** none directly

## AssetType variants

```rust
// pallets/asset-registry/src/types.rs
pub enum AssetType {
    Token,       // native-ish (e.g. DOT)
    XYK,         // xyk share token
    StableSwap,  // stableswap share token
    Bond,        // pallet-bonds share
    External,    // external (XCM) origin
    Erc20,       // EVM-registered ERC20
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
