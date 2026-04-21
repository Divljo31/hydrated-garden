---
{"dg-publish":true,"permalink":"/wiki/hydration-runtime/","title":"hydration-runtime","tags":["runtime","construct_runtime","xcm","evm","frontier","cumulus","parachain"],"dg-note-properties":{"type":"runtime","title":"hydration-runtime","repo":"hydration-node","paths":["runtime/hydradx/src/lib.rs","runtime/hydradx/src/xcm.rs","runtime/hydradx/src/evm/mod.rs","runtime/hydradx/src/evm/precompiles/mod.rs","runtime/hydradx/src/governance/mod.rs","runtime/hydradx/src/system.rs","primitives/src/lib.rs"],"symbols":["Runtime","RuntimeCall","RuntimeEvent","RuntimeOrigin","construct_runtime","VERSION","SS58_PREFIX","BlockNumber","Balance","AssetId","AccountId","EvmAddress","HydraDXPrecompiles"],"traits_impl":[],"depends_on":["pallet-omnipool","pallet-stableswap","pallet-xyk","pallet-lbp","pallet-route-executor","pallet-asset-registry","pallet-ema-oracle","pallet-circuit-breaker","pallet-dynamic-fees","pallet-dynamic-evm-fee","pallet-currencies","pallet-evm-accounts","pallet-transaction-multi-payment","pallet-hsm","pallet-staking","pallet-dca","pallet-referrals","pallet-otc","pallet-otc-settlements","pallet-bonds","pallet-liquidation","pallet-liquidity-mining","pallet-omnipool-liquidity-mining","pallet-xyk-liquidity-mining","pallet-duster","pallet-nft","pallet-claims","pallet-dispenser","pallet-signet","pallet-broadcast","pallet-dispatcher","pallet-parameters","pallet-democracy","pallet-collator-rewards","pallet-xcm-rate-limiter","pallet-transaction-pause","pallet-relaychain-info","pallet-genesis-history"],"tags":["runtime","construct_runtime","xcm","evm","frontier","cumulus","parachain"],"last_updated":"2026-04-20"}}
---


# hydration-runtime

**TL;DR:** The HydraDX parachain runtime crate. Wires all 38 pallets into one `construct_runtime!`, defines primitives (Balance=u128, AssetId=u32, BlockNumber=u32, AccountId=sr25519-32, EvmAddress=H160), XCM configuration, EVM/Frontier configuration with custom precompiles, OpenGov tracks, and runtime APIs. Current `spec_version = 411`, `impl_name = "hydradx"`.

## Entrypoints

- `runtime/hydradx/src/lib.rs` — `construct_runtime!`, `VERSION`, runtime APIs, `ConstructRuntimeApi` impl
- `runtime/hydradx/src/system.rs` — `frame_system::Config`, base weights, `BlockWeights`, `BlockLength`
- `runtime/hydradx/src/xcm.rs` — XCM executor config, asset registry adapter, fee trader
- `runtime/hydradx/src/evm/mod.rs` — Frontier config, ChainId, gas weight mapping
- `runtime/hydradx/src/evm/precompiles.rs` — `HydraDXPrecompiles` set
- `runtime/hydradx/src/governance/mod.rs` — OpenGov tracks, origins
- `runtime/adapters/` — cross-pallet trait adapters (e.g. price oracles, fee handlers)

## Version

```rust
// runtime/hydradx/src/lib.rs
pub const VERSION: RuntimeVersion = RuntimeVersion {
    spec_name: Cow::Borrowed("hydradx"),
    impl_name: Cow::Borrowed("hydradx"),
    authoring_version: 1,
    spec_version: 411,
    impl_version: 0,
    apis: RUNTIME_API_VERSIONS,
    transaction_version: 1,
};

pub const SS58_PREFIX: u16 = 63;    // Hydration address prefix
```

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

System & consensus: System=1, Timestamp=3, ParachainSystem=5, ParachainInfo=6, Aura=33, AuraExt=34, CollatorSelection=21, Session=22.

Governance: Democracy=19, Scheduler=29, Preimage=30, Referenda=37, ConvictionVoting=36, Whitelist=38, Council=72, TechnicalCommittee=73, Treasury=17, Utility=10, Proxy=14, Multisig=11, Identity=16.

Tokens/assets: Balances=7, Tokens=77, Currencies=79, AssetRegistry=32, Duster=61, Claims=53, NFT (via Uniques)=32+78.

AMMs & trading: Omnipool=51, Stableswap=33 (inst), XYK=60, LBP=62, DCA=66, OTC=65, OTCSettlements=71, RouteExecutor=69.

Risk/fees: CircuitBreaker=63, DynamicFees=70, DynamicEvmFee=98, EmaOracle=72, TransactionPause=64, XcmRateLimiter=105.

Liquidity mining: LiquidityMining=99, WarehouseLM=100, OmnipoolLM=101, XYKLiquidityMining=95, XYKWarehouseLM=96.

EVM: EVM=90, Ethereum=91, EVMChainId=92, EVMAccounts=93.

HOLLAR: HSM=180.

Cross-chain: XcmpQueue=50, PolkadotXcm=103, CumulusXcm=104, XTokens=102, MessageQueue=108, Ismp=160, IsmpParachain=161, Bridge/ISMP related.

Infra: Vesting=15, Bonds=83 (or similar), Liquidation=94, Referrals=68, EthDispenser=85, SigNet=84, Broadcast=204, Dispatcher=40, Parameters=83, Staking=82, GenesisHistory=55, CollatorRewards=57, TransactionMultiPayment=12, RelaychainInfo=201.

(Exact indices tracked in `runtime/hydradx/src/lib.rs`; the above reflects the ordering at spec_version=411.)

## XCM configuration (runtime/hydradx/src/xcm.rs)

Key types:
- `Barrier` — standard Polkadot barrier (allow top-level paid, allow unpaid from Treasury, allow system parachains)
- `AssetTransactor` — `MultiCurrencyAdapter` bridging to `Currencies` + `AssetRegistry`
- `Trader` — `MultiCurrencyTrader` using `AssetRegistry` for fee-asset weights
- `XcmExecutor<XcmConfig>` — standard executor
- `LocationToAccountId` — `HashedDescription` + `AccountId32Aliases` + `ParentIsPreset`
- `ReserveAssetFilter` — via asset-registry-configured reserve locations
- XCM rate limiting: hooked through [[wiki/pallet-xcm-rate-limiter\|pallet-xcm-rate-limiter]]

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

## Governance (runtime/hydradx/src/governance/mod.rs)

OpenGov Tracks (examples): Root(0), WhitelistedCaller(1), GeneralAdmin(10), StakingAdmin(11), Treasurer(12), SpenderMajor(14), SpenderBig(21), SpenderSmall(22), ReferendumKiller(21), ReferendumCanceller(20), EmergencyAdmin.

Origins: `TechCommittee`, `TreasuryOrigin`, `MajorityCouncil`, `SuperMajorityCouncil`, `UnanimousCouncil`.

Legacy Gov1 [[wiki/pallet-democracy\|pallet-democracy]] retained at index 19; OpenGov [[pallet-referenda\|pallet-referenda]]=37 + ConvictionVoting=36 are the active path.

## Transaction payment

- `pallet_transaction_payment` (native) — HDX fees
- [[wiki/pallet-transaction-multi-payment\|pallet-transaction-multi-payment]] — accepts any registered asset, auto-swaps to HDX via Omnipool/Stableswap, uses EmaOracle for quoting

## Block production

- Aura (slot-based) via `pallet_aura` + `pallet_aura_ext`
- Collator selection via `pallet_collator_selection` (index 21), rewards via [[wiki/pallet-collator-rewards\|pallet-collator-rewards]] (index 57)
- Cumulus parachain integration via `cumulus_pallet_parachain_system` (index 5)

## Runtime APIs (selected)

Standard: `Core`, `Metadata`, `BlockBuilder`, `TaggedTransactionQueue`, `OffchainWorkerApi`, `SessionKeys`, `AccountNonceApi`, `TransactionPaymentApi`.

EVM/Frontier: `EthereumRuntimeRPCApi`, `ConvertTransactionRuntimeApi`, `DebugRuntimeApi`, `TxPoolRuntimeApi`.

Cross-chain: `CollectCollationInfo` (cumulus), `AuraApi`, `CumulusPrimitivesAuraApi`, ISMP-related APIs.

AMM quoting (custom): Some runtime APIs expose omnipool / stableswap / router quotes for SDK clients (see SDK packages).

## Gotchas

- `VERSION.spec_version` must be bumped on every runtime upgrade; `transaction_version` only when signature encoding changes.
- `ChainId` for EVM is `222_222` on mainnet — different from Ethereum.
- `pallet_uniques` at index 32 is shared storage for the [[wiki/pallet-nft\|pallet-nft]] wrapper — don't double-register collections.
- `XcmpQueue`, `PolkadotXcm`, `CumulusXcm`, `MessageQueue` are interconnected — changes to one affect others; see `runtime/hydradx/src/xcm.rs`.
- Pallet `Parameters` (index 83) is the go-to for runtime knobs that need frequent changes — avoid adding new `pallet::constant` types; prefer a `Parameters` entry instead.
- Many pallet wirings use adapter types in `runtime/adapters/` — e.g. `OmnipoolHooksAdapter`, `PriceProviderAdapter`, `FeeCurrencyAdapter`. When tracing trait implementations, check there.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
