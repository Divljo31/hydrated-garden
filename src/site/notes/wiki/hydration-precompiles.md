---
{"dg-publish":true,"permalink":"/wiki/hydration-precompiles/","title":"hydration-precompiles","tags":["evm","precompiles","frontier","erc20","ntt","gigahdx","runtime","rust"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"runtime","title":"hydration-precompiles","repo":"hydration-node","paths":["runtime/hydradx/src/evm/precompiles/mod.rs","runtime/hydradx/src/evm/precompiles/erc20_mapping.rs","runtime/hydradx/src/evm/precompiles/multicurrency.rs","runtime/hydradx/src/evm/precompiles/chainlink_adapter.rs","runtime/hydradx/src/evm/precompiles/costs.rs","runtime/hydradx/src/evm/precompiles/handle.rs","runtime/hydradx/src/evm/precompiles/substrate.rs","runtime/hydradx/src/evm/erc20_currency.rs","runtime/hydradx/src/evm/mod.rs","precompiles/call-permit/src/lib.rs","precompiles/flash-loan/src/lib.rs","precompiles/lock-manager/src/lib.rs","precompiles/utils/src/lib.rs","traits/src/evm.rs"],"symbols":["HydraDXPrecompiles","LockManagerPrecompile","GigaHdxATokenAddress","AllowedFlashLoanCallers","CallPermitPrecompile","FlashLoanReceiverPrecompile","MultiCurrencyPrecompile","ChainlinkOraclePrecompile","HydraErc20Mapping","LOCK_MANAGER","CALLPERMIT","FLASH_LOAN_RECEIVER","DISPATCH_ADDR","ETH_PRECOMPILE_END","is_precompile","is_standard_precompile","is_asset_address","is_oracle_address","encode_oracle_address","decode_oracle_address","emit_approval_log","revert_custom_error","Erc20Mapping","Erc20Encoding","Erc20Inspect","Function","addr"],"traits_impl":["PrecompileSet","Precompile","Erc20Mapping","Erc20Encoding"],"depends_on":["pallet-evm-accounts","pallet-gigahdx","pallet-hsm","pallet-circuit-breaker","pallet-liquidation","pallet-stableswap"],"tags":["evm","precompiles","frontier","erc20","ntt","gigahdx","runtime","rust"],"last_updated":"2026-08-15"}}
---


# hydration-precompiles

**TL;DR:** Hydration's EVM precompile set — standard Frontier precompiles at `0x01..=0x09`, plus fixed-address Hydration precompiles (Dispatch, CallPermit, FlashLoanReceiver, **LockManager `0x0806`**) and two *dynamic-address* families (ERC-20-per-`AssetId`, Chainlink-style oracles). Wired into `pallet_evm::Config::PrecompilesType` as `HydraDXPrecompiles<Runtime>`.
Entry point: `runtime/hydradx/src/evm/precompiles/mod.rs` (**not** `evm/precompiles.rs` — that path no longer exists).

## Address map

### Standard Ethereum precompiles

| Address | Name | Impl |
|---|---|---|
| `0x…0001` | ECRecover | `pallet_evm_precompile_simple::ECRecover` |
| `0x…0002` | SHA256 | `pallet_evm_precompile_simple::Sha256` |
| `0x…0003` | RIPEMD160 | `pallet_evm_precompile_simple::Ripemd160` |
| `0x…0004` | Identity | `pallet_evm_precompile_simple::Identity` |
| `0x…0005` | Modexp | `pallet_evm_precompile_modexp::Modexp` |
| `0x…0006` | BN128Add | `pallet_evm_precompile_bn128::Bn128Add` |
| `0x…0007` | BN128Mul | `pallet_evm_precompile_bn128::Bn128Mul` |
| `0x…0008` | BN128Pairing | `pallet_evm_precompile_bn128::Bn128Pairing` |
| `0x…0009` | Blake2F | `pallet_evm_precompile_blake2::Blake2F` |

`ETH_PRECOMPILE_END = BLAKE2F`; `is_standard_precompile(a) == (!a.is_zero() && a <= 0x09)`.

### Hydration fixed-address precompiles

| Address | Dec | Const | Crate / impl | Purpose |
|---|---|---|---|---|
| `0x0000000000000000000000000000000000000401` | 1025 | `DISPATCH_ADDR` | `pallet_evm_precompile_dispatch::Dispatch` (upstream [[wiki/pallet-frontier\|pallet-frontier]]) | Execute a SCALE-encoded Substrate `RuntimeCall` from EVM context — the main EVM→Substrate bridge |
| `0x0000000000000000000000000000000000000806` | 2054 | `LOCK_MANAGER` | `precompiles/lock-manager` → `LockManagerPrecompile` | **New.** Reports locked GIGAHDX per account; makes GIGAHDX aTokens non-transferable |
| `0x000000000000000000000000000000000000080a` | 2058 | `CALLPERMIT` | `precompiles/call-permit` → `CallPermitPrecompile` | EIP-712 typed call permit (gasless / relayer pattern) |
| `0x000000000000000000000000000000000000090a` | 2314 | `FLASH_LOAN_RECEIVER` | `precompiles/flash-loan` → `FlashLoanReceiverPrecompile` | ERC-3156 `onFlashLoan` callback; callers gated by `AllowedFlashLoanCallers` |

Addresses `0x0401 / 0x080a / 0x090a` match Moonbeam/Centrifuge for tooling interoperability.

### Dynamic-address families

| Family | Prefix test | Layout | Handler |
|---|---|---|---|
| Asset ERC-20 | `is_asset_address` — bytes `0..16 == 0x00000000000000000000000000000001` | bytes `16..20` = `AssetId` (u32, big-endian) | `MultiCurrencyPrecompile<R>` |
| Chainlink oracle | `is_oracle_address` — bytes `0..3 == 0x000001` | `[3]` = `OraclePeriod`, `[4..12]` = `Source` (8-byte ascii, e.g. `omnipool` / `gigahdxs`), `[12..16]` = asset A BE, `[16..20]` = asset B BE | `ChainlinkOraclePrecompile<R>` |

Examples: `AssetId=5` → `0x0000000000000000000000000000000100000005`; stHDX(670)/0 ten-minute GIGAHDX oracle → `0x0000010267696761686478730000029e00000000` (`02` = `TenMinutes`, `6769676168647873` = `gigahdxs`, `0000029e` = 670).

## `HydraDXPrecompiles<R>` dispatch

```rust
// runtime/hydradx/src/evm/precompiles/mod.rs (abridged)
fn execute(&self, handle: &mut impl PrecompileHandle) -> Option<PrecompileResult> {
    let address = handle.code_address();
    // custom precompiles may not be reached via DELEGATECALL / CALLCODE
    if handle.context().address != address && is_precompile(address) && !is_standard_precompile(address) {
        return Some(Err(PrecompileFailure::Revert { .. "precompile cannot be called with DELEGATECALL or CALLCODE" }));
    }
    if address == ECRECOVER { .. }                       // 0x01..=0x09
    else if address == CALLPERMIT { .. }
    else if address == FLASH_LOAN_RECEIVER { .. }
    else if address == LOCK_MANAGER {
        Some(LockManagerPrecompile::<R, GigaHdxATokenAddress>::execute(handle))
    } else if address == DISPATCH_ADDR { /* nonce clamp, see Gotchas */ }
    else if is_asset_address(address) { Some(MultiCurrencyPrecompile::<R>::execute(handle)) }
    else if is_oracle_address(address) { Some(ChainlinkOraclePrecompile::<R>::execute(handle)) }
    else { None }
}
```

`R` bounds: `pallet_evm + pallet_currencies + pallet_evm_accounts + pallet_stableswap + pallet_liquidation + pallet_hsm + pallet_gigahdx` (the last added with the lock manager).

## LockManager precompile (`0x0806`)

**Why it exists:** GIGAHDX ([[wiki/pallet-gigahdx\|pallet-gigahdx]], asset 67) is an AAVE aToken minted to the staker. Aave aTokens are freely transferable by default, which would let a staker sell their staked position out from under the HDX lock. `LockableAToken.sol` overrides `freeBalance` as `balanceOf - locked` and sources `locked` from this precompile.

| Item | Value |
|---|---|
| Crate | `precompiles/lock-manager` (`pallet-evm-precompile-lock-manager`, new workspace member in root `Cargo.toml`) |
| Type | `LockManagerPrecompile<Runtime, ExpectedToken>` |
| Address | `0x0000000000000000000000000000000000000806` |
| Selector | `getLockedBalance(address token, address account) → uint256` (`#[precompile::view]`) |
| Backing state | `pallet_gigahdx::Pallet::locked_gigahdx(who)` = `Stakes[who].gigahdx`, `0` when absent |
| `ExpectedToken` | `GigaHdxATokenAddress` = `HydraErc20Mapping::asset_address(GigaHdxAssetIdConst)` (asset 67), resolved **at call time** so it tracks registry remapping |
| Caller check | `token != ExpectedToken::get()` → returns `0` (no revert) |
| Gas | `DbWeight::reads(1)` → `GasWeightMapping::weight_to_gas` → `record_cost` (proof-size aware; deliberately not `record_db_read`) |
| Runtime wiring | `mod.rs` `LOCK_MANAGER` branch; also added to `is_precompile()` |
| Integration tests | `integration-tests/src/gigahdx.rs::lock_manager_precompile_*` |

```rust
// precompiles/lock-manager/src/lib.rs
#[precompile::public("getLockedBalance(address,address)")]
#[precompile::view]
fn get_locked_balance(handle: &mut impl PrecompileHandle, token: Address, account: Address) -> EvmResult<U256> {
    if H160::from(token) != ExpectedToken::get() {
        return Ok(U256::zero());
    }
    // Charge for the `Stakes` StorageMap read via DbWeight (proof-size aware).
    let read_weight = <Runtime as frame_system::Config>::DbWeight::get().reads(1);
    let read_gas = <Runtime as pallet_evm::Config>::GasWeightMapping::weight_to_gas(read_weight);
    handle.record_cost(read_gas)?;

    let account_id = Runtime::AddressMapping::into_account_id(account.into());
    let locked = pallet_gigahdx::Pallet::<Runtime>::locked_gigahdx(&account_id);
    Ok(U256::from(locked))
}
```

Lifecycle of `locked` across the [[wiki/gigahdx\|gigahdx]] flows:

| Flow | Effect on `Stakes[who].gigahdx` | Result at `0x0806` |
|---|---|---|
| `giga_stake` | set to the actual aToken balance delta (AAVE scaled-balance rounding — see `AaveMoneyMarket::supply` in `runtime/hydradx/src/gigahdx.rs`) | `locked == balanceOf` → `free = 0` → user transfers revert |
| `giga_unstake` | **pre-decremented** by the unstaked amount before invoking the money market | `Pool.withdraw → aToken.burn` passes the `freeBalance` check |
| liquidation seize | `on_pre_seize` zeroes it, `on_seize` restores `orig - seized` (`traits/src/gigahdx.rs::Seize`) | Aave's internal aToken transfer is accepted for the duration of the seize |

## MultiCurrency precompile (asset ERC-20)

Selector set lives in `runtime/hydradx/src/evm/erc20_currency.rs::Function`:

| Selector | Notes |
|---|---|
| `name() / symbol() / decimals() / totalSupply() / balanceOf(address)` | view; backed by [[wiki/pallet-asset-registry\|pallet-asset-registry]] + `pallet_currencies` |
| `allowance(address,address)` / `approve(address,uint256)` | allowances stored in `pallet_evm_accounts`; `approve` now emits a real ERC-20 `Approval` log via `emit_approval_log` |
| `transfer(address,uint256)` / `transferFrom(address,address,uint256)` | non-payable |
| `mint(address,uint256)` / `burn(uint256)` | **New** — `INttToken` surface. Gated by `ensure_ntt_minter`: caller must equal `pallet_evm_accounts::NttMinters[asset_id]`, else reverts with the Solidity custom error `CallerNotMinter(address)`. `burn` reverts `InsufficientBalance(uint256,uint256)` on shortfall. There is **no** `setMinter` selector — the minter is set only via the `set_ntt_minter` extrinsic ([[wiki/pallet-evm-accounts\|pallet-evm-accounts]]) |

Over-limit NTT mints take the deposit `PostDeposit` [[wiki/pallet-circuit-breaker\|pallet-circuit-breaker]] path (reserve excess + lockdown), same as XCM — hence the new `pallet_circuit_breaker::Config` bound on `MultiCurrencyPrecompile`.

## Chainlink oracle precompile

AggregatorV3-ish surface: `latestAnswer()`, `latestTimestamp()`, `latestRound()`, `getAnswer(uint256)`, `getTimestamp(uint256)`, `decimals()`. Price resolution branches on the decoded `Source`:

- default → `pallet_ema_oracle::get_price(a, b, period, source)` ([[wiki/pallet-ema-oracle\|pallet-ema-oracle]])
- `OMNIPOOL_SOURCE` (`*b"omnipool"`) with a route → route-derived spot via `RouteProvider`
- **`GIGAHDX_SOURCE`** (`*b"gigahdxs"`, `primitives/src/constants.rs`) → `pallet_gigahdx::Pallet::exchange_rate()`, floored at 1.0 by the pallet so AAVE never reads a sub-1 rate. Adds a `pallet_gigahdx::Config` bound to the precompile.

## Crate / module layout

| Path | Contents |
|---|---|
| `runtime/hydradx/src/evm/precompiles/mod.rs` | `HydraDXPrecompiles`, address consts, `is_precompile`, `Address`, `Bytes`, `Output`, `succeed`/`revert`/`error`/`revert_custom_error`, `emit_approval_log`, `addr()`, `AllowedFlashLoanCallers`, `GigaHdxATokenAddress` |
| `runtime/hydradx/src/evm/precompiles/erc20_mapping.rs` | `HydraErc20Mapping` (`Erc20Mapping` / `Erc20Encoding`), `is_asset_address`, `Erc20MappingApi` runtime API, `SetCodeForErc20Precompile` |
| `runtime/hydradx/src/evm/precompiles/multicurrency.rs` | `MultiCurrencyPrecompile`, `ensure_ntt_minter` |
| `runtime/hydradx/src/evm/precompiles/chainlink_adapter.rs` | `ChainlinkOraclePrecompile`, `encode_oracle_address` / `decode_oracle_address`, `is_oracle_address` |
| `runtime/hydradx/src/evm/precompiles/{costs,handle,substrate}.rs` | gas cost helpers (`log_costs`), handle wrappers, `RuntimeHelper` |
| `precompiles/call-permit/` | `CallPermitPrecompile` — `dispatch(address,address,uint256,bytes,uint64,uint256,uint8,bytes32,bytes32)`, `nonces(address)`, `DOMAIN_SEPARATOR()` |
| `precompiles/flash-loan/` | `FlashLoanReceiverPrecompile<R, AllowedCallers>` — `onFlashLoan(address,address,uint256,uint256,bytes)` |
| `precompiles/lock-manager/` | `LockManagerPrecompile<R, ExpectedToken>` — `getLockedBalance(address,address)` |
| `precompiles/utils/` | `precompile-utils` — `#[precompile_utils::precompile]` macro, solidity codec, `EvmDataWriter`, test harness |

There is **no** `precompiles/dispatch/` crate — Dispatch comes from upstream `pallet_evm_precompile_dispatch`.

## `Erc20Mapping` / `Erc20Encoding` traits

Defined in `traits/src/evm.rs`:

- `Erc20Mapping::asset_address(AssetId) -> EvmAddress` — registry `contract_address` (bound ERC-20) first, else `encode_evm_address`
- `Erc20Mapping::address_to_asset(EvmAddress) -> Option<AssetId>` — decode prefix, else `AccountKey20` location lookup
- `Erc20Encoding::{encode_evm_address, decode_evm_address}` — pure prefix codec
- `Erc20Inspect::{contract_address, is_atoken}` — bound-contract / aToken inspection
- `BoundErc20` (`traits/src/registry.rs`) — registry-bound real contract addresses

## Synthetic EVM logs (adjacent)

`emit_approval_log` in `precompiles/mod.rs` reuses `APPROVAL_TOPIC` / `encode_u256_be` / `h160_to_h256` from `runtime/hydradx/src/evm/synthetic_logs.rs`. The synthetic-log machinery itself (`evm/synthetic_logs.rs`, `evm/event_logs.rs` — substrate events → synthetic ETH txs/logs, assembled node-side) is **not** a precompile and is documented in [[wiki/hydration-runtime\|hydration-runtime]].

## Gotchas

- **`is_precompile()` omits CallPermit and FlashLoanReceiver.** It returns true only for `DISPATCH_ADDR`, `LOCK_MANAGER`, asset addresses and `0x01..=0x09` — so the DELEGATECALL/CALLCODE guard and `pallet_evm`'s precompile checks do not cover `0x080a` / `0x090a`, nor oracle addresses.
- **Dispatch nonce clamp.** After `Dispatch::execute`, the runtime forces the caller's `frame_system` nonce to `original + 1` when the dispatched call bumped it more than once — keeps the EVM and Substrate nonce views in sync ([[wiki/pallet-evm-accounts\|pallet-evm-accounts]] `AddressMapping` derives the origin).
- **LockManager returns 0, it does not revert,** for a non-GIGAHDX `token`. A contract reading `0x0806` for the wrong aToken silently sees "nothing locked" — intentional, so an unrelated `LockableAToken` cannot over-lock holders based on gigahdx-stake state.
- **`GigaHdxATokenAddress` is resolved per call**, not cached — an asset-registry remap of asset 67 or a bound-contract change takes effect immediately.
- **Flash-loan callers are dynamic:** `AllowedFlashLoanCallers` reads `pallet_hsm::flash_minter()`; when unset the precompile is effectively unavailable (empty allow-list + warn log). See [[wiki/pallet-hsm\|pallet-hsm]].
- **CallPermit domain separator** = `keccak(EIP712Domain(...))` over name `"Call Permit Precompile"`, version `"1"`, `Runtime::ChainId::get()` and the precompile address. `ChainId` is **chain state** (`pallet_evm_chain_id`, runtime index 91), not a compile-time constant — the in-repo chainspec genesis sets `2_222_222` (`node/src/chain_spec/mod.rs`). Read it from chain rather than hardcoding.
- **`WETH_ASSET_ID` is now the constant `20`** (`evm/mod.rs`); the old XCM-location lookup (`weth_asset_location()`, Moonbeam para 2004) was removed so the gas asset can't be zeroed by a registry repoint.
- **Asset ERC-20 addresses can't collide with deployed contracts** (16-byte `…0001` prefix), and can't collide with oracle addresses (byte 2 is `0x00` for assets vs `0x01` for oracles). Never deploy into either subspace.
- **`0x00FF…` is not an oracle prefix** (a stale claim in earlier revisions of this page). Oracles are `0x000001…`; `0xFFFF…FF` is `erc20_currency::HOLDING_ADDRESS`, the ERC-20 holding account.
- **Gas accounting** — every custom precompile must `record_cost()` for its own reads/logs; `WEIGHT_PER_GAS = WEIGHT_REF_TIME_PER_SECOND / GAS_PER_SECOND` with `GAS_PER_SECOND = 40_000_000` (`evm/mod.rs`).

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hydration-runtime\|hydration-runtime]]
- [[wiki/pallet-gigahdx\|pallet-gigahdx]]
- [[wiki/gigahdx\|gigahdx]]
- [[wiki/pallet-evm-accounts\|pallet-evm-accounts]]
- [[wiki/pallet-hsm\|pallet-hsm]]
