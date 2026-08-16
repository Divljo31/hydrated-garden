---
{"dg-publish":true,"permalink":"/wiki/pallet-gigahdx-rewards/","title":"pallet-gigahdx-rewards","tags":["gigahdx","rewards","governance","opengov","conviction-voting","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-gigahdx-rewards","repo":"hydration-node","paths":["pallets/gigahdx-rewards/src/lib.rs","pallets/gigahdx-rewards/src/types.rs","pallets/gigahdx-rewards/src/voting_hooks.rs","pallets/gigahdx-rewards/src/traits.rs","pallets/gigahdx-rewards/src/weights.rs","runtime/hydradx/src/gigahdx.rs","runtime/hydradx/src/governance/mod.rs"],"symbols":["Pallet","Config","claim_rewards","ReferendaTotalWeightedVotes","ReferendumTracks","ReferendaRewardPool","UserVoteRecords","UserVoteCount","PendingRewards","ReferendaReward","ReferendumLiveTally","UserVoteRecord","conviction_reward_multiplier","REWARD_MULTIPLIER_SCALE","VotingHooksImpl","maybe_allocate_and_record","record_user_reward","weighted","reward_accumulator_pot","allocated_rewards_pot"],"traits_impl":["VotingHooks","VotingCommitmentInspect"],"depends_on":["pallet-gigahdx","pallet-conviction-voting","pallet-referenda"],"runtime_index":87,"tags":["gigahdx","rewards","governance","opengov","conviction-voting","runtime","rust","substrate"],"last_updated":"2026-08-15"}}
---


# pallet-gigahdx-rewards

**TL;DR:** Distributes HDX rewards to [[wiki/pallet-gigahdx\|pallet-gigahdx]] stakers in proportion to their **conviction-voting activity**. Two pots (accumulator, fed with 25% of trade fees; allocated). On referendum completion a track-specific `Permill` of the accumulator is allocated and split pro-rata by `staked_vote × conviction_multiplier`. `claim_rewards` compounds the payout straight back into the caller's gigahdx position. Also the source of truth for the `giga_unstake` voting-commitment freeze. Runtime index = 87.

## Role

- Replaces the legacy [[wiki/pallet-staking\|pallet-staking]] action-point/sigmoid reward model with a per-referendum pro-rata split. Rewards are earned by **voting with conviction**, not by holding.
- Its `UserVoteRecords` map is a shared substrate: reward weighting, the [[wiki/pallet-gigahdx\|pallet-gigahdx]] unstake freeze, and the [[wiki/pallet-liquidation\|pallet-liquidation]] vote-clearance loop all read it.
- Wired as one half of `CombinedVotingHooks` on `pallet_conviction_voting::Config::VotingHooks` (the other half is legacy staking's `StakingConvictionVoting`) — see `runtime/hydradx/src/governance/mod.rs`.

## Config trait (excerpt)

```rust
// pallets/gigahdx-rewards/src/lib.rs
#[pallet::config]
pub trait Config: frame_system::Config<RuntimeEvent: From<Event<Self>>> + pallet_gigahdx::Config {
    type TrackId: Parameter + Member + Copy + MaybeSerializeDeserialize + Debug
        + MaxEncodedLen + TypeInfo + Ord + HasCompact;
    /// Lookup of referendum → track id.
    type Referenda: ReferendaTrackInspect<ReferendumIndex, Self::TrackId>;
    /// Track id → reward percentage table.
    type TrackRewardConfig: TrackRewardTable<Self::TrackId>;
    /// PalletId of the externally-funded accumulator pot; the allocated pot
    /// is a deterministic sub-account of it.
    #[pallet::constant] type RewardPotPalletId: Get<PalletId>;
    type WeightInfo: WeightInfo;
}
```

Note it is a **tightly coupled** pallet: `Config: pallet_gigahdx::Config`, so it reuses `NativeCurrency`, `ExternalClaims` and `MoneyMarket` from the gigahdx config.

## Pots

| Pot | Account | Funded by | Drained by |
|---|---|---|---|
| Accumulator | `RewardPotPalletId::into_account_truncating()` → `reward_accumulator_pot()` | `GigaHdxRewardsFeeReceiver` — **25% of trade fees** (`runtime/hydradx/src/assets.rs`) | per-referendum allocation |
| Allocated | `RewardPotPalletId::into_sub_account_truncating(*b"alc")` → `allocated_rewards_pot()` | allocation transfer | `claim_rewards`; leftover dust recycled back to the accumulator |

Runtime constant: `GigaRewardPotPalletId = PalletId(*b"gigarwd!")`.

## Reward multiplier table

```rust
// pallets/gigahdx-rewards/src/types.rs
pub const REWARD_MULTIPLIER_SCALE: u128 = 100;
pub fn conviction_reward_multiplier(conviction: Conviction) -> u128 {
    match conviction {
        Conviction::None     => 0,     // 0×   — no lock, no reward
        Conviction::Locked1x => 25,    // 0.25×  (7 d)
        Conviction::Locked2x => 50,    // 0.5×   (14 d)
        Conviction::Locked3x => 100,   // 1×  base (28 d)
        Conviction::Locked4x => 200,   // 2×   (56 d)
        Conviction::Locked5x => 400,   // 4×   (112 d)
        Conviction::Locked6x => 800,   // 8×   (224 d)
    }
}
```

`weighted = staked_vote × mult / 100`, where `staked_vote = min(vote.balance(), Stakes[who].hdx)`.

## Per-track allocation percentage

`TrackRewardConfig` in `runtime/hydradx/src/gigahdx.rs` (tracks defined in `runtime/hydradx/src/governance/tracks.rs`):

| Track id | Track | % of accumulator |
|---|---|---|
| 0 | root | 10% |
| 1 | whitelisted_caller | 8% |
| 5 | treasurer | 5% |
| 9 | economic_parameters | 5% |
| _other_ | — | 2% (default) |

## Key types

```rust
// pallets/gigahdx-rewards/src/types.rs
pub struct ReferendumLiveTally { pub total_weighted: u128, pub voters_count: u32 }

pub struct ReferendaReward<TrackId> {
    pub track_id: TrackId,
    pub total_reward: Balance,        // allocation snapshot
    pub total_weighted_votes: u128,   // frozen pro-rata denominator
    pub voters_remaining: u32,        // countdown; 0 → recycle dust, delete pool
    pub remaining_reward: Balance,
}

pub struct UserVoteRecord {
    pub staked_vote_amount: Balance,  // min(vote.balance(), Stakes[who].hdx) at cast time
    pub conviction: Conviction,       // off-chain attribution only
    pub weighted: u128,
}
```

## Storage

| Name | Kind | Key → Value | Lifetime |
|---|---|---|---|
| `ReferendaTotalWeightedVotes` | StorageMap | `ReferendumIndex → ReferendumLiveTally` | voting period; deleted at allocation |
| `ReferendumTracks` | StorageMap | `ReferendumIndex → TrackId` | cached at first vote; deleted at allocation |
| `ReferendaRewardPool` | StorageMap | `ReferendumIndex → ReferendaReward<TrackId>` | presence == "allocation has run"; deleted when the last voter is paid |
| `UserVoteRecords` | StorageDoubleMap | `(AccountId, ReferendumIndex) → UserVoteRecord` | cast → `on_remove_vote` |
| `UserVoteCount` | StorageMap | `AccountId → u32` | mirrors the record count, for exact liquidation weight |
| `PendingRewards` | StorageMap | `AccountId → Balance` | until `claim_rewards` |

`STORAGE_VERSION = 0`.

## Extrinsics

| # | Name | Origin | Description |
|---|---|---|---|
| 0 | `claim_rewards()` | Signed | Re-checks `ExternalClaims::on(who) == 0` (mirrors `giga_stake`), `take`s the full `PendingRewards[who]`, transfers it allocated-pot → caller, then calls `pallet_gigahdx::Pallet::do_stake` to compound it into GIGAHDX. Emits `RewardsClaimed`. |

Errors: `NoPendingRewards`, `PotInsufficient`, `Overflow` (plus `pallet_gigahdx::Error::BlockedByExternalLock`).
Events: `RewardPoolAllocated`, `UserRewardRecorded`, `RewardsClaimed`.

## Hooks / lifecycle

`VotingHooksImpl<T>` (`pallets/gigahdx-rewards/src/voting_hooks.rs`) implements `pallet_conviction_voting::VotingHooks`:

| Hook | Behaviour |
|---|---|
| `on_before_vote` | Short-circuits `Ok(())` when `Stakes[who].hdx == 0`. Records **every** vote variant; `Split`/`SplitAbstain` are stored with `Conviction::None` (→ `weighted = 0`) so they still take a `voters_remaining` slot and are still reachable by the freeze and vote-clearance. Diffs against any prior record (edit → tally swap; new → `voters_count`/`UserVoteCount` bump). Caches the track id. **Never blocks voting** — saturating arithmetic, `Ok(())` on every defensive branch. |
| `on_remove_vote` | `take`s the `UserVoteRecord`. If the pool already exists → `record_user_reward` (must always run, regardless of the possibly-pruned referendum status, or `voters_remaining` never reaches zero and the pool leaks). Otherwise drop from the live tally, and if `status == Completed` → `maybe_allocate_and_record`. |
| `lock_balance_on_unsuccessful_vote` | Returns `record.staked_vote_amount` — **opts the losing side into the conviction lock**. Without it a staker could vote max-conviction on the losing side, collect the boosted multiplier and exit on only the unstake cooldown. |

### Allocation flow (`maybe_allocate_and_record`)

1. First `on_remove_vote` after `Status::Completed` and with no pool entry → resolve `track_id` (`ReferendumTracks` cache, falling back to `Referenda::track_of`); if unresolvable, return silently (that voter forfeits, tally preserved).
2. `allocation = TrackRewardConfig::reward_percentage(track) × free_balance(accumulator_pot)`; transfer accumulator → allocated pot.
3. Freeze the snapshot: `total_weighted_votes = live.total_weighted + record.weighted`, `voters_remaining = live.voters_count + 1` (the caller was already subtracted by `on_remove_vote`), `remaining_reward = total_reward`.
4. Delete `ReferendaTotalWeightedVotes[ref]` and `ReferendumTracks[ref]`; emit `RewardPoolAllocated`.
5. Then `record_user_reward` for the caller and for every subsequent remover.

### Per-user payout (`record_user_reward`)

`share = weighted × total_reward / total_weighted_votes` (rounded down), capped at `remaining_reward`, credited to `PendingRewards[who]`. When `voters_remaining` hits 0 the residual `remaining_reward` is transferred **back to the accumulator pot** (never scooped by the last claimant) and the pool entry is deleted.

## Unstake-freeze provider

```rust
// pallets/gigahdx-rewards/src/voting_hooks.rs
impl<T: Config> pallet_gigahdx::traits::VotingCommitmentInspect<T::AccountId> for Pallet<T> {
    fn committed_with_count(who: &T::AccountId) -> (Balance, u32) {
        // max, not sum: the same locked HDX backs every concurrent vote
        let mut max = 0; let mut count = 0u32;
        for record in UserVoteRecords::<T>::iter_prefix_values(who) {
            count = count.saturating_add(1);
            if record.staked_vote_amount > max { max = record.staked_vote_amount; }
        }
        (max, count)
    }
    fn committed_weight() -> Weight {
        // worst case: MaxVotes (25) × governance tracks (10) = 250 reads
        <T as frame_system::Config>::DbWeight::get().reads(250)
    }
}
```

Pulled **lazily at unstake time**, never maintained on the voting path — so voting costs nothing extra and `giga_unstake` refunds post-dispatch down to the reservations actually scanned.

## Integration points

| Direction | Counterparty | Seam |
|---|---|---|
| implements | `pallet-conviction-voting` | `VotingHooksImpl` inside `CombinedVotingHooks<StakingConvictionVoting, VotingHooksImpl>` (`runtime/hydradx/src/governance/mod.rs`) |
| implements | [[wiki/pallet-gigahdx\|pallet-gigahdx]] | `VotingCommitmentInspect` → `type VotingCommitment = GigaHdxRewards` |
| calls | [[wiki/pallet-gigahdx\|pallet-gigahdx]] | `Pallet::do_stake` from `claim_rewards` (compounding) |
| consumes | `pallet-referenda` | `ReferendaTrackInspect` → `RuntimeReferenda`; only `ReferendumInfo::Ongoing` exposes the track id, hence the `ReferendumTracks` cache |
| read by | [[wiki/pallet-liquidation\|pallet-liquidation]] | `GigaHdxVoteClearance::clear_conflicting_votes` walks `UserVoteRecords`; `clear_weight_for` reads `UserVoteCount` |
| funded by | `pallet-fee-processor` | `GigaHdxRewardsFeeReceiver` — 25% of trade fees |

## Gotchas

- `CombinedVotingHooks` is a runtime-local tuple adapter — upstream `VotingHooks` has no tuple impl. Its `lock_balance_on_unsuccessful_vote` takes the `max` of both consumers.
- `Conviction::None` votes earn **nothing** but still occupy a `voters_remaining` slot.
- Reward percentage is applied to the accumulator's **balance at completion time**, so a referendum that completes right after a large fee inflow allocates more; sequencing matters.
- `weights.rs` currently ships **placeholder** weights for `claim_rewards` (~140 ms, 15 reads / 7 writes), documented as "replaced by benchmarking during runtime upgrades".
- Behavioural coverage: `integration-tests/src/gigahdx_rewards.rs` (pro-rata splits, aye/nay symmetry, split-vote exclusion, track percentages, vote edits, cancelled referenda, dust recycling, compounding claim, unstake freeze).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/gigahdx\|gigahdx]]
- [[wiki/pallet-gigahdx\|pallet-gigahdx]]
- [[wiki/hdx\|hdx]]
- [[wiki/pallet-staking\|pallet-staking]]
- [[wiki/pallet-liquidation\|pallet-liquidation]]
- [[wiki/hydration-runtime\|hydration-runtime]]
