---
{"dg-publish":true,"permalink":"/wiki/pallet-claims/","title":"pallet-claims","tags":["claims","ethereum","genesis","migration","runtime","rust","substrate"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"pallet","title":"pallet-claims","repo":"hydration-node","paths":["pallets/claims/src/lib.rs","pallets/claims/src/traits.rs"],"symbols":["Pallet","Config","claim","Claims","EthereumAddress","EcdsaSignature","ValidateClaim","MESSAGE_PREFIX"],"traits_impl":["TransactionExtension"],"depends_on":[],"runtime_index":53,"tags":["claims","ethereum","genesis","migration","runtime","rust","substrate"],"last_updated":"2026-04-13"}}
---


# pallet-claims

**TL;DR:** Lets holders of pre-genesis HDX on Ethereum claim their tokens on Hydration by presenting an Ethereum ECDSA signature over their Substrate account hex. Zero-cost call (`Pays::No`). Runtime index = 53.

## Role

One-time HDX distribution bridge from Ethereum → Hydration. Maps `EthereumAddress → Balance` from genesis config; each claim recovers the signer from an ECDSA signature, matches it to the map, and mints the balance to the caller's AccountId.

## Config trait (excerpt)

```rust
// pallets/claims/src/lib.rs
pub trait Config: frame_system::Config {
    type Prefix: Get<&'static [u8]>;
    type WeightInfo: WeightInfo;
    type Currency: Currency<Self::AccountId>;
    type CurrencyBalance: From<Balance> + Into<Currency::Balance>;
}
```

## Storage

| Name | Kind | Key → Value |
|------|------|-------------|
| `Claims` | StorageMap | `EthereumAddress → Balance` (ValueQuery, defaults to 0 after claim) |

## Events

`Claim` (account, ethereum_address, balance).

## Errors

`InvalidEthereumSignature`, `NoClaimOrAlreadyClaimed`, `BalanceOverflow`.

## Extrinsics

| Name | Description |
|------|-------------|
| `claim` | Caller submits ECDSA signature → balance minted on success (fee-free) |

## Hooks

None.

## Integration

- **Traits implemented:** `TransactionExtension` via `ValidateClaim` signed extension (pre-validates the claim in mempool before dispatch)
- **Traits consumed:** `Currency` (for `deposit_creating`)
- **Pallets depended on:** none

## Key code

```rust
// pallets/claims/src/lib.rs
fn validate_claim(
    who: &T::AccountId,
    signature: &EcdsaSignature,
) -> Result<(BalanceOf<T>, EthereumAddress), Error<T>> {
    let sender_hex = who.using_encoded(to_ascii_hex); // account → hex ASCII
    let signer = signature.recover(&sender_hex, T::Prefix::get());
    match signer {
        Some(address) => {
            let due = Claims::<T>::get(address);
            ensure!(due != Zero::zero(), Error::<T>::NoClaimOrAlreadyClaimed);
            Ok((due, address))
        }
        None => Err(Error::<T>::InvalidEthereumSignature),
    }
}
```

## Gotchas

- Signature is over `Prefix || ascii_hex(account_id)` — user signs with MetaMask/etherjs personal_sign; the prefix is `"I hereby claim all my HDX tokens to wallet:"` in Hydration's runtime.
- `ValidateClaim` TransactionExtension runs in the pool before inclusion — invalid signatures are rejected free.
- On success the claim is zeroed → no double-claims.
- `Pays::No` — users never pay to claim.
- Genesis config pre-populates the `Claims` map; no way to add claims post-genesis.
- Uses `secp256k1_ecdsa_recover` — not sr25519/ed25519.

## Sources

- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
