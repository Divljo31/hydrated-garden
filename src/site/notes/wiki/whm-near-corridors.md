---
{"dg-publish":true,"permalink":"/wiki/whm-near-corridors/","title":"WHM — NEAR corridors (NEAR NTT, ZEC UTXO deposit / withdraw)","tags":["whm","near","ntt","zec","zcash","omni","mpc","chain-signatures","sputnik-dao","wormhole","utxo","security"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"WHM — NEAR corridors (NEAR NTT, ZEC UTXO deposit / withdraw)","repo":"whm","paths":["docs/near-ntt/spec.md","docs/near-ntt/deployment.md","docs/near-utxo/deposit/spec.md","docs/near-utxo/withdraw/spec.md","crates/near/contracts/ntt-manager/src","crates/near/contracts/utxo-withdraw/src","contracts/src/utxo/ZecUtxoEmitter.sol","deployments/prod/near-ntt-near.json","deployments/prod/near-ntt-zec.json","deployments/prod/utxo-emitter-zec.json","deployments/prod/utxo-withdraw-zec.json","agents/relayer/src/apps/ntt-near","agents/utxo/src"],"key_exports":["ntt-manager","utxo-withdraw","ZecUtxoEmitter","send","WithdrawRequested","ft_on_transfer","complete","withdraw","setFee","sweep","refuel"],"tags":["whm","near","ntt","zec","zcash","omni","mpc","chain-signatures","sputnik-dao","wormhole","utxo","security"],"last_updated":"2026-11-02"}}
---


# WHM — NEAR corridors

**TL;DR:** Hydration ↔ NEAR in one Wormhole hop with a **custom NEAR NTT contract** (no public NEAR NTT exists): `ntt-<token>.hydration-whm.near` is manager + transceiver in one, **LOCKING** on NEAR; Hydration runs stock EVM `NttManager`s, **BURNING**. Tokens: **NEAR** (`wrap.near`, 24 dp) and **ZEC** (`zec.omft.near`, 8 dp — Omni's NEAR representation of Zcash). On top sit UTXO flows: **deposit** native ZEC → Omni connector → NEAR NTT → Hydration, and **withdraw** Hydration → `ZecUtxoEmitter` → NTT to a NEAR account `A` derived from the destination address → `utxo-withdraw` signs (NEAR MPC chain signatures) the only allowed action for `A`: an Omni withdrawal to that address. Live since 2026-10-12 (NTT) / 2026-10-20 (withdraw). BTC is "next".

## Deployments

| Component | Chain | Address | Notes |
|---|---|---|---|
| `ntt-near.hydration-whm.near` | NEAR | emitter `d473056b…cb9e3` | LOCKING `wrap.near`; peer = Hydration manager `0x374E5239a2342Be6Bf31B410EC03984D03A8b7dE`, transceiver `0xc74BcE0b…81d1`; inbound limit 1e27 (1,000 NEAR) |
| `ntt-zec.hydration-whm.near` | NEAR | emitter `ca409242…3976` | LOCKING `zec.omft.near`; peer = Hydration manager `0xe20cED5f5798cD4FeBc7aE182309b9f51345e0cC`, transceiver `0x0dd43A80…DE85`; inbound limit 1e9 (10 ZEC) |
| Hydration NTT managers (NEAR, ZEC) | Hydration | above | deployed from the separate NTT repo ("Hydration side is hydration-ntt's") |
| `ZecUtxoEmitter` (UUPS) | Hydration | `0xb3e365a6241683d5d4ef2701de6bedc16dcc259e` | `send(amount, destination, nearAccount)`; fee 10000 (0.0001 ZEC); wired to both managers |
| `withdraw-zec.hydration-whm.near` | NEAR | — | `UtxoWithdraw` for Zcash; stateless |

## Trust & admin

| Component | Moves funds? | Owner / upgrade | Notes |
|---|---|---|---|
| `ntt-near` / `ntt-zec` (NEAR) | **custody of all locked NEAR / ZEC backing Hydration supply** | **`hydration.sputnik-dao.near`** (Sputnik DAO; DAO-upgradeable); deployer full-access key `ed25519:9hziFbfc…xQ` **deleted** | trust = Wormhole guardians + this DAO's council |
| Hydration NTT managers | mint/burn NEAR, ZEC on Hydration | NTT repo, not here (per [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]] Hydration-side managers are owned by `EmergencyAdminAccount`) | verify for these two |
| `ZecUtxoEmitter` (Hydration) | pulls user ZEC, burns via NTT | **`EmergencyAdminAccount` `0xaa7e…aa7e1`** — `setFee`, `setZecNttManager`, `setNearNttManager`, **`sweep`**, upgrade | fees accumulate here; governance `refuel` converts them to NEAR gas for bots |
| `withdraw-zec` (NEAR) | signs Omni withdrawals for derived accounts | **`hydration.sputnik-dao.near`**, keys deleted | can only sign the fixed meta-tx for `A` → destination; depends on NEAR MPC (`v1.signer`) and Omni connector |
| Omni bridge / connector | custody of native ZEC | Omni (NEAR MPC, trusted relayer gate, Solana admin that can replace the MPC key) | **second trust stack** under every Hydration ZEC — the spec notes Omni locks ~101,706 `zec.omft.near` |
| Bots (`ntt-near`, `utxo` relayer) | none (pay gas) | keys | no authority by design |

Dependency chain for **ZEC on Hydration**: Zcash → Omni (NEAR MPC + relayer + Solana admin) → `zec.omft.near` → our NEAR NTT (Sputnik DAO) → Wormhole guardians → Hydration NTT manager (EmergencyAdmin). Any link failing makes Hydration ZEC unbacked → [[wiki/runbook-collateral-damage-playbooks\|runbook-collateral-damage-playbooks]] §A.

## Gotchas

- Rate limits on NEAR side are per-token inbound limits (above); Hydration-side limits live with the NTT managers.
- Withdrawal has **no refund**: funds parked in `A` can only ever leave to the address encoded in its derivation path (`"<chain>:v1:" + address`).
- NEAR self-redeem in v1; no relayer fee on the NEAR redeem leg.

## Sources

- [[wiki/source-whm-codebase\|source-whm-codebase]], [[wiki/whm-contracts\|whm-contracts]], [[wiki/whm-agents\|whm-agents]], [[wiki/wormhole\|wormhole]], [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]]
