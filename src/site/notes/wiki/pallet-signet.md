---
{"dg-publish":true,"permalink":"/wiki/pallet-signet/","title":"pallet-signet","tags":["signet","mpc","signing","bridge","evm","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-signet","repo":"hydration-node","paths":["pallets/signet/src/lib.rs","pallets/signet/src/types.rs","pallets/signet/src/weights.rs"],"symbols":["Pallet","Config","set_config","withdraw_funds","sign","sign_bidirectional","respond","respond_error","respond_bidirectional","pause","unpause","SignetConfig","SignetConfigData","Signature","AffinePoint","SerializationFormat","ErrorResponse"],"traits_impl":[],"depends_on":[],"runtime_index":84,"tags":["signet","mpc","signing","bridge","evm","runtime","rust","substrate"],"last_updated":"2026-05-14"}}
---


# pallet-signet

**TL;DR:** Event-driven coordination layer for an off-chain MPC signing service. The pallet **only emits events and accepts response submissions**; it does NOT persist requests on-chain. Users charge a deposit to request a signature; an off-chain responder watches events, runs the MPC protocol, and submits the signature back via `respond` / `respond_bidirectional`. A single `SignetConfig` storage item holds pause + deposit + chain-id config. Runtime index = 84.

## Role

Hydration needs to issue signed EVM transactions from MPC-managed keys (used by [[wiki/pallet-dispenser\|pallet-dispenser]] for the Ethereum gas faucet, and other cross-chain operations) without any single party holding the private key. SigNet (off-chain) is the signing service; this pallet is the on-chain bridge:

1. User calls `sign` or `sign_bidirectional`, paying a deposit. The pallet emits a `SignatureRequested` / `SignBidirectionalRequested` event with the full payload.
2. Off-chain SigNet operator observes the event, runs MPC, and submits the result via `respond` (batch) or `respond_bidirectional` (single, with a serialised output).
3. The response emits `SignatureResponded` / `RespondBidirectionalEvent` / `SignatureError`. Consumers (other pallets, off-chain workers) match by `request_id`.

Note: requests themselves are **not stored on-chain**. The pallet does not enforce uniqueness or TTL — replay protection lives in callers like [[wiki/pallet-dispenser\|pallet-dispenser]] (`UsedRequestIds`).

## Config trait

```rust
// pallets/signet/src/lib.rs
pub trait Config: frame_system::Config<RuntimeEvent: From<Event<Self>>> {
    /// Origin allowed to call set_config / withdraw_funds / pause / unpause.
    type UpdateOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    /// Currency for deposits and fees.
    type Currency: Currency<Self::AccountId>;
    /// Pallet ID used to derive the deposit-holding account.
    #[pallet::constant] type PalletId: Get<PalletId>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Description |
|---|---|---|
| `SignetConfig` | `StorageValue<_, SignetConfigData<BalanceOf<T>>, OptionQuery>` | Sole storage item — `None` means the pallet is unconfigured and all user-facing extrinsics will fail with `NotConfigured` |

```rust
pub struct SignetConfigData<Balance> {
    pub paused: bool,
    pub signature_deposit: Balance,
    pub max_chain_id_length: u32,
    pub max_evm_data_length: u32,
    pub chain_id: BoundedVec<u8, ConstU32<128>>, // CAIP-2 chain id
}
```

## Events

- `ConfigUpdated { signature_deposit, max_chain_id_length, max_evm_data_length, chain_id }`
- `Paused`, `Unpaused`
- `FundsWithdrawn { amount, recipient }`
- `SignatureRequested { sender, payload: [u8; 32], key_version, deposit, chain_id, path, algo, dest, params }`
- `SignBidirectionalRequested { sender, serialized_transaction, caip2_id, key_version, deposit, path, algo, dest, params, output_deserialization_schema, respond_serialization_schema }`
- `SignatureResponded { request_id, responder, signature }`
- `SignatureError { request_id, responder, error }`
- `RespondBidirectionalEvent { request_id, responder, serialized_output, signature }`

## Errors

`NotConfigured`, `Paused`, `InsufficientFunds`, `InvalidTransaction`, `InvalidInputLength`, `DataTooLong`, `InvalidAddress`, `InvalidGasPrice`.

## Extrinsics

| call_index | Name | Origin | Description |
|---|---|---|---|
| 0 | `set_config` | `UpdateOrigin` | Set / update `SignetConfigData` (deposit, chain_id, length limits). Preserves the existing `paused` flag |
| 1 | `withdraw_funds` | `UpdateOrigin` | Transfer accumulated deposits from the pallet account to a recipient |
| 2 | `sign` | Signed | User pays `signature_deposit`; emits `SignatureRequested` with a 32-byte payload + key_version + chain_id + arbitrary `path`/`algo`/`dest`/`params` bytes |
| 3 | `sign_bidirectional` | Signed | User pays `signature_deposit`; emits `SignBidirectionalRequested` carrying a serialised EVM transaction + CAIP-2 id + two schemas (deserialisation + respond serialisation) |
| 4 | `respond` | Signed (responder) | Batch: emit `SignatureResponded` for each `(request_id, Signature)` pair. Lengths must match (`InvalidInputLength`) |
| 5 | `respond_error` | Signed (responder) | Batch: emit `SignatureError` for each `ErrorResponse` |
| 6 | `respond_bidirectional` | Signed (responder) | Emit `RespondBidirectionalEvent` with a single `request_id`, serialised output, and signature |
| 7 | `pause` | `UpdateOrigin` | Set `paused = true` |
| 8 | `unpause` | `UpdateOrigin` | Set `paused = false` |

## Key types

```rust
pub struct AffinePoint { pub x: [u8; 32], pub y: [u8; 32] }
pub struct Signature   { pub big_r: AffinePoint, pub s: [u8; 32], pub recovery_id: u8 }
pub struct ErrorResponse {
    pub request_id: [u8; 32],
    pub error_message: BoundedVec<u8, ConstU32<MAX_ERROR_MESSAGE_LENGTH>>,
}
pub enum SerializationFormat { Borsh = 0, AbiJson = 1 }
```

Hard-coded bounds in `pallets/signet/src/lib.rs`: `MAX_PATH_LENGTH=256`, `MAX_ALGO_LENGTH=32`, `MAX_DEST_LENGTH=64`, `MAX_PARAMS_LENGTH=1024`, `MAX_TRANSACTION_LENGTH=65536`, `MAX_SERIALIZED_OUTPUT_LENGTH=65536`, `MAX_SCHEMA_LENGTH=4096`, `MAX_ERROR_MESSAGE_LENGTH=1024`, `MAX_BATCH_SIZE=100`, `MAX_CHAIN_ID_LENGTH=128`.

## Helper: `build_evm_tx`

`Pallet::build_evm_tx(...)` builds an EIP-1559 EVM transaction (RLP-encoded `EIP1559TransactionMessage`) used by callers like [[wiki/pallet-dispenser\|pallet-dispenser]] to assemble the `serialized_transaction` they pass to `sign_bidirectional`.

## Integration

- **Traits implemented:** none external
- **Traits consumed:** `Currency` (deposits), `EnsureOrigin` (UpdateOrigin)
- **Pallets depended on:** none. Callers include [[wiki/pallet-dispenser\|pallet-dispenser]] (gas faucet) and any future bridge/relay pallet that needs MPC signatures.

## Gotchas

- **Stateless requests.** The pallet does not store pending requests, used request IDs, or expiry — everything lives in events. Replay protection is the caller's responsibility (see `UsedRequestIds` in [[wiki/pallet-dispenser\|pallet-dispenser]]).
- **Anyone can call `respond`.** The pallet does NOT verify that the responder is the legitimate MPC operator — any signed account can emit `SignatureResponded`. Downstream consumers must verify the submitted signature against the registered MPC public key off-chain or in the consuming pallet.
- **`SignetConfig` is the only storage**. If it is `None`, every user-facing call (`sign`, `sign_bidirectional`) fails with `NotConfigured`. `set_config` must be called once via `UpdateOrigin` (Root / TC majority) before the pallet is usable.
- **Deposit per request.** Each `sign` / `sign_bidirectional` transfers `signature_deposit` from the requester to the pallet account. Funds accumulate until `withdraw_funds` is invoked.
- **Pause path.** `pause` blocks new `sign` / `sign_bidirectional` calls but does NOT block responders — `respond` / `respond_bidirectional` continue to work, so in-flight requests can still settle.
- **Previous wiki revision** documented a `Keys` registry, `PendingRequests`, `CompletedRequests`, `register_key`, `enable_key`, `submit_signature`, `expire_requests` and similar APIs — none of those exist. The pallet was substantially refactored: it is now event-only.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/pallet-dispenser\|pallet-dispenser]]
