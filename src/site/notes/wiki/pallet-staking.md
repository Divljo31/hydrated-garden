---
{"dg-publish":true,"permalink":"/wiki/pallet-staking/","title":"pallet-staking","tags":["staking","governance","gigahdx","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-staking","repo":"hydration-node","paths":["pallets/staking/src/lib.rs","pallets/staking/src/types.rs","pallets/staking/src/integrations.rs","pallets/staking/src/integrations/conviction_voting.rs","pallets/staking/src/integrations/democracy_legacy.rs","pallets/staking/src/traits.rs"],"symbols":["Pallet","Config","Position","Staking","Vote","ConvictionVote","initialize_staking","stake","increase_stake","claim","unstake","force_unstake","process_votes","pot_account_id","get_user_position_id","Positions","Votes","VotesRewarded","PositionVotes","ProcessedVotes","SixSecBlocksSince","ExternalClaims","StakingConvictionVoting"],"traits_impl":["DemocracyHooks","VotingHooks","PayablePercentage"],"depends_on":["pallet-democracy","pallet-nft","pallet-referenda","pallet-conviction-voting","pallet-gigahdx"],"runtime_index":69,"tags":["staking","governance","gigahdx","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
---


# pallet-staking

**TL;DR:** Legacy HDX staking with governance-aligned rewards. Users stake HDX, receive an NFT position, vote in [[wiki/opengov\|opengov]]; voting with conviction increases reward-action-points. Rewards come from the staking pot and are distributed proportionally to aligned voting history. Runtime index = 69. **Superseded by [[wiki/gigahdx\|gigahdx]] / [[wiki/pallet-gigahdx\|pallet-gigahdx]]** — see "GigaHDX interaction" below.

## Role

Hydration's original staking mechanism. Differs from typical parachain staking: no validation, no slashing — rewards are earned by *voting* in governance with conviction, tying stake incentives to governance participation.

Now in wind-down: its trade-fee share is 5% (vs 15% gigapot + 25% gigahdx rewards), new stakes are blocked when the account carries a `ghdxlock` lock, and `force_unstake` exists solely as the exit ramp into [[wiki/pallet-gigahdx\|pallet-gigahdx]].

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
    /// NEW: sum of HDX claimed by other pallets that must not back a legacy
    /// stake (e.g. `ghdxlock`). `()` disables the check.
    type ExternalClaims: ExternalClaims<Self::AccountId>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Staking` | StorageValue | `StakingData` (global: total_stake, accumulated_reward_per_stake, pot_reserved_balance) |
| `Positions` | StorageMap | `PositionItemId → Position` |
| `NextPositionId` | StorageValue | `PositionItemId` |
| `Votes` | StorageMap | `PositionItemId → Voting<MaxVotes>` — conviction-voting votes (getter `get_position_votes`) |
| `VotesRewarded` | StorageDoubleMap | `(AccountId, ReferendumIndex) → Vote` — processed votes, drives the lock-on-losing-vote decision (getter `processed_votes`) |
| `PositionVotes` | StorageMap | **Legacy** — democracy-era votes, kept until `pallet-democracy` is removed |
| `ProcessedVotes` | StorageDoubleMap | **Legacy** — democracy-era processed votes |
| `SixSecBlocksSince` | StorageValue | `BlockNumber` at which the chain moved to 6 s blocks (time-point math) |

## Events

`PositionCreated`, `StakeAdded`, `RewardsClaimed`, `Unstaked`, `StakingInitialized`, `AccumulatedRpsUpdated`, `ForceUnstaked` (new — GigaHDX migration; distinct from `Unstaked` because no rewards were forfeited).

## Errors

`InsufficientBalance`, `InsufficientStake`, `PositionNotFound`, `MaxVotesReached`, `NotInitialized`, `AlreadyInitialized`, `Arithmetic`, `MissingPotBalance`, `PositionAlreadyExists`, `Forbidden`, `ExistingVotes`, `ExistingProcessedVotes`, `ActiveVotesOngoing` (retained for error-index stability, **no longer emitted**), `BlockedByExternalLock` (new), `InconsistentState(InconsistentStateError)`.

## Extrinsics

| # | Name | Description |
|---|------|-------------|
| 0 | `initialize_staking` | One-time setup (AuthorityOrigin) |
| 1 | `stake` | Create position, lock HDX, mint NFT |
| 2 | `increase_stake` | Add more HDX to existing position |
| 3 | `claim` | Claim accumulated rewards |
| 4 | `unstake` | Close position, burn NFT, return locked HDX (forfeits unclaimed rewards) |

## Public helpers (not extrinsics)

| Symbol | Description |
|---|---|
| `force_unstake(who) -> Balance` | GigaHDX migration path. See below. |
| `pot_account_id()` | Staking pot; also the `HdxStakingFeeReceiver` destination |
| `get_user_position_id(who)` | Resolves the caller's NFT position id |
| `process_trade_fee(...)` | Fee-processor entry point |

## GigaHDX interaction

Three changes since the 2026-05-13 sync tie this pallet to [[wiki/gigahdx\|gigahdx]]:

**1. `Config::ExternalClaims` — strict lock-overlap policy.** `ensure_stakeable_balance` (called by `stake` and `increase_stake`) now rejects outright if any non-whitelisted lock exists:

```rust
// pallets/staking/src/lib.rs — Pallet::ensure_stakeable_balance
ensure!(T::ExternalClaims::on(who) == 0, Error::<T>::BlockedByExternalLock);
```

The runtime wires `LegacyStakingExternalClaims` (`runtime/hydradx/src/gigahdx.rs`), whitelisting only `stk_stks` (its own), `ormlvest` (already netted out via `Config::Vesting`) and `pyconvot` (governance overlap is intended). **`ghdxlock` is not whitelisted**, so a gigahdx staker cannot also open a legacy position — the mirror image of `HdxExternalClaims` on the gigahdx side.

**2. `force_unstake(who)` — the migration exit ramp.** Pallet-internal (`pub`, no origin check, no weight — the calling `pallet_gigahdx::migrate` extrinsic owns both), `#[transactional]`:

```rust
// pallets/staking/src/lib.rs
pub fn force_unstake(who: &T::AccountId) -> Result<Balance, DispatchError> {
    ensure!(Self::is_initialized(), Error::<T>::NotInitialized);
    let position_id = Self::get_user_position_id(who)?.ok_or(...PositionNotFound)?;
    let voting = Votes::<T>::get(position_id);
    ensure!(
        voting.votes.is_empty() && !VotesRewarded::<T>::contains_prefix(who),
        Error::<T>::ExistingVotes
    );
    // ... pays 100% of rewards, burns the NFT, removes STAKING_LOCK_ID,
    //     drains Votes / VotesRewarded, emits ForceUnstaked
    // returns stake + accumulated_locked_rewards + paid_rewards
}
```

Differences from `unstake`: **no sigmoid `PayablePercentage` slash and no `UnclaimablePeriods` early-exit penalty** — 100% of accumulated rewards are paid so migrating users lose nothing. Whole position only; no partial migration.

The `ExistingVotes` precondition (any registered vote, finished *or* ongoing) is load-bearing: votes must be removed while the legacy position still backs them, so conviction-voting applies the conviction lock to winning **and** losing votes (via the legacy `lock_balance_on_unsuccessful_vote` hook). Letting finished-referendum votes survive migration would strand those losing-vote locks — post-migration neither the legacy hook (position burned) nor the [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] hook (never tracked legacy-era votes) owns them.

Wired as `pallet_gigahdx::traits::LegacyStakeMigrator` via `LegacyStakingMigrator` in `runtime/hydradx/src/gigahdx.rs`.

**3. `process_votes` removed from `on_before_vote`.** `pallets/staking/src/integrations/conviction_voting.rs` no longer calls `Pallet::process_votes` on the vote path; it only records the capped `staking_vote` into `Votes`. Points for finished referenda are settled elsewhere (`claim` / `increase_stake` / `unstake` paths).

## Hooks

- `VotingHooks` via `StakingConvictionVoting<T>` (`pallets/staking/src/integrations/conviction_voting.rs`). In the runtime this is the *first* half of `CombinedVotingHooks<StakingConvictionVoting<Runtime>, pallet_gigahdx_rewards::voting_hooks::VotingHooksImpl<Runtime>>` (`runtime/hydradx/src/governance/mod.rs`) — both consumers see every vote.
- `on_before_vote` caps the recorded vote at `min(amount, position.stake, free_balance − vested − accumulated_locked_rewards)` so overlapping locks don't earn points twice.
- Legacy `DemocracyHooks` retained in `pallets/staking/src/integrations/democracy_legacy.rs`.

## Integration

- **Traits implemented:** `VotingHooks` (conviction voting), `DemocracyHooks` (legacy, [[wiki/pallet-democracy\|pallet-democracy]])
- **Traits consumed:** `ReferendumInfo`/`GetReferendumState`, `VestingDetails`, `ExternalClaims`, `NFTHandler`, `SigmoidPercentage`
- **Called by:** [[wiki/pallet-gigahdx\|pallet-gigahdx]] `migrate()` → `force_unstake`
- **Pallets depended on:** [[wiki/pallet-democracy\|pallet-democracy]], [[wiki/pallet-nft\|pallet-nft]], `pallet-conviction-voting`, vesting

## Reward formula (high-level)

- **Points** = `TimePointsWeight * time_points + ActionPointsWeight * action_points`
- **Payable %** = sigmoid(points / max_points) — user gets only a fraction of pro-rata rewards until they accrue points
- **Action points** earned by voting with conviction in referenda; unstaking slashes them

## Gotchas

- NFT position is non-transferable (locked via NFT collection config).
- `UnclaimablePeriods` enforces a cooldown — cannot claim rewards until N periods have passed since stake.
- Rewards-per-stake (RPS) is updated lazily on every stake/claim/unstake.
- Action-point system penalizes non-voting stakers — "stake and forget" yields minimal rewards.
- Unstaking forfeits all unclaimed rewards (have to `claim` first) — but `force_unstake` (migration only) does not.
- `ActiveVotesOngoing` is dead API surface kept only to preserve error indices; migration refuses with `ExistingVotes` instead.
- `Votes` / `VotesRewarded` are the live storage items; `PositionVotes` / `ProcessedVotes` are democracy-era leftovers with confusingly similar names.
- A `ghdxlock` lock blocks `stake` and `increase_stake` — users must fully exit gigahdx (including pending unstake positions) before legacy-staking again.
- Behavioural coverage: `pallets/staking/src/tests/force_unstake.rs`; end-to-end migration in `integration-tests/src/gigahdx.rs`.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hdx\|hdx]]
- [[wiki/opengov\|opengov]]
- [[wiki/gigahdx\|gigahdx]]
- [[wiki/pallet-gigahdx\|pallet-gigahdx]]
- [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]
