---
{"dg-publish":true,"permalink":"/wiki/wormhole/","title":"Wormhole","tags":["bridge","cross-chain","ntt","ethereum","solana","sui","near","robinhood","hyperevm","whm"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"concept","title":"Wormhole","tags":["bridge","cross-chain","ntt","ethereum","solana","sui","near","robinhood","hyperevm","whm"],"source_count":4,"last_updated":"2026-11-02"}}
---


# Wormhole

**TL;DR:** Wormhole is [[wiki/hydration\|hydration]]'s bridge beyond [[wiki/polkadot\|polkadot]], complementing [[wiki/snowbridge\|snowbridge]]. Hydration is a **first-class Wormhole chain — id `73`** (EVM chain id `222222`, para id `2034`) with its **own Wormhole core** `0x3792a6d63c31941b2805181771795d9176fa82a1` (guardian set 7, 19 keys). Two layers: **tokens move over NTT** (per-token `NttManager` + `WormholeTransceiver`; Hydration BURNING for inbound assets, LOCKING for HDX/HOLLAR going out), and **everything else rides WHM** — our own Wormhole messaging monorepo ([[wiki/source-whm-codebase\|source-whm-codebase]]): oracle relay, Basejump fast payouts, intents, cross-chain OpenGov, NEAR/Zcash corridors.

## Token layer — NTT

Hydration talks NTT directly to every spoke — `NttManager` + `WormholeTransceiver` (`xc-core/bridge/ntt.ts`), tags `Tag.Ntt` / `Tag.NttExecutor`, asset keys `*_wh`, rate limits checked by `NttRateLimitValidation`.

**Live NTT routes into Hydration** (relayer-served): DAI, WBTC, WETH, USDC, USDT, sUSDS ← Ethereum; EURC ← Base; SOL, jitoSOL, PRIME ← Solana; SUI ← Sui; NEAR, ZEC ← NEAR (custom NEAR NTT — [[wiki/whm-near-corridors\|whm-near-corridors]]). Also deployed: HYPE ← HyperEVM, SPY ← Robinhood, and **HDX / HOLLAR locking on Hydration → spokes on Robinhood**. Per-token Hydration manager/transceiver addresses: [[wiki/whm-agents\|whm-agents]]. Token NTT deployments themselves live in a separate NTT ops repo, not in WHM.

Each SDK pair carries two routes: plain (`NttManager.transfer`, user self-redeems) and executor (`Tag.NttExecutor`, Wormhole Executor delivers) — mechanics in [[wiki/xc-cfg\|xc-cfg]] / [[wiki/xc-core\|xc-core]]. Chain side: [[wiki/pallet-evm-accounts\|pallet-evm-accounts]] keeps the NTT minter registry (`NttMinters`, `set_ntt_minter`, emergency `clear_ntt_minter` under `NttEmergencyOrigin`). Adding a new chain: [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]].

## Messaging layer — WHM

| Flow | Direction | Page |
|---|---|---|
| Oracle relay (Kamino Scope PRIME/SOL, Jito jitoSOL rate, wstETH, apyUSD, Chainlink SPY) → Hydration oracle contracts; HDX price → Robinhood | in / out | [[wiki/whm-oracle-relay\|whm-oracle-relay]] |
| Basejump — instant USDC Ethereum → Hydration from a pre-funded pool, NTT settles behind | in | [[wiki/whm-basejump\|whm-basejump]] |
| Intents — Hydration asset → WETH → NTT → Ethereum → NEAR Intents 1Click deposit address | out | [[wiki/whm-intents\|whm-intents]] |
| Cross-chain governance — OpenGov → `GovernanceDispatcher` → `GovernanceExecutor` on Ethereum/Base/Robinhood/HyperEVM | out | [[wiki/whm-cross-chain-governance\|whm-cross-chain-governance]] |
| NEAR NTT + ZEC UTXO deposit/withdraw (Omni, NEAR MPC) | both | [[wiki/whm-near-corridors\|whm-near-corridors]] |
| Relayers / broadcasters / indexer | — | [[wiki/whm-agents\|whm-agents]] |
| Address book + owners | — | [[wiki/whm-contracts\|whm-contracts]] |

## Trust summary

- **Guardians** attest every token transfer and every message — a guardian-quorum compromise can forge mints, oracle prices and Basejump payouts at once.
- **Admin domains are split:** Hydration-side contracts answer to **`EmergencyAdminAccount`** (TC majority); external-EVM contracts answer to the **TC Safe `0xD557…81E7`** (4-of-7) or, increasingly, to OpenGov through `GovernanceExecutor`; NEAR contracts to **`hydration.sputnik-dao.near`**; the oracle relays are **renounced** (frozen). Full map: [[wiki/whm-contracts\|whm-contracts]].
- Incident handling for a Wormhole-side failure: [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]] §A, levers in [[wiki/note-emergency-levers\|note-emergency-levers]].

## Sources

- [[wiki/source-whm-codebase\|source-whm-codebase]]
- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
- [[wiki/source-hydration-node-codebase\|source-hydration-node-codebase]]
- [[wiki/source-hydration-ui-codebase\|source-hydration-ui-codebase]]
