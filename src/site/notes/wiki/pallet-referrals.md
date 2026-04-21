---
{"dg-publish":true,"permalink":"/wiki/pallet-referrals/","title":"pallet-referrals","tags":["referrals","rewards","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-referrals","repo":"hydration-node","paths":["pallets/referrals/src/lib.rs","pallets/referrals/src/types.rs","pallets/referrals/src/traits.rs"],"symbols":["Pallet","Config","Level","FeeDistribution","register_code","link_code","claim_rewards","set_reward_percentage","Tier","ReferralCodes","LinkedAccounts","ReferrerShares","TraderShares"],"traits_impl":["Convert"],"depends_on":["pallet-broadcast"],"runtime_index":75,"tags":["referrals","rewards","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-referrals

**TL;DR:** On-chain referral program. Users register a code, others link via `link_code`; trade fees route a portion to the referrer and back to the trader as rebate. Tiered by accumulated referred volume. Runtime index = 75.

## Role

Implements [[wiki/referrals\|referrals]]. Trade pallets (omnipool, stableswap, route-executor) call into this pallet through fee-handler adapters to split a portion of the fee into referrer / trader buckets.

## Config trait (excerpt)

```rust
// pallets/referrals/src/lib.rs
pub trait Config: frame_system::Config + pallet_broadcast::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    type Currency: GetByKey<Self::AssetId, Balance> + MultiCurrency<Self::AccountId, CurrencyId = Self::AssetId>;
    type Convert: Convert<AccountId, (AssetId, AssetId, Balance), Result<Balance, DispatchError>>;
    #[pallet::constant] type RewardAsset: Get<Self::AssetId>;
    type RegistrationFee: Get<(Self::AssetId, Balance, Self::AccountId)>;
    #[pallet::constant] type CodeLength: Get<u32>;
    #[pallet::constant] type TierVolume: Get<Balance>;
    #[pallet::constant] type MinTradingAmount: Get<Balance>;
    #[pallet::constant] type SeedNativePrice: Get<(Balance, Balance)>;
    type PriceProvider: SpotPriceProvider<Self::AssetId, Price = FixedU128>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `ReferralCodes` | StorageMap | `ReferralCode → AccountId` (owner) |
| `ReferralAccounts` | StorageMap | `AccountId → ReferralCode` |
| `LinkedAccounts` | StorageMap | `AccountId (trader) → AccountId (referrer)` |
| `ReferrerShares` | StorageMap | `AccountId → Balance` (unclaimed HDX rewards) |
| `TraderShares` | StorageMap | `AccountId → Balance` |
| `TotalShares` | StorageValue | `Balance` |
| `AssetRewards` | StorageDoubleMap | `(Level, FeeDistributionType) → FeeDistribution` |
| `CounterForReferrer` | StorageMap | `AccountId → Balance` (referred volume accumulator) |
| `ReferrerLevel` | StorageMap | `AccountId → Level` |

## Events

`CodeRegistered`, `CodeLinked`, `Converted`, `Claimed`, `LevelUp`, `RewardsClaimed`.

## Errors

`AlreadyRegistered`, `TooLong`, `TooShort`, `InvalidCharacter`, `AlreadyLinked`, `LinkNotAllowed`, `ConversionZeroAmount`, `ZeroAmount`, `ConversionLimitReached`, `ConversionMinTradingAmountNotReached`, `CodeNotFound`.

## Extrinsics

| Name | Description |
|------|-------------|
| `register_code` | Reserve a referral code (pays `RegistrationFee`) |
| `link_code` | Link account to a referrer's code (one-time, immutable) |
| `claim_rewards` | Referrer or trader claims accumulated HDX rewards |
| `set_reward_percentage` | Governance sets per-tier reward split |

## Hooks

None.

## Integration

- **Traits implemented:** `Convert<AccountId, TradeInfo, Balance>` — called by AMM pallets to register a trade
- **Traits consumed:** `SpotPriceProvider`, `Currency`, `GetByKey`
- **Pallets depended on:** [[wiki/pallet-broadcast\|pallet-broadcast]], [[wiki/pallet-asset-registry\|pallet-asset-registry]]

## Key concept: Level tiers

```rust
// pallets/referrals/src/types.rs
pub enum Level { None, Novice, Advanced, Expert, Master }
// Each level accumulates a volume threshold via `CounterForReferrer`.
// Higher levels = larger share of fees diverted into referrer/trader buckets.
```

## Gotchas

- Once linked, a trader's referrer is **immutable** — no re-linking.
- Rewards accumulate in [[wiki/hdx\|hdx]]; swapped on-the-fly via `Convert` trait from the trade's output asset.
- Trader self-rebate acts like a discount — reduces effective trading cost.
- Tier progression is one-way: referrers only level up as volume accumulates.
- Conversion min-trading-amount prevents micro-trade spam reward gaming.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/referrals\|referrals]]
