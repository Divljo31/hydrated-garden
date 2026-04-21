---
{"dg-publish":true,"permalink":"/wiki/pallet-democracy/","title":"pallet-democracy","tags":["governance","democracy","legacy","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-democracy","repo":"hydration-node","paths":["pallets/democracy/src/lib.rs","pallets/democracy/src/types.rs","pallets/democracy/src/conviction.rs","pallets/democracy/src/vote.rs","pallets/democracy/src/vote_threshold.rs","pallets/democracy/src/traits.rs"],"symbols":["Pallet","Config","propose","second","vote","unvote","delegate","undelegate","external_propose","fast_track","cancel_referendum","note_preimage","ReferendumInfoOf","VotingOf","PublicProps","DemocracyHooks"],"traits_impl":[],"depends_on":[],"runtime_index":19,"tags":["governance","democracy","legacy","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-democracy

**TL;DR:** Hydration's forked variant of Substrate `pallet_democracy`. Legacy governance (Gov1) kept alive during the transition to OpenGov; adds a `DemocracyHooks` trait so [[wiki/pallet-staking\|pallet-staking]] can bind vote weight to HDX position points. Runtime index = 19.

## Role

Public-proposal + referendum framework with conviction voting. Largely superseded by OpenGov ([[pallet-referenda\|pallet-referenda]] + ConvictionVoting, indices 36/37), but remains for historical/transition reasons. The Hydration-specific addition is the `DemocracyHooks` trait that staking uses to track votes.

## Config trait (excerpt)

```rust
// pallets/democracy/src/lib.rs
pub trait Config: frame_system::Config + Sized {
    type WeightInfo: WeightInfo;
    type Scheduler: ScheduleNamed<BlockNumberFor<Self>, CallOf<Self>, ...>;
    type Preimages: QueryPreimage<H = Self::Hashing> + StorePreimage;
    type Currency: ReservableCurrency<Self::AccountId>
        + LockableCurrency<Self::AccountId, Moment = BlockNumberFor<Self>>;
    #[pallet::constant] type EnactmentPeriod: Get<BlockNumberFor<Self>>;
    #[pallet::constant] type LaunchPeriod: Get<BlockNumberFor<Self>>;
    #[pallet::constant] type VotingPeriod: Get<BlockNumberFor<Self>>;
    #[pallet::constant] type MinimumDeposit: Get<BalanceOf<Self>>;
    type ExternalOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type ExternalMajorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type ExternalDefaultOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type FastTrackOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type CancellationOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type BlacklistOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type CancelProposalOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type VetoOrigin: EnsureOrigin<Self::RuntimeOrigin, Success = Self::AccountId>;
    type Slash: OnUnbalanced<NegativeImbalanceOf<Self>>;
    type PalletsOrigin: From<RawOrigin<Self::AccountId>>;
    type MaxVotes: Get<u32>;
    type MaxProposals: Get<u32>;
    type MaxDeposits: Get<u32>;
    type MaxBlacklisted: Get<u32>;
    type DemocracyHooks: DemocracyHooks<Self::AccountId, BalanceOf<Self>>;
    // ... + MinimumDeposit, CooloffPeriod, VoteLockingPeriod, SubmissionDeposit
}
```

## Storage (major)

| Name | Kind | Key → Value |
|------|------|-------------|
| `PublicPropCount` | StorageValue | `PropIndex` |
| `PublicProps` | StorageValue | `BoundedVec<(PropIndex, BoundedCall, AccountId), MaxProposals>` |
| `DepositOf` | StorageMap | `PropIndex → (BoundedVec<AccountId, MaxDeposits>, Balance)` |
| `ReferendumCount` | StorageValue | `ReferendumIndex` |
| `ReferendumInfoOf` | StorageMap | `ReferendumIndex → ReferendumInfo` |
| `VotingOf` | StorageMap | `AccountId → Voting` |
| `Blacklist` | StorageMap | `Hash → (BlockNumber, BoundedVec<AccountId, MaxBlacklisted>)` |

## Events

`Proposed`, `Tabled`, `ExternalTabled`, `Started`, `Passed`, `NotPassed`, `Cancelled`, `Delegated`, `Undelegated`, `Vetoed`, `Blacklisted`, `Voted`, `Seconded`, `ProposalCanceled`, `MetadataSet`, `MetadataCleared`, `MetadataTransferred`.

## Errors

`ValueLow`, `ProposalMissing`, `DuplicateProposal`, `ProposalBlacklisted`, `NotSimpleMajority`, `InvalidHash`, `NoProposal`, `AlreadyVetoed`, `ReferendumInvalid`, `NotVoter`, `AlreadyDelegating`, `NotDelegating`, `VotesExist`, `MaxVotesReached`, `TooMany`, `VotingPeriodLow`.

## Extrinsics (major)

| Name | Description |
|------|-------------|
| `propose` | Submit proposal hash with deposit |
| `second` | Co-deposit / endorse an existing proposal |
| `vote` | Vote on a referendum with conviction |
| `remove_vote` | Retract an active vote |
| `delegate` | Delegate voting power to another account |
| `undelegate` | Cancel delegation |
| `external_propose` | Propose as `ExternalOrigin` (e.g. council) |
| `external_propose_majority` | With majority threshold |
| `fast_track` | Shorten voting period (emergency) |
| `cancel_referendum` | Root-cancel a referendum |
| `note_preimage` | Register proposal preimage (legacy path) |

## Hooks

`on_initialize` — `begin_block`: launches new referenda, finalizes ended ones, schedules enactment.

## Integration

- **Traits implemented:** none external (standard Substrate-style pallet)
- **Traits consumed:** `ReservableCurrency`, `LockableCurrency`, `ScheduleNamed`, `QueryPreimage`, `StorePreimage`, `DemocracyHooks`
- **Pallets depended on:** `pallet_scheduler`, `pallet_preimage`, [[wiki/pallet-staking\|pallet-staking]] (via `DemocracyHooks` — `LegacyStakingDemocracy` adapter)

## Hydration-specific: `DemocracyHooks`

```rust
// pallets/democracy/src/traits.rs
pub trait DemocracyHooks<AccountId, Balance> {
    fn on_vote(who: &AccountId, ref_index: ReferendumIndex, vote: AccountVote<Balance>) -> DispatchResult;
    fn on_remove_vote(who: &AccountId, ref_index: ReferendumIndex) -> DispatchResult;
}
```

Hydration staking uses this to award "action points" when users vote with conviction — locking HDX to vote counts toward staking rewards.

## Gotchas

- Conviction lock scales vote weight 1×..6× and locks stake 1..32 enactment periods.
- Fast-track requires `FastTrackOrigin` (typically Technical Committee).
- Preimage deposits are separate from proposal deposits; returned on enactment.
- Metadata hash is stored separately and can be transferred to a new owner.
- OpenGov ([[pallet-referenda\|pallet-referenda]] at index 37) is the preferred path; this pallet exists for legacy referenda still in flight.
- The `DemocracyHooks` binding is the key Hydration divergence from upstream.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
