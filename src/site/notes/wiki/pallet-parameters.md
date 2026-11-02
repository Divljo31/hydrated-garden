---
{"dg-publish":true,"permalink":"/wiki/pallet-parameters/","title":"pallet-parameters","tags":["parameters","runtime","flags","xcm","emergency","uniswap-v3","aave","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-parameters","repo":"hydration-node","paths":["pallets/parameters/src/lib.rs","runtime/hydradx/src/system.rs","runtime/hydradx/src/xcm.rs"],"symbols":["Pallet","Config","AuthorityOrigin","EmergencyOrigin","IsTestnet","RelayParentOffsetOverride","TwoSecBlocksSince","AaveGasLimitsOverride","AaveGasLimits","UniswapV3Factory","UniswapV3SwapRouter","UniswapV3Quoter","XcmSendSuspended","XcmExecutionSuspended","set_uniswap_v3_addresses","set_aave_gas_limits","suspend_xcm_send","resume_xcm_send","suspend_xcm_execution","resume_xcm_execution","suspend_xcm","resume_xcm","PausableRouter","XcmExecutionPausedQuery"],"traits_impl":[],"depends_on":["hydration-runtime"],"runtime_index":83,"tags":["parameters","runtime","flags","xcm","emergency","uniswap-v3","aave","rust","substrate"],"last_updated":"2026-11-02"}}
---


# pallet-parameters

> ⚠ **Live status (2026-11-02):** the XCM suspend / resume calls below are on master (spec 458) only. The live chain runs spec 455, whose `Parameters` exposes only `set_uniswap_v3_addresses` and `set_aave_gas_limits` — see [[wiki/note-live-chain-check\|note-live-chain-check]].

**TL;DR:** Hydration-specific (NOT the FRAME `pallet_parameters`) runtime-parameter store: testnet / relay-parent flags, the 2 s block switch point, Aave executor gas limits, Uniswap v3 contract addresses, and the **XCM kill switch**. `suspend_xcm` stops all inbound and outbound XCM in one `Operational` call and can be dispatched by **any single Technical Committee member**; resuming needs a full `AuthorityOrigin`. Runtime index = 83.

## Config trait

```rust
// pallets/parameters/src/lib.rs
pub trait Config: frame_system::Config {
    /// Origin allowed to change parameters.
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    /// Origin allowed to stop all XCM in one call. Meant to be wider than `AuthorityOrigin`
    /// (e.g. any single technical committee member), as it can only stop, never resume.
    type EmergencyOrigin: EnsureOrigin<Self::RuntimeOrigin>;
}
```

Runtime (`runtime/hydradx/src/system.rs`):

```rust
type AuthorityOrigin = EitherOf<EnsureRoot<AccountId>, EitherOf<TechCommitteeSuperMajority, GeneralAdmin>>;
type EmergencyOrigin = EitherOfDiverse<EnsureSignedBy<TechCommAccounts, AccountId>, Self::AuthorityOrigin>;
```

## Storage

| Name | Kind | Value | Used by |
|---|---|---|---|
| `IsTestnet` | Value | `bool` | adapters branching on testnet |
| `RelayParentOffsetOverride` | Value | `bool` | fork/test environments |
| `TwoSecBlocksSince` | Value | `BlockNumber` (default `u32::MAX`) — block at which the chain switched 6 s → 2 s | [[wiki/pallet-staking\|pallet-staking]], [[wiki/pallet-gigahdx\|pallet-gigahdx]] period math via `TwoSecBlocksSinceProvider` (`runtime/hydradx/src/assets.rs`) |
| `AaveGasLimitsOverride` | Value | `Option<AaveGasLimits { trade, view, reserves_list }>` | Aave trade executor; `None` = runtime defaults |
| `UniswapV3Factory` / `UniswapV3SwapRouter` / `UniswapV3Quoter` | Value | `Option<H160>` | `UniswapV3TradeExecutor` ([[wiki/pallet-route-executor\|pallet-route-executor]] `PoolType::UniswapV3`) |
| `XcmSendSuspended` | Value | `bool` | `PausableRouter` (outbound) in `runtime/hydradx/src/xcm.rs` |
| `XcmExecutionSuspended` | Value | `bool` | `XcmExecutionPausedQuery` (MessageQueue inbound, relay + siblings) |

## Extrinsics

| # | Name | Origin | Effect |
|---|---|---|---|
| 0 | `set_uniswap_v3_addresses(factory, swap_router, quoter)` | Authority | Point the v3 venue at a deployment |
| 1 | `set_aave_gas_limits(Option<AaveGasLimits>)` | Authority | Override / reset Aave executor gas limits |
| 2 | `suspend_xcm_send` | Authority | Stop outbound XCM |
| 3 | `resume_xcm_send` | Authority | Resume outbound |
| 4 | `suspend_xcm_execution` | Authority | Stop inbound XCM processing |
| 5 | `resume_xcm_execution` | Authority | Resume inbound |
| 6 | `suspend_xcm` | **Emergency** (one TC member) — `DispatchClass::Operational` | Stop both directions |
| 7 | `resume_xcm` | Authority (TC super-majority = 2/3) — `Operational` | Resume both |

`IsTestnet` / `RelayParentOffsetOverride` are seeded by `GenesisConfig`; `set_testnet_flag` / `set_relay_parent_offset_override` are plain `pub fn` helpers for tests, not extrinsics.

Events: `UniswapV3AddressesSet`, `AaveGasLimitsSet`, `XcmSendSuspended`, `XcmSendResumed`, `XcmExecutionSuspended`, `XcmExecutionResumed`, `XcmSuspended`, `XcmResumed`. Errors: `AlreadySuspended`, `AlreadyResumed`.

## Gotchas

- **Stop is cheap, resume is expensive by design.** Any one TC member can `suspend_xcm`; resuming needs TC super-majority / GeneralAdmin / Root. Plan the resume motion before pulling the lever.
- Inbound suspension holds messages in `MessageQueue` (paused queues), it does not drop them — they execute after resume, including any malicious ones. Pair with asset-level controls ([[wiki/note-emergency-levers\|note-emergency-levers]]).
- Outbound suspension makes every `SendXcm::validate` fail with `Transport("XCM sending is suspended")` — user XCM transfers out and any pallet that sends XCM fail.
- A wrong Uniswap v3 address fails silently: `getPool` finds no pool and the venue just goes quiet. Verify against deployment artefacts after enactment; off-chain consumers should read these storage values, not hardcode.
- `TwoSecBlocksSince` defaults to `u32::MAX` (= "not yet switched"); period math in staking/gigahdx depends on it being set correctly.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hydration-runtime\|hydration-runtime]]
- [[wiki/note-emergency-levers\|note-emergency-levers]]
