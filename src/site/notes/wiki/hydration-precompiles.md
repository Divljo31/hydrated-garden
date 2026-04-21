---
{"dg-publish":true,"permalink":"/wiki/hydration-precompiles/","title":"hydration-precompiles","tags":["evm","precompiles","frontier","erc20","runtime","rust"],"dg-note-properties":{"type":"runtime","title":"hydration-precompiles","repo":"hydration-node","paths":["runtime/hydradx/src/evm/precompiles.rs","precompiles/call-permit/src/lib.rs","precompiles/flashloan-receiver/src/lib.rs","precompiles/dispatch/src/lib.rs"],"symbols":["HydraDXPrecompiles","CallPermitPrecompile","FlashLoanReceiverPrecompile","DispatchPrecompile","Erc20Mapping","AssetId"],"traits_impl":["PrecompileSet"],"depends_on":["pallet-evm","pallet-currencies","pallet-evm-accounts"],"tags":["evm","precompiles","frontier","erc20","runtime","rust"],"last_updated":"2026-04-13"}}
---


# hydration-precompiles

**TL;DR:** Hydration's EVM precompile set — the standard Frontier/Ethereum precompiles at `0x01..=0x09`, plus Hydration-specific precompiles for dispatching Substrate calls, ERC-20 wrapping of every registered asset, Chainlink-style oracles, call permits, and flash-loan receiver. Wired into `pallet_evm::Config::PrecompilesType`.

## Standard Ethereum precompiles

| Address | Name |
|---------|------|
| `0x0000000000000000000000000000000000000001` | ECRecover |
| `0x0000000000000000000000000000000000000002` | SHA256 |
| `0x0000000000000000000000000000000000000003` | RIPEMD160 |
| `0x0000000000000000000000000000000000000004` | Identity |
| `0x0000000000000000000000000000000000000005` | Modexp |
| `0x0000000000000000000000000000000000000006` | BN128Add |
| `0x0000000000000000000000000000000000000007` | BN128Mul |
| `0x0000000000000000000000000000000000000008` | BN128Pairing |
| `0x0000000000000000000000000000000000000009` | Blake2F |

## Hydration-specific precompiles

| Address | Name | Purpose |
|---------|------|---------|
| `0x000000000000000000000000000000000000080a` (2058) | CallPermit | EIP-712-style typed call permit — lets an EVM account pre-authorize another to dispatch a call on its behalf (permit + relayer pattern) |
| `0x000000000000000000000000000000000000090a` (2314) | FlashLoanReceiver | ERC-3156-compatible flashloan callback glue for cross-pallet flashloans |
| `0x0000000000000000000000000000000000000401` (1025) | Dispatch | Executes a Substrate runtime call from EVM context — the main Substrate↔EVM bridge |

## Dynamic-address precompiles

Hydration encodes *every registered `AssetId`* as a dedicated ERC-20 precompile address, so EVM contracts can trade/transfer native Hydration assets as if they were standard ERC-20s.

| Prefix | Purpose | Decoding |
|--------|---------|----------|
| `0x0000000000000000000000000000000100......` | Asset ERC-20 | last 4 bytes big-endian = `AssetId` |
| `0x00000000000000000000000000000000000FF...` | Chainlink-style oracle | asset-specific oracle interfaces |

```rust
// pallets/evm-accounts / traits - Erc20Encoding pattern:
// AssetId = u32; ERC-20 address = H160:
//   bytes 0..=15 = 0x000100... (20-byte prefix filled with leading zeros + 0x01 marker)
//   bytes 16..=19 = AssetId as big-endian u32
```

Example: AssetId=5 (DAI) → `0x0000000000000000000000000000000100000005`.

## HydraDXPrecompiles<Runtime>

```rust
// runtime/hydradx/src/evm/precompiles.rs (abridged)
pub struct HydraDXPrecompiles<Runtime>(PhantomData<Runtime>);

impl<R> PrecompileSet for HydraDXPrecompiles<R> where R: pallet_evm::Config + ... {
    fn execute(&self, handle: &mut impl PrecompileHandle) -> Option<PrecompileResult> {
        match handle.code_address() {
            a if a == H160::from_low_u64_be(1) => Some(ECRecover::execute(handle)),
            // ... standard 1..=9 ...
            a if a == H160::from_low_u64_be(1025) => Some(Dispatch::<R>::execute(handle)),
            a if a == H160::from_low_u64_be(2058) => Some(CallPermit::<R>::execute(handle)),
            a if a == H160::from_low_u64_be(2314) => Some(FlashLoanReceiver::<R>::execute(handle)),
            addr if is_asset_address(&addr) => Some(Erc20AssetAdapter::<R>::execute(handle, addr)),
            _ => None,
        }
    }
    fn is_precompile(&self, a: H160, _: u64) -> IsPrecompileResult { ... }
}
```

## Crate layout

- `precompiles/call-permit/` — `CallPermitPrecompile` (EIP-712 encoding, signature recovery)
- `precompiles/flashloan-receiver/` — `FlashLoanReceiverPrecompile` (wraps Aave-style flashloan callback)
- `precompiles/dispatch/` — `DispatchPrecompile` (decodes Substrate `RuntimeCall` bytes, dispatches via `AddressMapping`-derived Substrate origin)

## Erc20Mapping / Erc20Encoding traits

Defined in `traits/` (see [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]):
- `Erc20Mapping::asset_id_to_address(AssetId) -> H160`
- `Erc20Mapping::address_to_asset_id(H160) -> Option<AssetId>`
- `Erc20Encoding` — low-level encoding/decoding (inverse of each other)
- `Erc20Inspect` — read-only ERC-20 interface (name, symbol, decimals, totalSupply)

## Gotchas

- **Dispatch precompile (0x0401 / 1025)**: the `origin` is derived from the EVM caller address via `pallet_evm_accounts::AddressMapping`. If the EVM account has a bound Substrate account ([[wiki/pallet-evm-accounts\|pallet-evm-accounts]]), the dispatched call runs as that account; otherwise it uses the truncated address representation.
- **Asset ERC-20 addresses**: the 20-byte prefix pattern means asset IDs can never collide with real contract addresses (they start with `0x000100...`). Protocol must never deploy a contract to that subspace.
- **CallPermit**: uses EIP-712 domain separator including chain_id (222_222) — mismatches will cause signature recovery failures.
- **Flashloan receiver**: only works in conjunction with Aave-style callers; not all Hydration flashloan paths go through it (native Substrate flashloans are separate).
- **Gas accounting**: each custom precompile must record `record_cost()` for its work; under-recording lets EVM callers DoS. Standard ones use FRAME weights mapped via `WEIGHT_PER_GAS = 25_000`.
- **ChainId = 222_222** — EVM tools (wallet RPCs, etherscan mirrors) must be configured to this chain id.
- **Oracle precompile addresses** (`0x00FF...`) wrap the [[wiki/pallet-ema-oracle\|pallet-ema-oracle]] into a Chainlink-like AggregatorV3 interface for EVM consumers.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/hydration-runtime\|hydration-runtime]]
- [[wiki/pallet-evm-accounts\|pallet-evm-accounts]]
