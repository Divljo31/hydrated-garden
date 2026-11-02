---
{"dg-publish":true,"permalink":"/wiki/pallet-ema-oracle/","title":"pallet-ema-oracle","tags":["oracle","ema","price","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-ema-oracle","repo":"hydration-node","paths":["pallets/ema-oracle/src/lib.rs","pallets/ema-oracle/src/types.rs","pallets/ema-oracle/src/migrations/mod.rs","pallets/ema-oracle/src/migrations/v3.rs","math/src/ema/math.rs"],"symbols":["Pallet","Config","Oracle","OracleEntry","Accumulator","Source","OraclePeriod","PriceOracle","RawEntry","OracleWhitelist"],"traits_impl":["OnActivityHandler","PriceOracle","AggregatedPriceOracle","RawOracle","PegRawOracle"],"depends_on":[],"runtime_index":202,"tags":["oracle","ema","price","runtime","rust","substrate"],"last_updated":"2026-11-02"}}
---


# pallet-ema-oracle

**TL;DR:** Exponential moving average oracle for price, volume, liquidity, and shares across multiple assets. Accumulates raw entries per block from AMM callbacks, then rolls them forward per configured period (runtime: `LastBlock`, `Short`, `TenMinutes`, `Day`, …). Price smoothing is **direction-independent** (storage v3): rising prices are averaged in reciprocal space so averaging commutes with inversion. Smoothing constants are calibrated for **2 s blocks**. External sources push prices through signed, feeless `set_external_oracle*` calls by authorized accounts. Runtime index = 202.

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
| `update_bifrost_oracle` (2) | Signed — Bifrost price push (kept for the Bifrost integration) |
| `set_external_oracle(source, asset_a: VersionedLocation, asset_b, price)` (6) | Signed, authorized for `(source, pair)`; `Pays::No` on success |
| `set_external_oracle_by_ids(source, asset_a: AssetId, asset_b, price)` (8) | Same, keyed by local ids (skips the location → id conversion) |
| `register_external_source(source)` (3) | AuthorityOrigin |
| `remove_external_source(source)` (5) | AuthorityOrigin |
| `add_authorized_account(source, pair, account)` (4) | AuthorityOrigin |
| `remove_authorized_account(source, pair, account)` (7) | AuthorityOrigin |

Runtime `AuthorityOrigin = Root | EconomicParameters` (referendum speed — no TC fast path to de-authorize a compromised external price pusher).

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
// Smoothing factors are hard-coded `Fraction::from_bits(..)` per period, computed from the
// period length in blocks at 2 s/block (e.g. TenMinutes ≈ 0.0066).
```

## Price averaging (storage v3)

```rust
// math/src/ema/math.rs
// Falling prices: arithmetic average. Rising prices: reciprocal of the arithmetic average
// of both reciprocals. Same (slower) response in both directions; commutes with inversion.
if prev.is_zero() || incoming.is_zero() || incoming <= prev {
    return arithmetic_price_weighted_average(prev, incoming, weight);
}
arithmetic_price_weighted_average(prev.inverted(), incoming.inverted(), weight).inverted()
```

Volume / liquidity / shares still use plain weighted averages. `migrations/v3.rs` (`InnerMigrateV2ToV3`) re-settled stored oracles for the new smoothing, batched at `MAX_ORACLE_ENTRIES_TO_MIGRATE = 768` rows.

## Gotchas

- Only whitelisted `(Source, asset_a, asset_b)` tuples produce oracle entries. Unwhitelisted activity is ignored.
- First update for an oracle entry "bootstraps" with zero-smoothing; later entries interpolate missed blocks.
- `on_finalize` weight scales with number of entries in accumulator — `MaxUniqueEntries` bounds it.
- Price is stored as `EmaPrice` (fixed-point rational) to preserve precision across periods.
- External prices (Bifrost LSTs via `update_bifrost_oracle`; any registered external source via `set_external_oracle*`) come from authorized signed accounts, not AMM activity. Treat each authorized account as an oracle dependency (see [[wiki/note-dependency-registry\|note-dependency-registry]]).
- Oracle `(A,B)` and `(B,A)` now yield mutually inverse prices after smoothing; do not compensate for direction bias in consumers.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/ema-oracle\|ema-oracle]]
