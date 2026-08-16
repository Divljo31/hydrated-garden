---
{"dg-publish":true,"permalink":"/wiki/hydration-runtime/","title":"hydration-runtime","tags":["runtime","construct_runtime","xcm","evm","frontier","cumulus","parachain","gigahdx","synthetic-logs"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"runtime","title":"hydration-runtime","repo":"hydration-node","paths":["runtime/hydradx/src/lib.rs","runtime/hydradx/src/assets.rs","runtime/hydradx/src/xcm.rs","runtime/hydradx/src/evm/mod.rs","runtime/hydradx/src/evm/precompiles/mod.rs","runtime/hydradx/src/evm/synthetic_logs.rs","runtime/hydradx/src/evm/event_logs.rs","runtime/hydradx/src/gigahdx.rs","runtime/hydradx/src/governance/mod.rs","runtime/hydradx/src/governance/tracks.rs","runtime/hydradx/src/system.rs","runtime/hydradx/src/migrations/mod.rs","runtime/adapters/src/lib.rs","primitives/src/lib.rs"],"symbols":["Runtime","RuntimeCall","RuntimeEvent","RuntimeOrigin","construct_runtime","VERSION","BlockNumber","Balance","AssetId","AccountId","EvmAddress","HydraDXPrecompiles","GigaHdx","GigaHdxRewards","FeeProcessor","UnreleasedSingleBlockMigrations","CombinedVotingHooks","AaveMoneyMarket","TrackRewardConfig","InTradeContext","WETH_ASSET_ID","LOCK_MANAGER","SENTINEL_ADDRESS"],"traits_impl":[],"depends_on":["pallet-omnipool","pallet-stableswap","pallet-xyk","pallet-lbp","pallet-route-executor","pallet-asset-registry","pallet-ema-oracle","pallet-circuit-breaker","pallet-dynamic-fees","pallet-dynamic-evm-fee","pallet-currencies","pallet-evm-accounts","pallet-transaction-multi-payment","pallet-hsm","pallet-staking","pallet-dca","pallet-referrals","pallet-fee-processor","pallet-gigahdx","pallet-gigahdx-rewards","pallet-otc","pallet-otc-settlements","pallet-bonds","pallet-liquidation","pallet-liquidity-mining","pallet-omnipool-liquidity-mining","pallet-xyk-liquidity-mining","pallet-duster","pallet-nft","pallet-claims","pallet-dispenser","pallet-signet","pallet-broadcast","pallet-dispatcher","pallet-parameters","pallet-democracy","pallet-collator-rewards","pallet-collator-rotation","pallet-xcm-rate-limiter","pallet-transaction-pause","pallet-relaychain-info","pallet-genesis-history"],"tags":["runtime","construct_runtime","xcm","evm","frontier","cumulus","parachain","gigahdx","synthetic-logs"],"last_updated":"2026-08-15"}}
---


# hydration-runtime

**TL;DR:** The HydraDX parachain runtime crate. Wires all 42 pallets into one `construct_runtime!`, defines primitives (Balance=u128, AssetId=u32, BlockNumber=u32, AccountId=sr25519-32, EvmAddress=H160), XCM configuration, EVM/Frontier configuration with custom precompiles, OpenGov tracks, and runtime APIs. Current `spec_version = 439`, `impl_name = "hydradx"`. Spec 439 (Aug 2026) added the [[wiki/gigahdx\|gigahdx]] stack ([[wiki/pallet-gigahdx\|pallet-gigahdx]]=86, [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]]=87), [[wiki/pallet-fee-processor\|pallet-fee-processor]]=207, the `LOCK_MANAGER` precompile, NTT mint/burn on the multicurrency precompile, and off-chain **synthetic EVM logs**.

## Entrypoints

- `runtime/hydradx/src/lib.rs` — `construct_runtime!`, `VERSION`, runtime APIs, benchmark list
- `runtime/hydradx/src/assets.rs` — pallet `Config` impls for the AMM / fee / gigahdx stack (incl. `pallet_fee_processor::Config`, `pallet_gigahdx::Config`, `pallet_gigahdx_rewards::Config`)
- `runtime/hydradx/src/system.rs` — `frame_system::Config`, base weights, `BlockWeights`, `BlockLength`, `pallet_transaction_multi_payment::Config`
- `runtime/hydradx/src/xcm.rs` — XCM executor config, asset registry adapter, fee trader (**unchanged** since spec 419)
- `runtime/hydradx/src/evm/mod.rs` — Frontier config, ChainId, gas weight mapping, `WethAssetId`
- `runtime/hydradx/src/evm/precompiles/mod.rs` — `HydraDXPrecompiles` set, precompile addresses
- `runtime/hydradx/src/evm/synthetic_logs.rs`, `evm/event_logs.rs` — substrate-event → synthetic ETH tx/log translation primitives
- `runtime/hydradx/src/gigahdx.rs` — gigahdx runtime wiring (AAVE money-market adapter, reward-track table, external-claims / liquidation adapters)
- `runtime/hydradx/src/governance/mod.rs` — OpenGov origins, conviction voting, treasury
- `runtime/hydradx/src/migrations/mod.rs` — single-block + multi-block migration tuples
- `runtime/adapters/` — cross-pallet trait adapters (`OmnipoolHookAdapter`, price oracles, `XcmAssetExchanger`)

## Version

```rust
// runtime/hydradx/src/lib.rs
pub const VERSION: RuntimeVersion = RuntimeVersion {
    spec_name: Cow::Borrowed("hydradx"),
    impl_name: Cow::Borrowed("hydradx"),
    authoring_version: 1,
    spec_version: 439,          // was 419 at the previous vault sync
    impl_version: 0,
    apis: RUNTIME_API_VERSIONS,
    transaction_version: 1,
    system_version: 1,
};
```

Crate version tracks spec: `runtime/hydradx/Cargo.toml → version = "439.0.0"`.

`frame_system`'s prefix is `SS58Prefix: u16 = 0` in `runtime/hydradx/src/system.rs` (the `63` commonly quoted for Hydration is the ss58-registry entry used by wallets, not a runtime constant).

## Primitive types

```rust
// primitives/src/lib.rs
pub type BlockNumber = u32;
pub type Balance = u128;
pub type Amount = i128;             // signed for ORML
pub type AssetId = u32;
pub type AccountId = AccountId32;   // sr25519 32-byte
pub type Hash = H256;
pub type Signature = MultiSignature;
pub type Header = generic::Header<BlockNumber, BlakeTwo256>;
pub type Block = generic::Block<Header, UncheckedExtrinsic>;
pub type Price = FixedU128;
pub type EvmAddress = H160;
pub type Nonce = u32;
pub type Moment = u64;              // timestamp in ms
```

## Pallet indices (construct_runtime!)

System & consensus: System=1, Timestamp=3, Scheduler=5, Balances=7, TransactionPayment=9, Treasury=11, Utility=13, Preimage=15, Identity=17, Authorship=161, CollatorSelection=163, Session=165, Aura=167, AuraExt=169.

Governance: Democracy=19, TechnicalCommittee=25, Proxy=29, Multisig=31, Uniques=32, StateTrieMigration=35, ConvictionVoting=36, Referenda=37, Origins(custom)=38, Whitelist=39, Dispatcher=40.

HydraDX modules: AssetRegistry=51, Claims=53, GenesisHistory=55, CollatorRewards=57, CollatorRotation=58, Omnipool=59, TransactionPause=60, Duster=61, OmnipoolWarehouseLM=62, OmnipoolLiquidityMining=63, OTC=64, CircuitBreaker=65, DCA=66, Router=67, DynamicFees=68, Staking=69, Stableswap=70, Bonds=71, OtcSettlements=72, LBP=73, XYK=74, Referrals=75, Liquidation=76, HSM=82, Parameters=83, Signet=84, EthDispenser=85, **GigaHdx=86**, **GigaHdxRewards=87**.

ORML: Tokens=77, Currencies=79, Vesting=81, OrmlXcm=135, XTokens=137, UnknownTokens=139.

EVM/Frontier: EVM=90, EVMChainId=91, Ethereum=92, EVMAccounts=93, DynamicEvmFee=94.

Liquidity mining (XYK): XYKLiquidityMining=95, XYKWarehouseLM=96.

Parachain / cross-chain: ParachainSystem=103, ParachainInfo=105, PolkadotXcm=107, CumulusXcm=109, XcmpQueue=111, MessageQueue=114, WeightReclaim=115, MultiBlockMigrations=116.

Infra: RelayChainInfo=201, EmaOracle=202, MultiTransactionPayment=203, Broadcast=204, **FeeProcessor=207**.

Hyperbridge / ISMP / token-gateway: **removed** in spec 419; the `CleanupHyperbridge` migration ran on that upgrade and its file was deleted in spec 439. Nothing to reference any more.

Retired index notes carried in-source: 19/21/23/27 were gov-v1; 113 was `DmpQueue` (replaced by `MessageQueue`=114).

New in spec 439: `GigaHdx=86`, `GigaHdxRewards=87`, `FeeProcessor=207`. (Exact indices tracked in `runtime/hydradx/src/lib.rs`; the above reflects the ordering at spec_version=439.)

## Migrations (runtime/hydradx/src/migrations/mod.rs)

```rust
// runtime/hydradx/src/migrations/mod.rs
pub type UnreleasedSingleBlockMigrations = pallet_stableswap::migrations::v2::MigrateV1ToV2<Runtime>;
pub type PermanentSingleBlockMigrations = pallet_xcm::migration::MigrateToLatestXcmVersion<Runtime>;
pub type MultiBlockMigrations = ();
```

| Spec | Migration | Notes |
|---|---|---|
| 439 | `pallet_stableswap::migrations::v2::MigrateV1ToV2` | [[wiki/pallet-stableswap\|pallet-stableswap]] storage v1 → v2 (share mint/burn rework) |
| 419 (gone) | `pallet_ema_oracle::migrations::v2::MigrateV1ToV2`, `CleanupHyperbridge` | dropped from the tuple; `migrations/cleanup_hyperbridge.rs` deleted |

`UnreleasedSingleBlockMigrations` is emptied out after every runtime upgrade — treat its current contents as "what ships in the next runtime", not history.

## Trade-fee routing (spec 439 rewire)

`hydradx_adapters::OmnipoolHookAdapter::on_trade_fee` (`runtime/adapters/src/lib.rs`) no longer chains referrals-then-staking itself. It delegates the whole asset fee to [[wiki/pallet-fee-processor\|pallet-fee-processor]]:

```rust
// runtime/adapters/src/lib.rs — OmnipoolHooks::on_trade_fee
if asset == Lrna::get() {
    return Ok(vec![]);      // LRNA fees are not distributed
}
let result = pallet_fee_processor::Pallet::<Runtime>::process_trade_fee(fee_account, trader, asset.into(), amount)?;
Ok(vec![result])
```

Receiver split is declared in `runtime/hydradx/src/assets.rs` as `pallet_fee_processor::Config::FeeReceivers` / `HdxFeeReceivers`:

| Receiver struct | Destination account | Share | Path |
|---|---|---|---|
| `GigaHdxFeeReceiver` | `pallet_gigahdx::gigapot_account_id()` | 15% | converted to HDX |
| `GigaHdxRewardsFeeReceiver` | `pallet_gigahdx_rewards::reward_accumulator_pot()` | 25% | converted to HDX |
| `StakingFeeReceiver` / `HdxStakingFeeReceiver` | `pallet_staking::pot_account_id()` | 5% | non-HDX / HDX path |
| `ReferralsFeeReceiver` | `pallet_referrals::pot_account_id()` | 5% | `accepts_raw_asset() == true` — takes the slice in the **raw** asset, then `on_raw_fee_received` calls `pallet_referrals::process_trade_fee` |

Conversion is `ConvertViaOmnipool<Omnipool>`, moved from `pallet_referrals::traits::Convert` to `hydradx_traits::fee_processor::Convert`; its errors are now `pallet_fee_processor::Error::{PriceNotAvailable, ConversionFailed}`.

[[wiki/pallet-referrals\|pallet-referrals]] `FeeDistribution` lost its `external:` field — `referrer` + `trader` now split 100% **of the referrals slice** (Tier0 60/40 … Tier4 80/20), not of the whole trade fee. `ReferralsExternalRewardAccount` / `Config::ExternalAccount` were deleted.

## GigaHDX runtime wiring (runtime/hydradx/src/gigahdx.rs)

| Symbol | Role |
|---|---|
| `AaveMoneyMarket` | `MoneyMarketOperations` adapter — `supply`/`withdraw` against the EVM AAVE-V3 fork; pool address from `pallet_gigahdx::GigaHdxPoolContract` |
| `BenchmarkMoneyMarket` | benchmark-only stand-in (`runtime-benchmarks` feature) |
| `TrackRewardConfig` | per-OpenGov-track reward %: track 0 → 10%, 1 → 8%, 5 and 9 → 5%, default 2% |
| `RuntimeReferenda` | `ReferendaTrackInspect` over `pallet_referenda::ReferendumInfoFor`; only `Ongoing` carries a track id, completed variants return `None` |
| `HdxExternalClaims` | sums competing HDX locks; `ghdxlock` and `pyconvot` are the allowed overlaps |
| `GigaHdxVoteClearance` | force-removes conviction votes that would pin HDX about to be seized |
| `GigaHdxLiquidationSupport` | `Seize` + `pallet_liquidation::traits::GigaHdxSupport` |
| `LegacyStakingMigrator` / `LegacyStakingExternalClaims` | bridges [[wiki/pallet-staking\|pallet-staking]] positions into gigahdx |

Key `parameter_types!` in `runtime/hydradx/src/assets.rs`: `StHdxAssetId = 670`, `GigaHdxAssetIdConst = 67`, `GigaHdxPalletId = *b"gigahdx!"`, `GigaRewardPotPalletId = *b"gigarwd!"`, `GigaHdxLockId = *b"ghdxlock"`, `GigaHdxCooldownPeriod = 28 * DAYS`, `GigaHdxMinStake = UNITS`, `GigaHdxMaxPendingUnstakes = 10`.

`GIGAHDX_SOURCE = *b"gigahdxs"` (`primitives/src/constants.rs`) is a new oracle source: the chainlink-adapter precompile serves `pallet_gigahdx::exchange_rate()` under it instead of hitting [[wiki/pallet-ema-oracle\|pallet-ema-oracle]].

`ExtendedDustRemovalWhitelist` gained the gigahdx pallet account, both gigahdx-rewards pots, the fee-processor pot, and the [[wiki/pallet-otc-settlements\|pallet-otc-settlements]] account.

## XCM configuration (runtime/hydradx/src/xcm.rs)

Key types:
- `Barrier` — standard Polkadot barrier (allow top-level paid, allow unpaid from Treasury, allow system parachains)
- `AssetTransactor` — `MultiCurrencyAdapter` bridging to `Currencies` + `AssetRegistry`
- `Trader` — `MultiCurrencyTrader` using `AssetRegistry` for fee-asset weights
- `XcmExecutor<XcmConfig>` — standard executor
- `LocationToAccountId` — `HashedDescription` + `AccountId32Aliases` + `ParentIsPreset`
- `ReserveAssetFilter` — via asset-registry-configured reserve locations
- XCM rate limiting: hooked through [[wiki/pallet-xcm-rate-limiter\|pallet-xcm-rate-limiter]]

`runtime/hydradx/src/xcm.rs` itself is unchanged since spec 419. The XCM **exchange** adapter did change (`runtime/adapters/src/xcm_exchange.rs`, `XcmAssetExchanger`):

- `pallet_broadcast::add_to_context(ExecutionType::XcmExchange)` is now pushed **after** the circuit-breaker mint check, not before — a rejected exchange no longer leaves a stale broadcast context (paired with the `BatchHook::on_batch_end` signature change below).
- Buy path checks `IssuanceIncreaseFuse::can_mint(asset_in, max_sell_amount)` instead of the router's quoted `amount_in` — the deposit limit must apply to what is actually minted (the ceiling), not the quote.

## EVM configuration (runtime/hydradx/src/evm/)

```rust
// runtime/hydradx/src/evm/mod.rs (abridged)
pub const WEIGHT_PER_GAS: u64 = 25_000;
pub const GAS_PER_SECOND: u64 = 40_000_000;
pub const BLOCK_GAS_LIMIT: u64 = 15_000_000;
pub struct FixedGasPrice; // reads from DynamicEvmFee

impl pallet_evm::Config for Runtime {
    type FeeCalculator = DynamicEvmFee;
    type GasWeightMapping = FixedGasWeightMapping<Self>;
    type Runner = pallet_evm::runner::stack::Runner<Self>;
    type PrecompilesType = HydraDXPrecompiles<Runtime>;
    type ChainId = EVMChainId;     // 222_222 on mainnet (Hydration's custom)
    type OnChargeTransaction = TransferEvmFees<...>;
    type AddressMapping = pallet_evm_accounts::AddressMapping;
    ...
}
```

See [[wiki/hydration-precompiles\|hydration-precompiles]] for the precompile set.

### EVM changes in spec 439

**WETH asset id is pinned, not resolved.** `WethAssetId` used to look up a hard-coded Moonbeam `AssetLocation` in the asset registry. It is now a constant, because the MRL→NTT repoint of asset 20 to a local ERC-20 would have zeroed the gas asset the moment the registry entry moved:

```rust
// runtime/hydradx/src/evm/mod.rs
pub const WETH_ASSET_ID: AssetId = 20;

pub struct WethAssetId;
impl Get<AssetId> for WethAssetId {
    fn get() -> AssetId { WETH_ASSET_ID }
}
```

`weth_asset_location()` and `MOONBEAM_PARA_ID` are gone.

**New precompile: `LOCK_MANAGER` at `0x…0806`** (`precompiles/lock-manager/`, crate `pallet-evm-precompile-lock-manager`). Consumed by `LockableAToken.sol`; gated to a single caller via `GigaHdxATokenAddress`, which resolves the GIGAHDX aToken address through `HydraErc20Mapping` at call time. `is_precompile()` now includes it.

**NTT mint/burn on the multicurrency precompile.** `evm/erc20_currency.rs → Function` gained `Mint = "mint(address,uint256)"` and `Burn = "burn(uint256)"`; `MultiCurrencyPrecompile` serves them only for assets with a bound NTT minter, and now also requires `pallet_circuit_breaker::Config` (burn is throttled by the global withdraw limit, over-limit mint locks the asset down). There is deliberately **no** `setMinter` selector — the minter is set only via [[wiki/pallet-evm-accounts\|pallet-evm-accounts]]'s `set_ntt_minter` extrinsic, guarded by a new, faster origin:

```rust
// runtime/hydradx/src/evm/mod.rs — pallet_evm_accounts::Config
type ControllerOrigin    = EitherOf<EnsureRoot<Self::AccountId>, GeneralAdmin>;
type NttEmergencyOrigin  = EitherOf<EnsureRoot<Self::AccountId>,
                             EitherOf<crate::governance::TechCommitteeMajority, GeneralAdmin>>;
```

**Precompile helpers added** in `evm/precompiles/mod.rs`: `emit_approval_log()` (inline ERC-20 `Approval`) and `revert_custom_error(signature, args)` (Solidity custom-error revert encoding).

**Chainlink adapter** (`evm/precompiles/chainlink_adapter.rs`) now bounds `pallet_gigahdx::Config` and serves `GIGAHDX_SOURCE` from `pallet_gigahdx::exchange_rate()` (floored at 1.0 so AAVE never sees a sub-1 reading).

## Synthetic EVM logs

Substrate token/trade activity is surfaced to ETH JSON-RPC as synthetic ETH txs/logs. Pure translation lives in the runtime crate; assembly and serving are **entirely off-chain in the node** (no on-chain synth txs, no new runtime API) — see [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]].

| File | Contents |
|---|---|
| `runtime/hydradx/src/evm/synthetic_logs.rs` | primitives: `SENTINEL_ADDRESS` (`0x73796e74680000000000000000000073796e7468`, ascii `synth`), `SYNTH_SIG_RS`, `Bucket` (`Extrinsic(u32)` \| `Hook{phase, origin}`), `HookPhase`, `assemble_synth_txs`, `build_erc20_transfer_log`, `build_uniswap_v2_swap_log`, `reserved_address_of` / `frozen_address_of`, `APPROVAL_TOPIC`, `encode_u256_be`, `h160_to_h256` |
| `runtime/hydradx/src/evm/event_logs.rs` | `synthetic_txs_from_records` — `Vec<EventRecord>` → `(Transaction, TransactionStatus, Receipt)` triples with **no state reads**, so the node can call it for any runtime version. Balance moves → ERC-20 `Transfer` (reserved/frozen routed to per-owner sentinel addresses so aggregated transfers equal transferable balance); `pallet_broadcast::Swapped3` → Uniswap-V2 `Swap`; internal `pallet_evm::Log` deduped against real ETH txs |

## Governance (runtime/hydradx/src/governance/mod.rs)

OpenGov Tracks (examples): Root(0), WhitelistedCaller(1), GeneralAdmin(10), StakingAdmin(11), Treasurer(12), SpenderMajor(14), SpenderBig(21), SpenderSmall(22), ReferendumKiller(21), ReferendumCanceller(20), EmergencyAdmin.

Origins: `TechCommittee`, `TreasuryOrigin`, `MajorityCouncil`, `SuperMajorityCouncil`, `UnanimousCouncil`.

Legacy Gov1 [[wiki/pallet-democracy\|pallet-democracy]] retained at index 19; OpenGov [[pallet-referenda\|pallet-referenda]]=37 + ConvictionVoting=36 are the active path.

**No track or origin was added in spec 439** — `governance/tracks.rs` and `governance/origins.rs` are unchanged. What changed is the conviction-voting wiring:

```rust
// runtime/hydradx/src/governance/mod.rs — pallet_conviction_voting::Config
type MaxVotes = MaxVotes;                       // parameter_types! const = 25, moved here from assets.rs
type VotingHooks = CombinedVotingHooks<
    pallet_staking::integrations::conviction_voting::StakingConvictionVoting<Runtime>,
    pallet_gigahdx_rewards::voting_hooks::VotingHooksImpl<Runtime>,
>;
```

`CombinedVotingHooks<A, B>` is a runtime-local tuple adapter for `pallet_conviction_voting::VotingHooks` (upstream has no tuple impl), so both [[wiki/pallet-staking\|pallet-staking]] and [[wiki/pallet-gigahdx-rewards\|pallet-gigahdx-rewards]] observe every vote. `lock_balance_on_unsuccessful_vote` combines conservatively (`max` when both answer).

`MaxVotes` is now shared by `pallet_staking::Config` and `pallet_conviction_voting::Config`, enforced by a `static_assertions::const_assert_eq!` in `runtime/hydradx/src/assets.rs`. Bumping one without the other is a compile error — intentionally.

Voting on a referendum earns gigahdx rewards sized by `crate::gigahdx::TrackRewardConfig` (see the GigaHDX section above).

## Transaction payment

- `pallet_transaction_payment` (native) — HDX fees
- [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]] — accepts any registered asset, auto-swaps to HDX via Omnipool/Stableswap, uses EmaOracle for quoting
- New in 439: `pallet_transaction_multi_payment::Config::EvmFeePayer = evm::EvmFeePayerImpl` (`runtime/hydradx/src/evm/evm_fee.rs`) — lets an EVM call set/clear the account charged for fees (backs [[wiki/pallet-dispatcher\|pallet-dispatcher]]'s `dispatch_with_fee_payer`)

## Circuit breaker wiring

`pallet_circuit_breaker::Config` gained `type InTradeContext = InTradeContext` (`runtime/hydradx/src/assets.rs`):

```rust
// runtime/hydradx/src/assets.rs
pub struct InTradeContext;
impl Get<bool> for InTradeContext {
    fn get() -> bool { pallet_broadcast::Pallet::<Runtime>::get_swapper().is_some() }
}
```

True only while a route is executing — the only window in which the router holds a balance and a deposit-limit error should unwind rather than lock down. See [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]].

`RetryOnErrorForDca` also now retries `pallet_route_executor::Error::TradingLimitReached` (ERC-20 / aToken rounding can undershoot the dry-run output used as the router min limit). See [[wiki/pallet-dca\|pallet-dca]].

## Block production

- Aura (slot-based) via `pallet_aura` (index 167) + `pallet_aura_ext` (index 169)
- Collator selection via `pallet_collator_selection` (index 163) → wrapped by [[wiki/pallet-collator-rotation\|pallet-collator-rotation]] (index 58) → wrapped by [[wiki/pallet-collator-rewards\|pallet-collator-rewards]] (index 57). Final `pallet_session.SessionManager` is `CollatorRewards`. The rotation pallet benches one collator every odd `SessionIndex`.
- Cumulus parachain integration via `cumulus_pallet_parachain_system` (index 103)

## Runtime APIs (selected)

Standard: `Core`, `Metadata`, `BlockBuilder`, `TaggedTransactionQueue`, `OffchainWorkerApi`, `SessionKeys`, `AccountNonceApi`, `TransactionPaymentApi`.

EVM/Frontier: `EthereumRuntimeRPCApi`, `ConvertTransactionRuntimeApi`, `DebugRuntimeApi`, `TxPoolRuntimeApi`.

Cross-chain: `CollectCollationInfo` (cumulus), `AuraApi`, `CumulusPrimitivesAuraApi`. ISMP-related APIs were removed alongside the Hyperbridge pallets in spec 419.

AMM quoting (custom): Some runtime APIs expose omnipool / stableswap / router quotes for SDK clients (see SDK packages).

**No runtime API was added or removed in spec 439.** The synthetic-logs feature deliberately avoids one — the node translates events client-side so it works across runtime versions. The [[wiki/pepl-worker\|pepl-worker]] consumes existing APIs only: `EthereumRuntimeRPCApi`, `Erc20MappingApi`, `CurrenciesApi`, `xcm_runtime_apis::dry_run::DryRunApi`.

## Benchmarks

New entries in `mod benches` (`runtime/hydradx/src/lib.rs`): `[pallet_gigahdx, GigaHdx]`, `[pallet_gigahdx_rewards, GigaHdxRewards]`, and `[pallet_fee_processor, benchmarking::fee_processor::Benchmark]` (runtime-side benchmark in `runtime/hydradx/src/benchmarking/fee_processor.rs`). New weight files: `weights/pallet_fee_processor.rs`, `weights/pallet_gigahdx.rs`, `weights/pallet_gigahdx_rewards.rs`. Regenerated: omnipool, omnipool-liquidity-mining, stableswap, referrals, referenda, conviction-voting, liquidation, evm-accounts, xcm.

## Gotchas

- `VERSION.spec_version` must be bumped on every runtime upgrade; `transaction_version` only when signature encoding changes.
- `ChainId` for EVM is `222_222` on mainnet — different from Ethereum.
- `pallet_uniques` at index 32 is shared storage for the [[wiki/pallet-nft\|pallet-nft]] wrapper — don't double-register collections.
- `XcmpQueue`, `PolkadotXcm`, `CumulusXcm`, `MessageQueue` are interconnected — changes to one affect others; see `runtime/hydradx/src/xcm.rs`.
- Pallet `Parameters` (index 83) is **not** the FRAME generic parameters pallet — it's a small Hydration-specific store for `IsTestnet` / `RelayParentOffsetOverride` flags. See [[wiki/pallet-parameters\|pallet-parameters]].
- Many pallet wirings use adapter types in `runtime/adapters/` — e.g. `OmnipoolHookAdapter`, `PriceProviderAdapter`, `FeeCurrencyAdapter`. When tracing trait implementations, check there.
- Hyperbridge / ISMP / token-gateway integration was deleted in spec 419 and its cleanup migration file was removed in spec 439. `runtime/hydradx/src/migrations/cleanup_hyperbridge.rs` **no longer exists** — do not cite it, and do not add wiki references to those pallets.
- Trade fees no longer flow through the adapter's referrals→staking chain. Anything reasoning about fee destinations must go through [[wiki/pallet-fee-processor\|pallet-fee-processor]] (`runtime/hydradx/src/assets.rs` holds the split). LRNA fees are still dropped in the adapter before the processor is reached.
- `WethAssetId` is a **hard-coded id (20)**, not a registry lookup. Repointing asset 20 in the registry silently changes the EVM gas asset.
- `MaxVotes` lives in `governance/mod.rs` and is asserted equal across `pallet_staking` and `pallet_conviction_voting` — change both or the build fails.
- The `LOCK_MANAGER` precompile (`0x…0806`) sits **below** the `0x…080a` call-permit address and outside the asset-precompile range; `is_precompile()` special-cases it. It only answers the GIGAHDX aToken contract.
- Synthetic logs are node-side only. Nothing in `evm/synthetic_logs.rs` / `evm/event_logs.rs` reads chain state or is dispatchable — do not look for on-chain synth txs.
- `BatchHook::on_batch_end` no longer returns `DispatchResult` (`runtime/hydradx/src/system.rs`) — items may already be committed at that point, so `Broadcast::remove_from_context()` errors are swallowed and logged.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/pepl-worker\|pepl-worker]]
