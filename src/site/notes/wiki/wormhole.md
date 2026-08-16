---
{"dg-publish":true,"permalink":"/wiki/wormhole/","title":"Wormhole","tags":["bridge","cross-chain","ntt","ethereum","solana","sui"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"Wormhole","tags":["bridge","cross-chain","ntt","ethereum","solana","sui"],"source_count":1,"last_updated":"2026-08-15"}}
---


# Wormhole

**TL;DR:** Wormhole is the cross-chain bridge [[wiki/hydration\|hydration]] uses for connectivity beyond [[wiki/polkadot\|polkadot]], complementing [[wiki/snowbridge\|snowbridge]]. **Since Aug 2026 it runs on NTT (Native Token Transfers) — Hydration ↔ Ethereum / Base / Solana / Sui direct.** The old MRL model (Hydration → Moonbeam → Ethereum) and Moonbeam itself are gone.

## MRL → NTT

| Was (MRL) | Is (NTT) |
|---|---|
| Hydration → **Moonbeam** hop → Ethereum, XCM `Transact` payload | direct Hydration ↔ Ethereum / Base / Solana / Sui |
| TokenBridge / TokenRelayer contracts, `utils/mrl.ts` | `NttManager` + `WormholeTransceiver`, `xc-core/bridge/ntt.ts` |
| `Tag.Mrl` | `Tag.Ntt`, `Tag.NttExecutor` |
| asset keys `*_mwh` (moonbeam-wormhole) | asset keys `*_wh` |
| `HydrationMrlFeeValidation` (GLMR balance) | `NttRateLimitValidation` |

Hydration is now a **first-class Wormhole chain: wormhole id `73`** (EVM chain id `222222`), registered in `@wormhole-foundation/sdk-base`. Moonbeam was removed as a chain from [[wiki/xc-cfg\|xc-cfg]] altogether.

Each NTT pair carries two routes: plain (`NttManager.transfer`, user self-redeems) and executor (`Tag.NttExecutor`, the Wormhole Executor service delivers). Full mechanics, per-chain deployments and the Hydration-specific u128 approve gotcha: [[wiki/xc-cfg\|xc-cfg]] and [[wiki/xc-core\|xc-core]].

Chain side: [[wiki/pallet-evm-accounts\|pallet-evm-accounts]] keeps an NTT minter registry (`NttMinters`, `set_ntt_minter`, `NttEmergencyOrigin`). UI side: `transactions/utils/wormhole.ts` was deleted, `XcmTag.NttExecutor` added to `BRIDGE_PROVIDER_TAGS`, with `useNttInbound/OutboundLimit` hooks ([[wiki/hydration-ui-api\|hydration-ui-api]], [[wiki/hydration-ui-modules\|hydration-ui-modules]]).

## Sources

- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]]
