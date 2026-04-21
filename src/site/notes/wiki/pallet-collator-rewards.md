---
{"dg-publish":true,"permalink":"/wiki/pallet-collator-rewards/","title":"pallet-collator-rewards","tags":["collator","rewards","session","runtime","rust","substrate"],"dg-note-properties":{"type":"pallet","title":"pallet-collator-rewards","repo":"hydration-node","paths":["pallets/collator-rewards/src/lib.rs"],"symbols":["Pallet","Config","Collators","RewardPerCollator","RewardCurrencyId","RewardsBag","ExcludedCollators"],"traits_impl":["SessionManager"],"depends_on":[],"runtime_index":57,"tags":["collator","rewards","session","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-collator-rewards

**TL;DR:** Pays each non-excluded collator a fixed per-session reward (in `RewardCurrencyId`, typically HDX) pulled from a `RewardsBag` account at `end_session`. Wraps the inner `SessionManager` (usually `CollatorSelection`). Runtime index = 57.

## Role

Substrate's session rotation doesn't natively pay collators — this pallet inserts itself between `pallet_session` and `pallet_collator_selection` via the `SessionManager` trait, recording collators at `new_session` and paying them at `end_session`.

## Config trait (excerpt)

```rust
// pallets/collator-rewards/src/lib.rs
pub trait Config: frame_system::Config {
    type Balance: Parameter + Member + AtLeast32BitUnsigned + Default + Copy + ...;
    type CurrencyId: Parameter + Member + Copy + MaybeSerializeDeserialize + Ord;
    type Currency: MultiCurrency<Self::AccountId, CurrencyId = Self::CurrencyId, Balance = Self::Balance>;
    #[pallet::constant] type RewardPerCollator: Get<Self::Balance>;
    #[pallet::constant] type RewardCurrencyId: Get<Self::CurrencyId>;
    #[pallet::constant] type RewardsBag: Get<Self::AccountId>;
    type ExcludedCollators: Get<Vec<Self::AccountId>>;
    type SessionManager: SessionManager<Self::AccountId>;
    type MaxCandidates: Get<u32>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Collators` | StorageMap | `SessionIndex → BoundedVec<AccountId, MaxCandidates>` |

## Events

`CollatorRewarded` (who, amount, currency).

## Errors

None.

## Extrinsics

None.

## Hooks

None directly — lifecycle logic is in the `SessionManager` impl (called by `pallet_session`).

## Integration

- **Traits implemented:** `SessionManager<AccountId>` — wraps inner `SessionManager`
- **Traits consumed:** `MultiCurrency` (reward transfer), inner `SessionManager`
- **Pallets depended on:** `pallet_session`, `pallet_collator_selection`

## Key code

```rust
// pallets/collator-rewards/src/lib.rs
impl<T: Config> SessionManager<T::AccountId> for Pallet<T> {
    fn new_session(index: SessionIndex) -> Option<Vec<T::AccountId>> {
        let maybe = T::SessionManager::new_session(index);
        if let Some(ref collators) = maybe {
            if let Ok(bounded) = BoundedVec::try_from(collators.clone()) {
                Collators::<T>::insert(index, bounded);
            }
        }
        maybe
    }

    fn end_session(index: SessionIndex) {
        T::SessionManager::end_session(index);
        let excluded = T::ExcludedCollators::get();
        for c in Collators::<T>::take(index) {
            if !excluded.contains(&c) {
                let _ = T::Currency::transfer(
                    T::RewardCurrencyId::get(), &T::RewardsBag::get(), &c,
                    T::RewardPerCollator::get());
            }
        }
    }
}
```

## Gotchas

- `RewardsBag` must hold enough HDX — if empty, the `transfer` silently fails (return value ignored).
- `ExcludedCollators` is compile-time — usually just the Invulnerables list.
- No slashing — misbehaving collators must be removed via `pallet_collator_selection`.
- Reward amount only changes via runtime upgrade (constant).
- Storage is wiped at `end_session` (no accumulation).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
