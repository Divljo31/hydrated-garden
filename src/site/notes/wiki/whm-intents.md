---
{"dg-publish":true,"permalink":"/wiki/whm-intents/","title":"WHM — Intents (Hydration → NEAR Intents via WETH / NTT)","tags":["whm","intents","near-intents","1click","weth","ntt","wormhole","zec","btc","ice","security"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"WHM — Intents (Hydration → NEAR Intents via WETH / NTT)","repo":"whm","paths":["docs/intents/spec.md","docs/intents/schema.md","docs/intents/relay-fee.md","docs/intents/refund.md","docs/intents-v2/spec.md","contracts/src/intents/IntentEmitter.sol","contracts/src/intents/IntentReceiver.sol","contracts/src/intents/IntentQuoteEmitter.sol","contracts/src/intents/IntentCodec.sol","contracts/src/utils/hydration/HydrationRouter.sol","deployments/prod/intent.json","agents/intent/src/apps","agents/relayer/src/apps/intent","agents/scan/src/apps/intents"],"key_exports":["IntentEmitter","IntentReceiver","IntentQuoteEmitter","placeOrder","processOrder","setAuthorizedRelayer","sweep","setNttManager","setIntentReceiver","maxRelayFee","depositAddress","TRIM_UNIT"],"tags":["whm","intents","near-intents","1click","weth","ntt","wormhole","zec","btc","ice","security"],"last_updated":"2026-11-02"}}
---


# WHM — Intents

**TL;DR:** Lets a Hydration user buy any NEAR-Intents asset (BTC, ZEC, NEAR, …) with one Hydration call. `IntentEmitter.placeOrder(assetIn, amountIn, minEthOut, depositAddress, maxRelayFee)` sells `assetIn` → WETH (asset 20) via the router, settles WETH over **NTT** to `IntentReceiver` on Ethereum, and publishes a second Wormhole message (consistency 202) carrying `(transferSequence, depositAddress, maxRelayFee)`. A relayer calls `IntentReceiver.processOrder(nttVaa, instructionVaa, fee)`, which delivers the settlement and forwards native ETH to the **1Click deposit address**; NEAR's solver network fills the quote. Deployed and live since 2026-09-09. The SDK side is [[wiki/xc-swap\|xc-swap]]. **v2** (`docs/intents-v2/`, draft) replaces 1Click with our own strategy, triggered from ICE `on_resolved` — see [[wiki/ice\|ice]].

## Contracts

| Contract | Chain | Address | Role |
|---|---|---|---|
| `IntentEmitter` (UUPS) | Hydration | `0x98f1ebc9dcc8ab7ba54d83c98500e9e313f793f2` | `placeOrder`; router sell via `0x0401` (`pallet 67`), NTT transfer, instruction publish |
| WETH NTT manager | Hydration | `0xB5cEf790D52A57fa619eD96eDd64c5328F3DCFb7` | settlement rail (Hydration side) |
| `IntentReceiver` (UUPS) | Ethereum | `0x2173f6ece25768e7efc5199f70f8783d88ba63c8` | `processOrder`; deliver-if-needed + forward ETH + pay caller |
| Authorized relayer | Ethereum | `0x43A8A728e2514EDEAe8e0C9FAb8B79fb5E4ecD5F` | 5-min exclusivity window from settlement timestamp, then permissionless |

`IntentQuoteEmitter` and the `intent-mpc` migration exist in code only — no prod record.

## Key design points

- **Two messages bound by the NTT manager's sequence** (not the Wormhole sequence): NTT carries no usable payload and `receiveMessage` is permissionless, so the destination travels separately.
- **Amount comes from the settlement**, never from the instruction — the receiver forwards what the rail actually released.
- **Fees from swap output**: NTT delivery price + Wormhole fee deducted on Hydration (works because the native fee currency *is* WETH); relay fee capped by the user's `maxRelayFee`, sized by `agents/intent` quoter.
- **Quantized to `TRIM_UNIT = 1e10`** (NTT trims WETH to 8 dp).
- **Nothing on-chain validates `depositAddress`** — the user trusts the 1Click quote they signed.
- Refunds return native ETH on Ethereum to the quote's `refundTo` (user's H160), never to the bridge contracts.

## Trust & admin

| Component | Moves funds? | Owner / upgrade | Admin powers |
|---|---|---|---|
| `IntentEmitter` (Hydration) | swaps user assets, sends WETH over NTT | **`EmergencyAdminAccount` `0xaa7e…aa7e1`** (TC majority) | `setNttManager`, `setIntentReceiver`, **`sweep(asset, to, amount)`**, upgrade |
| `IntentReceiver` (Ethereum) | forwards ETH | **TC Safe `0xD557…81E7`** (4-of-7) | `setEmitter`, `setAuthorizedRelayer`, **`sweep(to, amount)`**, upgrade |
| WETH NTT managers | mint/burn/lock WETH | Hydration side: see [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]] / [[wiki/whm-contracts\|whm-contracts]] | NTT `pause` |
| 1Click / POA | custody after forward | NEAR Intents | out of our control |
| Relayer `0x43A8…cD5F` | none (gets fee ≤ `maxRelayFee`) | bot key | exclusivity window only |

Changing `setIntentReceiver` on Hydration (TC) or `setEmitter` on Ethereum (Safe) alone would split the pairing — both sides must move together.

## Sources

- [[wiki/source-whm-codebase\|source-whm-codebase]], [[wiki/whm-contracts\|whm-contracts]], [[wiki/whm-agents\|whm-agents]], [[wiki/xc-swap\|xc-swap]], [[wiki/ice\|ice]], [[wiki/wormhole\|wormhole]]
