---
{"dg-publish":true,"permalink":"/wiki/pallet-signet/","title":"pallet-signet","tags":["signet","mpc","signing","bridge","evm","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-signet","repo":"hydration-node","paths":["pallets/signet/src/lib.rs","pallets/signet/src/types.rs"],"symbols":["Pallet","Config","sign_bidirectional","SignRequest","SigNetKeys","KeyId","WithdrawalRequest","RequestId"],"traits_impl":[],"depends_on":[],"runtime_index":84,"tags":["signet","mpc","signing","bridge","evm","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-signet

**TL;DR:** On-chain half of Hydration's SigNet MPC (multi-party computation) signing service. Manages a registry of shared signing keys and receives signing requests that the off-chain SigNet operator fulfills by producing a signed EVM transaction. Runtime index = 84.

## Role

Hydration needs to send signed EVM transactions from shared keys (Dispenser faucet, bridge relayers, Aave managers) without giving any single party the private key. SigNet is the off-chain MPC service; this pallet is the coordination layer:
- Registers public keys (and their EVM addresses).
- Accepts signing requests from authorized pallets ([[wiki/pallet-dispenser\|pallet-dispenser]] etc.) and emits a `SignRequested` event.
- Off-chain workers pick up the event, run MPC to produce a signature, and submit the result back on-chain.

## Config trait (excerpt)

```rust
// pallets/signet/src/lib.rs
pub trait Config: frame_system::Config {
    type RuntimeEvent: ...;
    type AuthorityOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type PalletId: Get<PalletId>;
    #[pallet::constant] type MaxKeyCount: Get<u32>;
    #[pallet::constant] type MaxRequestDataLen: Get<u32>;
    #[pallet::constant] type RequestTTL: Get<BlockNumberFor<Self>>;
    type WeightInfo: WeightInfo;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Keys` | StorageMap | `KeyId → SigNetKey { pubkey_hash, evm_address, enabled }` |
| `PendingRequests` | StorageMap | `RequestId → SignRequest { key_id, payload, requester, expires_at }` |
| `CompletedRequests` | StorageMap | `RequestId → Signature (or result)` |

## Events

`KeyRegistered`, `KeyEnabled`, `KeyDisabled`, `SignRequested`, `SignCompleted`, `SignFailed`, `RequestExpired`.

## Errors

`KeyNotFound`, `KeyDisabled`, `RequestNotFound`, `RequestExpired`, `DuplicateRequest`, `TooManyKeys`, `PayloadTooLarge`, `Unauthorized`.

## Extrinsics

| Name | Description |
|------|-------------|
| `register_key` | `AuthorityOrigin` adds a new MPC key entry (pubkey hash + EVM address) |
| `enable_key` / `disable_key` | Authority toggles a key's availability |
| `sign_bidirectional` | Callable by authorized callers (pallet-dispenser etc.); enqueues a `SignRequest` and emits `SignRequested` |
| `submit_signature` | SigNet operator submits the completed signature; pallet verifies + stores + emits `SignCompleted` |
| `expire_requests` | Housekeeping — removes requests past `RequestTTL` (usually called automatically) |

## Hooks

`on_initialize` may sweep expired requests up to a bounded count per block.

## Integration

- **Traits implemented:** none external (exposes `sign_bidirectional` as a dispatchable for other pallets)
- **Traits consumed:** `EnsureOrigin` (for authority actions)
- **Pallets depended on:** none directly; [[wiki/pallet-dispenser\|pallet-dispenser]] and bridge relays call `sign_bidirectional`

## Key type

```rust
// pallets/signet/src/types.rs
pub struct SignRequest<AccountId, BlockNumber> {
    pub key_id: KeyId,
    pub payload: BoundedVec<u8, MaxRequestDataLen>,
    pub requester: AccountId,
    pub created_at: BlockNumber,
}

pub struct SigNetKey {
    pub pubkey_hash: H256,
    pub evm_address: H160,
    pub enabled: bool,
}
```

## Gotchas

- Signatures themselves are produced **off-chain** by the MPC operator — the pallet only coordinates. No on-chain key material.
- `RequestTTL` ensures stale requests don't accumulate; consumers should check `CompletedRequests` within the TTL window.
- `submit_signature` must verify the signature matches the registered `evm_address` before accepting.
- `AuthorityOrigin` is typically a governance-controlled origin (Root or Technical Committee), not any signed user.
- Request IDs are deterministic (usually hash of caller + payload + nonce) to enable replay protection in consuming pallets like [[wiki/pallet-dispenser\|pallet-dispenser]] (`UsedRequestIds`).
- This pallet is tightly coupled with the off-chain SigNet operator — offline operator = stuck faucet/bridge.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/pallet-dispenser\|pallet-dispenser]]
