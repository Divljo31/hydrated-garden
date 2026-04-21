---
{"dg-publish":true,"permalink":"/wiki/pallet-ema-oracle/","title":"pallet-ema-oracle","tags":["oracle","ema","price","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-ema-oracle","repo":"hydration-node","paths":["pallets/ema-oracle/src/lib.rs","pallets/ema-oracle/src/types.rs","pallets/ema-oracle/src/migrations/mod.rs"],"symbols":["Pallet","Config","Oracle","OracleEntry","Accumulator","Source","OraclePeriod","PriceOracle","RawEntry","OracleWhitelist"],"traits_impl":["OnActivityHandler","PriceOracle","AggregatedPriceOracle","RawOracle","PegRawOracle"],"depends_on":[],"runtime_index":202,"tags":["oracle","ema","price","runtime","rust","substrate"],"last_updated":"2026-04-20"}}
---


# pallet-ema-oracle

**TL;DR:** Exponential moving average oracle for price, volume, liquidity, and shares across multiple assets. Accumulates raw entries per block from AMM callbacks, then rolls them forward through three periods (Short/TenMinutes/Hour/Day/Week). Runtime index = 202.

## Role

Primary on-chain oracle consumed by [[wiki/pallet-dynamic-fees\|pallet-dynamic-fees]], [[wiki/pallet-omnipool\|pallet-omnipool]] (price barrier), [[wiki/pallet-stableswap\|pallet-stableswap]] (peg oracle), [[wiki/pallet-route-executor\|pallet-route-executor]] (route validation), [[wiki/pallet-hsm\|pallet-hsm]] (peg enforcement). Populated via AMM hooks.

## Config trait (excerpt)

```rust
// pallets/ema-oracle/src/lib.rs
pub trait Config: frame_system::Config {
    type WeightInfo: WeightInfo;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type BlockNumberProvider: BlockNumberProvider<BlockNumber = BlockNumberFor<Self>>;
    type SupportedPeriods: Get<BoundedVec<OraclePeriod, ConstU32<MAX_PERIODS>>>;
    type OracleWhitelist: Contains<(Source, AssetId, AssetId)>;
    type InternalSources: Contains<Source>;
    type LocationToAssetIdConversion: Convert<polkadot_xcm::VersionedLocation, Option<AssetId>>;
    #[pallet::constant] type MaxUniqueEntries: Get<u32>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Accumulator` | StorageValue | `BTreeMap<(Source, AssetId, AssetId), OracleEntry>` |
| `Oracles` | StorageNMap | `(Source, AssetId, AssetId, OraclePeriod) → (OracleEntry, BlockNumber)` |
| `WhitelistedAssets` | StorageValue | `BoundedBTreeSet<(Source, (AssetId, AssetId)), MaxUniqueEntries>` |
| `ExternalSources` | StorageMap | `Source → ()` |
| `AuthorizedAccounts` | StorageNMap | `(Source, (AssetId, AssetId), AccountId) → ()` |

## Events

`AddedToWhitelist`, `RemovedFromWhitelist`, `OracleUpdated`, `ExternalSourceRegistered`, `ExternalSourceRemoved`, `AuthorizedAccountAdded`, `AuthorizedAccountRemoved`.

## Errors

`TooManyUniqueEntries`, `OnTradeValueZero`, `OracleNotFound`, `AssetNotFound`, `SourceAlreadyRegistered`, `SourceNotFound`, `NotAuthorized`, `PriceIsZero`.

## Extrinsics

| Name | Description |
|------|-------------|
| `add_oracle` | Whitelist asset for oracle tracking (AuthorityOrigin) |
| `remove_oracle` | Remove asset from whitelist (AuthorityOrigin) |
| `update_bifrost_oracle` | Deprecated: Update external Bifrost oracle price (legacy compat) |
| `set_external_oracle` | Update external oracle entry for authorized (source, pair) |
| `register_source` | Register external oracle source (AuthorityOrigin) |
| `remove_source` | Remove external oracle source (AuthorityOrigin) |
| `authorize_account` | Authorize account for (source, pair) updates (AuthorityOrigin) |
| `remove_authorization` | Remove account authorization (AuthorityOrigin) |

## Hooks

- `on_initialize`: O(1) — reserves weight for `on_finalize`.
- `on_finalize`: Drains `Accumulator`, updates EMAs for every `SupportedPeriods` period, stores to `Oracles`.

## Integration

- **Traits implemented:** `OnActivityHandler` (for AMMs to call `on_liquidity_changed`/`on_trade`), `PriceOracle`, `AggregatedPriceOracle`, `RawOracle`, `PegRawOracle`
- **Traits consumed:** `BifrostOracle` (external liquid-staking price feed)
- **Pallets depended on:** none; called by omnipool/stableswap/xyk/lbp via hooks

## Key periods

```rust
// pallets/ema-oracle/src/types.rs
pub enum OraclePeriod {
    LastBlock, Short, TenMinutes, Hour, Day, Week,
}
// Smoothing factors computed from period length in blocks:
// EMA_new = (entry * alpha) + (EMA_old * (1 - alpha))
```

## Gotchas

- Only whitelisted `(Source, asset_a, asset_b)` tuples produce oracle entries. Unwhitelisted activity is ignored.
- First update for an oracle entry "bootstraps" with zero-smoothing; later entries interpolate missed blocks.
- `on_finalize` weight scales with number of entries in accumulator — `MaxUniqueEntries` bounds it.
- Price is stored as `EmaPrice` (fixed-point rational) to preserve precision across periods.
- `BifrostOracle` used specifically for vDOT/vETH (liquid staking) prices; other assets come from on-chain AMM activity.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/ema-oracle\|ema-oracle]]
