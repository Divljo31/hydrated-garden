---
{"dg-publish":true,"permalink":"/wiki/pallet-dynamic-fees/","title":"pallet-dynamic-fees","tags":["fees","dynamic","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-dynamic-fees","repo":"hydration-node","paths":["pallets/dynamic-fees/src/lib.rs","pallets/dynamic-fees/src/types.rs","pallets/dynamic-fees/src/traits.rs"],"symbols":["Pallet","Config","FeeEntry","AssetFee","GetDynamicFee","UpdateAndRetrieveFees"],"traits_impl":["GetDynamicFee"],"depends_on":["pallet-ema-oracle"],"runtime_index":68,"tags":["fees","dynamic","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-dynamic-fees

**TL;DR:** Per-asset dynamic asset-fee and protocol-fee calculator for [[wiki/pallet-omnipool\|pallet-omnipool]]. Adjusts fees each block based on net-volume/liquidity delta vs. EMA, bounded by min/max config. Runtime index = 68.

## Role

Implements [[wiki/dynamic-fees\|dynamic-fees]]. Called by [[wiki/pallet-omnipool\|pallet-omnipool]] through the `GetDynamicFee` trait on every trade. Reduces fee during normal flow, increases it during volatile / directional trading to protect LPs.

## Config trait (excerpt)

```rust
// pallets/dynamic-fees/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type BlockNumberProvider: BlockNumberProvider<BlockNumber = BlockNumberFor<Self>>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    type Fee: Parameter + Member + Default + Copy;
    type RawOracle: RawEntry<Self::AssetId, Balance>;
    type OracleSource: Get<Source>;
    type OraclePeriod: Get<OraclePeriod>;
    #[pallet::constant] type AssetFeeParameters: Get<FeeParams<Permill>>;
    #[pallet::constant] type ProtocolFeeParameters: Get<FeeParams<Permill>>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `AssetFee` | StorageMap | `AssetId → FeeEntry<Permill, BlockNumber>` |
| `ProtocolFee` | StorageMap | `AssetId → FeeEntry<Permill, BlockNumber>` |

## Events

None (fee computation is called directly through trait).

## Errors

None (pure computation; falls back to min/max on bounds overflow).

## Extrinsics

None — all fee updates happen implicitly through `GetDynamicFee::get_and_store`.

## Hooks

`on_initialize` returns constant weight; no storage mutation. All updates are lazy on first fee query per block.

## Integration

- **Traits implemented:** `GetDynamicFee<(AssetId, Balance), Fee = (Permill, Permill)>` — omnipool calls this with (asset_id, post-trade reserve).
- **Traits consumed:** `RawOracle` ([[wiki/pallet-ema-oracle\|pallet-ema-oracle]]), `BlockNumberProvider`.
- **Pallets depended on:** [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] (volume/liquidity EMAs).

## Key computation

```rust
// pallets/dynamic-fees/src/lib.rs — simplified
pub fn compute_dynamic_fee(
    current_fee: Permill, last_update: BlockNumber, block: BlockNumber,
    volume_ratio: FixedI128, liquidity_ratio: FixedI128, params: FeeParams<Permill>,
) -> Permill {
    let blocks_elapsed = block - last_update;
    let decay = params.decay.saturating_mul(blocks_elapsed);
    let amplification = params.amplification.saturating_mul(volume_ratio.abs());
    let new = current_fee.saturating_add(amplification).saturating_sub(decay);
    new.clamp(params.min_fee, params.max_fee)
}
```

## Gotchas

- Fee storage key is the *asset* (not a pool) — omnipool feeds (asset_id, asset_reserve_after_trade).
- `FeeParams` contains `amplification`, `decay`, `min_fee`, `max_fee` — tuned per-asset via runtime constants.
- Two independent fees per asset: asset-fee (charged at trade-out) and protocol-fee (charged at hub-asset side). Both use the same algorithm, different parameter sets.
- Bounded EMA ratio clamping prevents runaway fee spikes from oracle glitches.
- Idempotent: multiple calls in the same block return the same value (updates only once).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/dynamic-fees\|dynamic-fees]]
