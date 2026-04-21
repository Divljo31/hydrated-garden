---
{"dg-publish":true,"permalink":"/wiki/pallet-bonds/","title":"pallet-bonds","tags":["bonds","fixed-term","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-bonds","repo":"hydration-node","paths":["pallets/bonds/src/lib.rs","pallets/bonds/src/types.rs"],"symbols":["Pallet","Config","Bond","issue","redeem","Bonds","BondIds"],"traits_impl":[],"depends_on":["pallet-asset-registry"],"runtime_index":71,"tags":["bonds","fixed-term","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-bonds

**TL;DR:** Tokenized fixed-term deposits: lock asset A, receive bond-token B representing "A at maturity block T". Bond tokens are tradable in [[wiki/pallet-lbp\|pallet-lbp]] / [[wiki/pallet-xyk\|pallet-xyk]] pools, enabling a yield-curve market. Runtime index = 71.

## Role

Primitive for fixed-term yield markets. Used to auction off future interest streams, or to let LPs lock stable tokens into time-boxed commitments.

## Config trait (excerpt)

```rust
// pallets/bonds/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    type Currency: MultiCurrency<Self::AccountId, CurrencyId = Self::AssetId, Balance = Balance>;
    type AssetRegistry: Inspect<AssetId = Self::AssetId>
        + Create<Balance, AssetId = Self::AssetId, Error = DispatchError>;
    type ExistentialDeposits: GetByKey<Self::AssetId, Balance>;
    type TimestampProvider: UnixTime;
    type PalletId: Get<PalletId>;
    type IssueOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    #[pallet::constant] type ProtocolFee: Get<Permill>;
    type FeeReceiver: Get<Self::AccountId>;
    #[pallet::constant] type BondsPalletId: Get<PalletId>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Bonds` | StorageMap | `AssetId (bond_id) → Bond { underlying_asset_id, maturity_unix }` |
| `BondIds` | StorageMap | `(AssetId, Moment) → AssetId` (underlying + maturity → bond_id) |

## Events

`TokenCreated`, `Issued`, `Redeemed`.

## Errors

`BondNotFound`, `NotMatured`, `UnderlyingAssetNotRegistered`, `InvalidMaturity`, `DisallowedAsset`.

## Extrinsics

| Name | Description |
|------|-------------|
| `issue` | Lock underlying, mint bond token with `maturity` timestamp |
| `redeem` | After maturity, burn bond token → receive underlying |

## Hooks

None.

## Integration

- **Traits implemented:** none external (pallet is referenced via `AssetType::Bond` in [[wiki/pallet-asset-registry\|pallet-asset-registry]])
- **Traits consumed:** `MultiCurrency`, `AssetRegistry` (Create), `UnixTime`
- **Pallets depended on:** [[wiki/pallet-asset-registry\|pallet-asset-registry]]

## Gotchas

- Bond assets are normal fungibles once issued — tradable on [[wiki/pallet-xyk\|pallet-xyk]] / [[wiki/pallet-lbp\|pallet-lbp]].
- `ProtocolFee` taken on `issue` (paid in underlying asset).
- Redemption locked until `maturity_unix` — on-chain wall clock (`UnixTime`).
- Each (underlying, maturity) pair is a distinct bond asset — multiple bonds for the same underlying at different maturities form an implicit yield curve.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/bonding-curve\|bonding-curve]]
