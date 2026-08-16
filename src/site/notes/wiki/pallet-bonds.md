---
{"dg-publish":true,"permalink":"/wiki/pallet-bonds/","title":"pallet-bonds","tags":["bonds","fixed-term","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-bonds","repo":"hydration-node","paths":["pallets/bonds/src/lib.rs"],"symbols":["Pallet","Config","issue","redeem","Bonds","BondIds","IssueOrigin","IssuerAccount","AssetTypeWhitelist","TokenCreated","Issued","Redeemed"],"traits_impl":[],"depends_on":["pallet-asset-registry"],"runtime_index":71,"tags":["bonds","fixed-term","runtime","rust","substrate"],"last_updated":"2026-05-14"}}
---


# pallet-bonds

**TL;DR:** Tokenized fixed-term deposits: lock asset A, receive bond-token B representing "A at maturity timestamp T". Bond tokens are tradable in [[wiki/pallet-lbp\|pallet-lbp]] / [[wiki/pallet-xyk\|pallet-xyk]] pools, enabling a yield-curve market. Runtime index = 71. **As of spec 419, `issue` is restricted to `Root` or the `Treasurer` OpenGov track** (breaking change — was previously a broader admin path).

## Role

Primitive for fixed-term yield markets. Used to auction off future interest streams, or to let LPs lock stable tokens into time-boxed commitments.

## Config trait (excerpt)

```rust
// pallets/bonds/src/lib.rs
pub trait Config: frame_system::Config {
    type Balance: Parameter + AtLeast32BitUnsigned + MaxEncodedLen + From<u128>;
    type Currency: MultiCurrency<Self::AccountId, CurrencyId = AssetId, Balance = Self::Balance>;
    type AssetRegistry: Inspect<AssetId = AssetId>
        + Create<Self::Balance, Error = DispatchError>;
    type ExistentialDeposits: GetByKey<AssetId, Self::Balance>;
    type TimestampProvider: Time<Moment = Moment>;
    #[pallet::constant] type PalletId: Get<PalletId>;
    /// Origin permitted to issue new bonds, in addition to Root.
    type IssueOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    /// Account that pays underlying and receives bonds on `issue`.
    #[pallet::constant] type IssuerAccount: Get<Self::AccountId>;
    /// Asset types permitted as underlying.
    type AssetTypeWhitelist: Contains<AssetKind>;
    type WeightInfo: WeightInfo;
}
```

Runtime wiring (`runtime/hydradx/src/assets.rs:1595`):

```rust
type IssueOrigin = EitherOf<EnsureRoot<Self::AccountId>, Treasurer>;
```

`Treasurer` is the OpenGov track at id 11 (see [[wiki/opengov\|opengov]] and `governance/tracks.rs`). No `ProtocolFee`, `FeeReceiver`, or separate `BondsPalletId` exist on the current Config — those fields documented previously have been removed.

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Bonds` | StorageMap | `AssetId (bond_id) → (underlying_asset_id, maturity: Moment)` |
| `BondIds` | StorageMap | `(AssetId, Moment) → AssetId` (underlying + maturity → bond_id) |

## Events

`TokenCreated { issuer, asset_id, bond_id, maturity }`, `Issued { issuer, bond_id, amount, fee }`, `Redeemed { who, bond_id, amount }`.

## Errors

`NotRegistered`, `NotMature`, `InvalidMaturity`, `DisallowedAsset`, `AssetNotFound`, `InvalidBondName`, `FailToParseName`.

## Extrinsics

| call_index | Name | Origin | Description |
|---|---|---|---|
| 0 | `issue` | `IssueOrigin` (Root / Treasurer) | Lock underlying from `IssuerAccount`, mint bond tokens 1:1 to `IssuerAccount` |
| 1 | `redeem` | Signed | After `maturity`, burn bond tokens, receive underlying 1:1 |

## Hooks

None.

## Integration

- **Traits implemented:** none external (pallet is referenced via `AssetType::Bond` in [[wiki/pallet-asset-registry\|pallet-asset-registry]])
- **Traits consumed:** `MultiCurrency`, `AssetRegistry` (`Inspect + Create`), `Time` (timestamp), `Contains<AssetKind>` (`AssetTypeWhitelist`)
- **Pallets depended on:** [[wiki/pallet-asset-registry\|pallet-asset-registry]]

## Gotchas

- **Issuance restricted to Treasurer / Root.** Spec 419 (`feat(bonds)!`) tightened `IssueOrigin` to `EitherOf<EnsureRoot, Treasurer>`. The old `GeneralAdmin` path no longer works. Bond issuance now requires an OpenGov referendum on the Treasurer track.
- `issue` always debits underlying from `T::IssuerAccount` and credits bonds back to the *same* account — the dispatching origin is just the authoriser, never the source/sink.
- **No `ProtocolFee` field.** The current Config has no fee config (the previous documented `ProtocolFee`/`FeeReceiver` is gone). `Issued.fee` event field is always `0`.
- Bond assets are normal fungibles once issued — tradable on [[wiki/pallet-xyk\|pallet-xyk]] / [[wiki/pallet-lbp\|pallet-lbp]].
- Redemption locked until `maturity` (`Moment`, unix-time-ms) — on-chain wall clock via `T::TimestampProvider`.
- Each (underlying, maturity) pair is a distinct bond asset — multiple bonds for the same underlying at different maturities form an implicit yield curve. Re-issuing an already-registered (underlying, maturity) tuple mints additional supply of the existing bond, even if it is already mature.
- Bond assets are registered via `AssetRegistry::register_insufficient_asset(AssetKind::Bond, …)` with the underlying's existential deposit.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/bonding-curve\|bonding-curve]]
