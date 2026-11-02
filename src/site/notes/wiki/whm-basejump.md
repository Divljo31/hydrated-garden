---
{"dg-publish":true,"permalink":"/wiki/whm-basejump/","title":"WHM — Basejump (instant Ethereum → Hydration USDC)","tags":["whm","basejump","wormhole","ntt","usdc","bridge","liquidity-pool","security"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"WHM — Basejump (instant Ethereum → Hydration USDC)","repo":"whm","paths":["docs/basejump/spec.md","docs/basejump/schema.md","docs/basejump/indexer.md","contracts/src/basejump/BasejumpEmitter.sol","contracts/src/basejump/BasejumpReceiver.sol","contracts/src/basejump/BasejumpLanding.sol","contracts/src/basejump/interfaces/IBasejumpPayload.sol","deployments/prod/basejump-ethereum.json","deployments/prod/basejump-landing-upgrade.json","agents/relayer/src/apps/basejump","chopsticks/probes"],"key_exports":["BasejumpEmitter","BasejumpReceiver","BasejumpLanding","bridgeViaWormhole","completeTransfer","transfer","fulfillPending","setAuthorizedBridge","setDestAsset","withdraw","assetFee","nttManagerFor","TransferPayload"],"tags":["whm","basejump","wormhole","ntt","usdc","bridge","liquidity-pool","security"],"last_updated":"2026-11-02"}}
---


# WHM — Basejump

**TL;DR:** Direct, inbound-only fast route Ethereum → Hydration. One `bridgeViaWormhole` call does two things: an **NTT settlement** of the gross amount to the landing pool (~13 min, guardian finality) and a **fast Wormhole message** (consistency 200, ~2 min) for the net amount, which `BasejumpReceiver` turns into a payout from the pre-funded `BasejumpLanding` pool on Hydration. Fee = gross − net (`assetFee`, 0.1 USDC), retained in the pool. Live token: **USDC → asset 21**. Went live with referendum **#404** (2026-09-16, `setAuthorizedBridge`).

## Contracts

| Contract | Chain | Address | Role |
|---|---|---|---|
| `BasejumpEmitter` (UUPS) | Ethereum | `0xa72e2bf29c840eb93adbb9ee1aa41580f01c9944` | `bridgeViaWormhole(asset, amount, recipient, data)` — settle then publish |
| `BasejumpReceiver` (UUPS, `MessageReceiver`) | Hydration | `0x35bf3a1b9ac564c8f66c97cea1ee410cd3f97c8a` | `completeTransfer(vaa)` → `landing.transfer(...)` |
| `BasejumpLanding` (UUPS) | Hydration | `0x70e9b12c3b19cb5f0e59984a5866278ab69df976` (impl upgraded 2026-09-17 → `0xd04b8996…b6f`) | pool; pays via `0x0401` dispatch → `currencies.transfer`; queues FIFO on shortfall |
| NTT manager USDC | Ethereum | `0x447b2c7485A3d6813F8197E605b10BcCD8dd8398` (LOCKING) | settlement rail |
| NTT manager USDC | Hydration | `0xEcEab64542A875C4472671D9Ed1E690cdD4e28fC` (BURNING) | mints asset 21 to the landing |

Rate limit (read live, per spec): 100,000 USDC outbound per 24 h sliding window on the Ethereum manager.

## Flow

1. **Source:** reject zero / unset landing / unset `nttManagerFor[asset]`; pull with balance-delta; `INttManager.transfer{value: deliveryPrice}(gross, 73, landing)` (3-arg overload, `shouldQueue = false`) — a rate-limit breach or paused rail **reverts before any message exists**; then publish `TransferPayload{sourceAsset, net, recipient(AccountId32), transferSequence, data}`.
2. **Receiver:** verify VAA, replay check, one authorized emitter per source chain, `landing.transfer(sourceAsset, amount, recipient, data)` — `data` is dropped today.
3. **Landing:** `destAssetFor[sourceAsset]`; pay if balance suffices, else queue. **`fulfillPending()` has no keeper** — alarm on `pendingTail − pendingHead > 0`.

## Trust & admin

| Component | Moves funds? | Owner / upgrade authority | Notes |
|---|---|---|---|
| `BasejumpEmitter` (Eth) | pulls user USDC into NTT | **TC Safe `0xD557…81E7`** (4-of-7) — `setLanding`, `setNttManager`, `setAssetFee`, upgrade | retargeting the landing on the emitter alone would send settlements elsewhere |
| `BasejumpReceiver` (Hydration) | instructs payouts | **`EmergencyAdminAccount` `0xaa7e…aa7e1`** — `setLanding`, `setAuthorizedEmitter`, upgrade | i.e. TC majority via `dispatch_as_emergency_admin` ([[wiki/pallet-dispatcher\|pallet-dispatcher]]) |
| `BasejumpLanding` (Hydration) | **holds the USDC pool** | **`EmergencyAdminAccount`** — `setAuthorizedBridge`, `setDestAsset`, **`withdraw(asset, amount, to)`**, upgrade | `withdraw` is the emergency drain; `setAuthorizedBridge(receiver,false)` is the pause |
| Wormhole guardians | attest the fast message | guardian set 7, 19 keys on Hydration core `0x3792…82a1` | a forged fast message pays out of the pool **before** settlement arrives — pool size is the blast radius |
| Relayer bot | none (permissionless submit) | — | see [[wiki/whm-agents\|whm-agents]] |

No `pause()` anywhere in WHM; the stops are `setAuthorizedBridge(receiver, false)` on the landing (TC) and the NTT manager pause (see [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]]).

## Gotchas

- **Pool binding is unchecked on-chain**: emitter `landing` (Ethereum) must equal `receiver.landing()` (Hydration). Divergence drains the pool while gross accumulates elsewhere.
- Foundry cannot exercise delivery: asset ERC-20s and `0x0401` are runtime precompiles. Only the chopsticks probe `_probeBasejumpDelivery.ts` moves real balances.

## Sources

- [[wiki/source-whm-codebase\|source-whm-codebase]], [[wiki/whm-contracts\|whm-contracts]], [[wiki/wormhole\|wormhole]]
- [[wiki/pallet-dispatcher\|pallet-dispatcher]] (`EmergencyAdminAccount`), [[wiki/hydration-precompiles\|hydration-precompiles]] (`0x0401`, asset ERC-20 addressing)
