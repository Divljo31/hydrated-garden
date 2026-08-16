---
{"dg-publish":true,"permalink":"/wiki/pallet-circuit-breaker/","title":"pallet-circuit-breaker","tags":["risk","limits","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-circuit-breaker","repo":"hydration-node","paths":["pallets/circuit-breaker/src/lib.rs","pallets/circuit-breaker/src/types.rs","pallets/circuit-breaker/src/weights.rs"],"symbols":["Pallet","Config","TradeVolumeLimit","LiquidityLimit","AllowedTradeVolumeLimitPerAsset","AllowedAddLiquidityAmountPerAsset","AllowedRemoveLiquidityAmountPerAsset","set_trade_volume_limit","set_add_liquidity_limit","set_remove_liquidity_limit","do_lock_deposit","DepositLockWhitelist","InTradeContext","DepositLimitExceededForWhitelistedAccount"],"traits_impl":["OnTradeHandler","OnLiquidityChangedHandler"],"depends_on":["pallet-broadcast"],"runtime_index":65,"tags":["risk","limits","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
---


# pallet-circuit-breaker

**TL;DR:** Per-block per-asset volume and liquidity change limits. Tracks accumulated deltas per block and aborts trades/liquidity ops that would exceed configured thresholds. Runtime index = 65.

## Role

Implements [[wiki/circuit-breaker\|circuit-breaker]] guardrails for [[wiki/pallet-omnipool\|pallet-omnipool]] and [[wiki/pallet-stableswap\|pallet-stableswap]]. Protects against flash-loan drain attacks and abnormal volume spikes.

## Config trait (excerpt)

```rust
// pallets/circuit-breaker/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AssetId: Parameter + Member + Copy + MaybeSerializeDeserialize + Ord + Default + MaxEncodedLen;
    type Balance: Parameter + Member + AtLeast32BitUnsigned + Copy + Default + MaxEncodedLen;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type WhitelistedAccounts: Contains<Self::AccountId>;
    /// Accounts exempt from deposit *locking* (runtime: the router account).
    /// The exemption only applies while `InTradeContext` holds.
    type DepositLockWhitelist: Contains<Self::AccountId>;
    /// Whether the current execution is inside a trade.
    /// `DepositLockWhitelist` is honoured only while this holds, since the
    /// exemption depends on the caller rolling the deposit back on error.
    type InTradeContext: Get<bool>;
    type OmnipoolHubAsset: Get<Self::AssetId>;
    type DepositLimiter: ...;
    #[pallet::constant] type DefaultMaxNetTradeVolumeLimitPerBlock: Get<(u32, u32)>;
    #[pallet::constant] type DefaultMaxAddLiquidityLimitPerBlock: Get<Option<(u32, u32)>>;
    #[pallet::constant] type DefaultMaxRemoveLiquidityLimitPerBlock: Get<Option<(u32, u32)>>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `TradeVolumeLimitPerAsset` | StorageMap | `AssetId → (u32, u32)` (numerator, denominator ratio of liquidity) |
| `AllowedTradeVolumeLimitPerAsset` | StorageMap | `AssetId → TradeVolumeLimit` |
| `LiquidityAddLimitPerAsset` | StorageMap | `AssetId → Option<(u32, u32)>` |
| `LiquidityRemoveLimitPerAsset` | StorageMap | `AssetId → Option<(u32, u32)>` |
| `AllowedAddLiquidityAmountPerAsset` | StorageMap | `AssetId → LiquidityLimit` |
| `AllowedRemoveLiquidityAmountPerAsset` | StorageMap | `AssetId → LiquidityLimit` |

## Events

`TradeVolumeLimitChanged`, `AddLiquidityLimitChanged`, `RemoveLiquidityLimitChanged`.

## Errors

`MaxTradeVolumePerBlockReached`, `MaxAddLiquidityPerBlockReached`, `MaxRemoveLiquidityPerBlockReached`, `InvalidLimitValue`, `LiquidityLimitNotStoredForAsset`.

## Extrinsics

| Name | Description |
|------|-------------|
| `set_trade_volume_limit` | Set per-asset trade volume ratio (TechnicalOrigin) |
| `set_add_liquidity_limit` | Set per-asset add-liquidity ratio |
| `set_remove_liquidity_limit` | Set per-asset remove-liquidity ratio |

## Hooks

`on_finalize` clears all allowed-*-per-asset storage so each block starts fresh.

## Integration

- **Traits implemented:** `OnTradeHandler`, `OnLiquidityChangedHandler`, `CircuitBreaker` (consumed by omnipool/stableswap adapters)
- **Traits consumed:** `Contains<AccountId>` (whitelist)
- **Pallets depended on:** none directly; wired via adapter traits in the runtime

## Key logic: volume accumulation

```rust
// pallets/circuit-breaker/src/lib.rs
pub fn ensure_and_update_trade_volume_limit(
    asset_in: T::AssetId, amount_in: T::Balance,
    asset_out: T::AssetId, amount_out: T::Balance,
) -> Result<(), DispatchError> {
    let (in_limit, in_delta) = Self::calculate_and_store_trade_volume(asset_in, amount_in)?;
    ensure!(in_delta.abs() <= in_limit, Error::<T>::MaxTradeVolumePerBlockReached);
    // ... same for asset_out
}
```

## Gotchas

- Limits reset every block via `on_finalize` — long-lived state is only the configured ratios.
- Ratios expressed as `(numerator, denominator)` against asset's total liquidity.
- Whitelisted accounts bypass all checks (used for runtime upgrades / authority-driven rebalancing).
- **Deposit-lock whitelist is now trade-context gated.** `do_lock_deposit` errors with `DepositLimitExceededForWhitelistedAccount` (instead of locking) only when the account is in `DepositLockWhitelist` **and** `InTradeContext::get()` is true. Erroring is only safe while a trade can unwind the deposit; outside a trade the deposit is locked normally rather than rejected.

```rust
// pallets/circuit-breaker/src/lib.rs
pub(crate) fn do_lock_deposit(who: &T::AccountId, asset_id: T::AssetId, amount: T::Balance) -> DispatchResult {
    // Whitelisted accounts error instead of locking; only safe inside a trade, where the error
    // unwinds the deposit.
    if T::DepositLockWhitelist::contains(who) && T::InTradeContext::get() {
        return Err(Error::<T>::DepositLimitExceededForWhitelistedAccount.into());
    }
    // ...
}
```

- Runtime wiring (`runtime/hydradx/src/assets.rs`): `DepositLockWhitelist` = the [[wiki/pallet-route-executor\|pallet-route-executor]] router account; `InTradeContext` = `pallet_broadcast::Pallet::<Runtime>::get_swapper().is_some()`. [[wiki/pallet-route-executor\|pallet-route-executor]] moved `set_swapper` to **before** the first user-funds transfer so the whole router window is inside a trade context.
- `DepositLimitExceededForWhitelistedAccount` is in the runtime's `RetryOnErrorForDca` list — a DCA execution hitting it retries rather than terminating.
- `None` for liquidity limits means the feature is disabled for that asset (vs. a zero limit which blocks all ops).
- Only applies to assets that have been explicitly configured — unconfigured assets use defaults from Config constants.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/circuit-breaker\|circuit-breaker]]
