---
{"dg-publish":true,"permalink":"/wiki/xc-sdk/","title":"xc-sdk","tags":["sdk","cross-chain","wallet","transfers"],"dg-note-properties":{"type":"entity","entity_kind":"product","title":"xc-sdk","tags":["sdk","cross-chain","wallet","transfers"],"source_count":1,"last_updated":"2026-04-13"}}
---


# xc-sdk

**TL;DR:** `@galacticcouncil/xc-sdk` provides the wallet interface and execution layer for cross-chain transfers, with platform adapters for Substrate, EVM, Solana, and Sui.

## Wallet API

```ts
const transfer = await wallet.transfer(asset, srcAddr, srcChain, dstAddr, dstChain);
await transfer.estimateFee(amount);
await transfer.validate();
const call = await transfer.buildCall(amount);
```

The Wallet uses `ConfigBuilder` to find valid routes, creates platform adapters for both chains, and fetches balances, fees, and fee swap opportunities in parallel.

## Platform Adapters

Each platform handles chain-specific transaction building:
- **SubstratePlatform** — builds Substrate extrinsics
- **EvmPlatform** — builds EVM contract calls (via viem)
- **SolanaPlatform** — builds Solana programs
- **SuiPlatform** — builds Sui moves

Returns a `Call` object with: `from`, `data` (hex-encoded), `type` (Extrinsic/Contract/Program/Move), and `dryRun()`.

## Fee Handling

Fees include source chain transaction fee, destination execution fee, and optional fee swaps (converting between fee assets using the HydrationDex from [[wiki/xc-cfg\|xc-cfg]]).

## Transfer Validation

`TransferValidator` checks: balance sufficiency, fee adequacy, minimum amount requirements, and route availability.

## Signing

Platform-specific signers: SubstrateSigner (polkadot-api), EvmSigner (viem WalletClient), SolanaSigner, SuiSigner. The SDK builds transactions but does not sign — the consumer provides the signer.

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
