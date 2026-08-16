---
{"dg-publish":true,"permalink":"/wiki/pallet-collator-rotation/","title":"pallet-collator-rotation","tags":["collator","session","rotation","parachain","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-collator-rotation","repo":"hydration-node","paths":["pallets/collator-rotation/src/lib.rs","pallets/collator-rotation/src/mock.rs","pallets/collator-rotation/src/tests.rs"],"symbols":["Pallet","Config","Inner","SessionManager","CollatorBenched","new_session","end_session","start_session"],"traits_impl":["SessionManager"],"depends_on":[],"runtime_index":58,"tags":["collator","session","rotation","parachain","runtime","rust","substrate"],"last_updated":"2026-05-14"}}
---


# pallet-collator-rotation

**TL;DR:** Thin `SessionManager` wrapper that benches one collator every **odd** session rotation by removing it from the active set produced by an inner `SessionManager`. Wired between [[wiki/pallet-collator-rewards\|pallet-collator-rewards]] and `pallet_collator_selection` to spread block authoring across the invulnerables pool over time. Runtime index = 58.

## Role

Hydration's collator set is closed (`MaxCandidates = 0`); the active set is drawn entirely from `Invulnerables` (up to 50 accounts — see [[wiki/runbook-run-collator\|runbook-run-collator]]). To prevent the same subset of invulnerables monopolising authoring slots, this pallet wraps the inner session manager and benches one collator each odd `SessionIndex`. The benched account rotates deterministically through the list.

Wiring order in `runtime/hydradx/src/system.rs`:

```text
Session.SessionManager = CollatorRewards
CollatorRewards.SessionManager = CollatorRotation   // this pallet
CollatorRotation.Inner       = CollatorSelection    // produces base list
```

So the rotation is applied *between* `CollatorSelection` (which decides who is eligible) and `CollatorRewards` (which pays out the active set).

## Config trait

```rust
// pallets/collator-rotation/src/lib.rs
pub trait Config: frame_system::Config<RuntimeEvent: From<Event<Self>>> {
    type Inner: SessionManager<Self::AccountId>;
}
```

That's the entire surface — a single associated type pointing at the underlying `SessionManager` impl (in production: `CollatorSelection`).

## Storage

None.

## Events

| Event | Fields | Trigger |
|---|---|---|
| `CollatorBenched` | `who: AccountId`, `session_index: SessionIndex` | Emitted from `new_session` when a collator was removed from the produced set |

## Errors

None.

## Extrinsics

None — this pallet exposes no dispatchables. All effects are via the `SessionManager` trait impl.

## Hooks / SessionManager impl

```rust
// pallets/collator-rotation/src/lib.rs
impl<T: Config> SessionManager<T::AccountId> for Pallet<T> {
    fn new_session(new_index: SessionIndex) -> Option<Vec<T::AccountId>> {
        let mut collators = T::Inner::new_session(new_index)?;
        // bench 1 collator every odd session rotation
        if new_index % 2 == 1 && collators.len() > 1 {
            let bench_idx = ((new_index / 2) as usize) % collators.len();
            let benched = collators.remove(bench_idx);
            Self::deposit_event(Event::CollatorBenched {
                who: benched,
                session_index: new_index,
            });
        }
        Some(collators)
    }
    fn end_session(end_index: SessionIndex) { T::Inner::end_session(end_index) }
    fn start_session(start_index: SessionIndex) { T::Inner::start_session(start_index) }
}
```

Behaviour:

- Even-indexed sessions: passed through untouched.
- Odd-indexed sessions: index `(new_index / 2) % len` is removed from the active list. Over time, every collator gets benched in rotation.
- Single-collator edge case: when `collators.len() == 1` the rotation is skipped (no benching) to keep the chain alive.
- A benched collator does **not** appear in the session-handler `validators` array and therefore receives no reward from [[wiki/pallet-collator-rewards\|pallet-collator-rewards]] for that session.

## Integration

- **Traits implemented:** `pallet_session::SessionManager<AccountId>` — exposed to wrap any inner `SessionManager`
- **Traits consumed:** `SessionManager` (via `T::Inner`)
- **Pallets depended on:** none directly; in production wired with [[wiki/pallet-collator-rewards\|pallet-collator-rewards]] above it and `pallet_collator_selection` below it (see [[wiki/hydration-runtime\|hydration-runtime]])

## Runtime wiring

```rust
// runtime/hydradx/src/system.rs
impl pallet_collator_rewards::Config for Runtime {
    // ...
    type SessionManager = CollatorRotation;  // wraps the inner SessionManager
}

impl pallet_collator_rotation::Config for Runtime {
    type Inner = CollatorSelection;
}
```

Construct_runtime index: `CollatorRotation = 58`.

## Gotchas

- **Benching is silent for the collator.** No on-chain warning beyond the `CollatorBenched` event; an operator watching `Session::Validators` will simply not see themselves in odd sessions.
- **Deterministic, not random.** The bench index is `(new_index / 2) % len` — collators benched in a given session can be predicted off-chain. This is a fairness mechanism, not anti-MEV.
- **`CollatorRewards` sits above this pallet**, so the benched collator does not earn the per-session HDX reward (`RewardPerCollator`) for the session it was removed from — see [[wiki/pallet-collator-rewards\|pallet-collator-rewards]].
- **No effect on `Invulnerables`.** Benched accounts remain in the invulnerables list; only the per-session active set is mutated.
- **`Period = 4 * HOURS`** (`runtime/hydradx/src/system.rs`) means a collator gets benched ~3 times per day if they keep landing on odd-session boundaries; cumulative downtime is roughly half of those, ~8 hours/week.
- This pallet's `SessionManager` impl is also what [[wiki/pallet-collator-rewards\|pallet-collator-rewards]] reads from — so any bug in `new_session` propagates to reward accounting.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/pallet-collator-rewards\|pallet-collator-rewards]]
- [[wiki/hydration-runtime\|hydration-runtime]]
- [[wiki/runbook-run-collator\|runbook-run-collator]]
