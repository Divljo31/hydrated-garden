---
{"dg-publish":true,"permalink":"/wiki/whm-cross-chain-governance/","title":"WHM — Cross-chain governance (OpenGov → GovernanceExecutor on EVM chains)","tags":["whm","governance","opengov","technical-committee","safe","wormhole","cross-chain","ntt","uniswap-v4","security"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"WHM — Cross-chain governance (OpenGov → GovernanceExecutor on EVM chains)","repo":"whm","paths":["docs/governance/spec.md","contracts/src/governance/GovernanceDispatcher.sol","contracts/src/governance/GovernanceExecutor.sol","contracts/src/governance/GovernanceCodec.sol","deployments/prod/governance-hydration.json","deployments/prod/governance-ethereum.json","deployments/prod/governance-base.json","deployments/prod/governance-robinhood.json","deployments/prod/governance-hyper-evm.json","deployments/prod/verification"],"key_exports":["GovernanceDispatcher","GovernanceExecutor","GovernanceCodec","publish","queue","queueTechnicalCommitteeAction","execute","veto","vetoTechnicalCommitteeAction","setVetoer","setSourceDispatcher","setTiming","ActionPublished","CrossChainGovernanceAccount","HDXGOV"],"tags":["whm","governance","opengov","technical-committee","safe","wormhole","cross-chain","ntt","uniswap-v4","security"],"last_updated":"2026-11-02"}}
---


# WHM — Cross-chain governance

**TL;DR:** Lets Hydration [[wiki/opengov\|opengov]] administer protocol-owned contracts on other EVM chains (initial targets: Uniswap v4 LP NFTs and Wormhole NTT admin roles). Root dispatches as a fixed synthetic account **`CrossChainGovernanceAccount` `0xaa7e…aa7e2`** → `GovernanceDispatcher.publish(destChain, executor, calls[])` on Hydration → Wormhole VAA (consistency 202, finalized) → `GovernanceExecutor.queue(vaa)` on the destination → executable after **24 h**, expires 7 days later. The destination **TC Safe** can veto or fast-track OpenGov actions, and can propose its own **recovery** calls that wait **30 days** with an OpenGov veto. Deployed 2026-10-15 … 10-21 to Ethereum, Robinhood, Base and HyperEVM.

## Deployments

| Contract | Chain (Wormhole id) | Proxy | Impl |
|---|---|---|---|
| `GovernanceDispatcher` | Hydration (73) | `0x00302167f9597b9d90bd342311c5c4be39bf0fa0` | `0xfddb5acb…3bed` |
| `GovernanceExecutor` | Ethereum (2) | `0xCEaab5A5AF21141a501FF5b561b6Cc3d4c55eF83` | `0xef47b42d…64f2` |
| `GovernanceExecutor` | Robinhood (72) | `0xceaab5a5af21141a501ff5b561b6cc3d4c55ef83` (same address) | `0xef47b42d…64f2` |
| `GovernanceExecutor` | Base (30) | `0x8830b6eb2e015184b86cfa3a0a759cd55ff47a4b` | `0xf1a5fe42…d188` |
| `GovernanceExecutor` | HyperEVM (47) | `0x102225852eca1937ae57d2fb23409877aee50699` | `0xf1a5fe42…d188` |

Executor config on every destination: source dispatcher `0x0030…0fa0` on chain 73; **vetoer = Safe `0xD557AeAf1e0cB3D226BfF3B7a10C2cdA9dA081E7`, threshold 4, 7 owners** (`0x990D…520A`, `0x8F6b…515D`, `0x6466…Cf30`, `0xF34e…4d99`, `0x538a…8A3a`, `0x2b6F…8497`, `0x16b2…5AC4`); `vetoPeriod` 86,400 s; `executionGracePeriod` 604,800 s.

## Source path (Hydration)

```
Root  →  Utility::dispatch_as(Signed(0xaa7e…aa7e2 ‖ 0…), Dispatcher::dispatch_evm_call(EVM::call(source = 0xaa7e…aa7e2,
          target = GovernanceDispatcher, publish(...))))
```

- Deliberately **not** `AaveManagerAccount` (`…aa7e0`) or `EmergencyAdminAccount` (`…aa7e1`): reusing either would let the EconomicParameters track or the TC publish cross-chain actions immediately. See [[wiki/pallet-dispatcher\|pallet-dispatcher]].
- The account has no key, holds only gas + Wormhole fee. `Utility::dispatch_as` returns outer success even when the inner call fails — check `Utility::DispatchedAs` **and** `ActionPublished`.
- Payload: `abi.encode(bytes6 "HDXGOV", uint8 1, destChain, destExecutor, governanceNonce, (address,uint256,bytes)[])`, ≤16 calls, ≤32 KiB per call, ≤64 KiB total; re-encoded and byte-compared on both ends; action id = `keccak256(domain, emitterChain, emitter, keccak256(payload))`.

## Trust & admin

| Component | Power | Upgrade | Who |
|---|---|---|---|
| `GovernanceDispatcher` | publishes cross-chain actions | UUPS, `_authorizeUpgrade` = `onlyGovernance` (the fixed caller); no owner, no setter for the caller | OpenGov **Root** only |
| `GovernanceExecutor` ×4 | **arbitrary calls** as itself on each chain (owns LP NFTs / NTT admin roles it is given) | UUPS, `_authorizeUpgrade` = `onlySelf` (i.e. via a governance action) | OpenGov (24 h) / TC Safe fast-track / TC Safe recovery (30 d, OpenGov-veto) |
| TC Safe `0xD557…81E7` (4-of-7, same address on all chains) | `veto(actionId, reason)`, fast-track (execute before maturity), `queueTechnicalCommitteeAction(payload)` (30-day recovery) | — | 7 signers above |

Executor self-config (`setVetoer`, `setSourceDispatcher`, `setTiming`, `vetoTechnicalCommitteeAction`) is `onlySelf` — reachable only through an executed action.
| Wormhole guardians (Hydration core set 7, 19 keys) | attest dispatcher messages | — | Wormhole |

**Same Safe everywhere.** `0xD557…81E7` is also the owner+pauser of the Ethereum NTT managers ([[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]]), owner of `BasejumpEmitter` and `IntentReceiver` ([[wiki/whm-basejump\|whm-basejump]], [[wiki/whm-intents\|whm-intents]]). The executor is the path to move those roles under OpenGov; whether any NTT/LP role has actually been transferred to an executor is **not recorded in this repo** — check on chain.

## Sources

- [[wiki/source-whm-codebase\|source-whm-codebase]], [[wiki/whm-contracts\|whm-contracts]], [[wiki/opengov\|opengov]], [[wiki/pallet-dispatcher\|pallet-dispatcher]], [[wiki/runbook-hyperevm-ntt\|runbook-hyperevm-ntt]]
