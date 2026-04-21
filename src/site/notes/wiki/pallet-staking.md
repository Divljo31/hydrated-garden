---
{"dg-publish":true,"permalink":"/wiki/pallet-staking/","title":"pallet-staking","tags":["staking","governance","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-staking","repo":"hydration-node","paths":["pallets/staking/src/lib.rs","pallets/staking/src/types.rs","pallets/staking/src/integrations.rs","pallets/staking/src/traits.rs"],"symbols":["Pallet","Config","Position","Staking","Vote","ConvictionVote","initialize_staking","stake","increase_stake","claim","unstake","Positions","PositionVotes"],"traits_impl":["DemocracyHooks","PayablePercentage"],"depends_on":["pallet-democracy","pallet-nft","pallet-referenda"],"runtime_index":69,"tags":["staking","governance","runtime","rust","substrate"],"last_updated":"2026-04-20"}}
---


# pallet-staking

**TL;DR:** HDX staking with governance-aligned rewards. Users stake HDX, receive an NFT position, vote in [[wiki/opengov\|opengov]]; voting with conviction increases reward-action-points. Rewards come from the staking pot and are distributed proportionally to aligned voting history. Runtime index = 69.

## Role

Hydration's native staking mechanism. Differs from typical parachain staking: no validation, no slashing — rewards are earned by *voting* in governance with conviction, tying stake incentives to governance participation.

## Config trait (excerpt)

```rust
// pallets/staking/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type AssetId: Parameter + Member + Copy + MaxEncodedLen;
    type Currency: MultiCurrencyExtended<Self::AccountId, CurrencyId = Self::AssetId, Balance = Balance>
        + MultiLockableCurrency<Self::AccountId>;
    #[pallet::constant] type HdxAssetId: Get<Self::AssetId>;
    type BlockNumberProvider: BlockNumberProvider<BlockNumber = BlockNumberFor<Self>>;
    #[pallet::constant] type PalletId: Get<PalletId>;
    #[pallet::constant] type MinStake: Get<Balance>;
    #[pallet::constant] type PeriodLength: Get<BlockNumberFor<Self>>;
    #[pallet::constant] type TimePointsPerPeriod: Get<u8>;
    #[pallet::constant] type TimePointsWeight: Get<Permill>;
    #[pallet::constant] type ActionPointsWeight: Get<Permill>;
    #[pallet::constant] type UnclaimablePeriods: Get<BlockNumberFor<Self>>;
    #[pallet::constant] type CurrentStakeWeight: Get<u8>;
    type PayablePercentage: SigmoidPercentage<FixedU128, Error = DispatchError>;
    type NFTCollectionId: Get<CollectionId>;
    type NFTHandler: Mutate<Self::AccountId, ItemId = Self::PositionItemId, CollectionId = CollectionId>;
    type Collections: Inspect<...>;
    type MaxVotes: Get<u32>;
    type ReferendumInfo: ReferendumInfo<Self::AccountId, Balance, BlockNumberFor<Self>>;
    type MaxPointsPerAction: Get<(u32, u8)>;
    type Vesting: VestingDetails<Self::AccountId, Balance>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Staking` | StorageValue | `StakingData` (global: total_stake, accumulated_reward_per_stake, pot) |
| `Positions` | StorageMap | `PositionItemId → Position` |
| `NextPositionId` | StorageValue | `PositionItemId` |
| `PositionVotes` | StorageMap | `PositionItemId → BoundedVec<(ReferendumIndex, Vote), MaxVotes>` |
| `ProcessedVotes` | StorageDoubleMap | `(AccountId, ReferendumIndex) → Vote` |

## Events

`PositionCreated`, `StakeAdded`, `RewardsClaimed`, `Unstaked`, `StakingInitialized`, `AccumulatedRpsUpdated`.

## Errors

`NotInitialized`, `AlreadyInitialized`, `PositionNotFound`, `InsufficientStake`, `PositionAlreadyExists`, `Forbidden`, `InconsistentState`, `InsufficientBalance`, `MaxVotesReached`, `Arithmetic`, `MissingPotBalance`.

## Extrinsics

| Name | Description |
|------|-------------|
| `initialize_staking` | One-time setup (AuthorityOrigin) |
| `stake` | Create position, lock HDX, mint NFT |
| `increase_stake` | Add more HDX to existing position |
| `claim` | Claim accumulated rewards |
| `unstake` | Close position, burn NFT, return locked HDX |

## Hooks

None directly; `DemocracyHooks` / `on_vote` / `on_remove_vote` callbacks process conviction-weighted action points.

## Integration

- **Traits implemented:** `DemocracyHooks` — consumed by [[wiki/pallet-democracy\|pallet-democracy]]
- **Traits consumed:** `ReferendumInfo`, `VestingDetails`, `NFTHandler`, `SigmoidPercentage`
- **Pallets depended on:** [[wiki/pallet-democracy\|pallet-democracy]], [[wiki/pallet-nft\|pallet-nft]], vesting

## Reward formula (high-level)

- **Points** = `TimePointsWeight * time_points + ActionPointsWeight * action_points`
- **Payable %** = sigmoid(points / max_points) — user gets only a fraction of pro-rata rewards until they accrue points
- **Action points** earned by voting with conviction in referenda; unstaking slashes them

## Gotchas

- NFT position is non-transferable (locked via NFT collection config).
- `UnclaimablePeriods` enforces a cooldown — cannot claim rewards until N periods have passed since stake.
- Rewards-per-stake (RPS) is updated lazily on every stake/claim/unstake.
- Action-point system penalizes non-voting stakers — "stake and forget" yields minimal rewards.
- Unstaking forfeits all unclaimed rewards (have to `claim` first).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hdx\|hdx]]
- [[wiki/opengov\|opengov]]
