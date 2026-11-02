---
{"dg-publish":true,"permalink":"/wiki/runbook-hyperevm-ntt/","title":"Runbook — Bring HyperEVM assets to Hydration over NTT","tags":["bridge","ntt","wormhole","hyperevm","hyperliquid","hype","feusd","runbook","cross-chain"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"runbook","title":"Runbook — Bring HyperEVM assets to Hydration over NTT","repo":"sdk","paths":["packages/xc-cfg/src/chains/evm","packages/xc-cfg/src/chains/evm/base.ts","packages/xc-cfg/src/chains/evm/hyperevm.ts","packages/xc-cfg/src/configs/evm/hyperevm/index.ts","packages/xc-cfg/src/configs/polkadot/hydration/index.ts","packages/xc-cfg/src/configs/polkadot/hydration/templates.ts","packages/xc-core/src/bridge/ntt.ts","packages/xc/docs/ntt.md"],"symbols":["NttManager","WormholeTransceiver","setPeer","setWormholePeer","NttTokenDef","NttDef","viaNttTemplate","viaNttExecutorTemplate","set_ntt_minter","clear_ntt_minter","NttMinters","EvmChain"],"depends_on":["wormhole","xc-cfg","xc-core","pallet-evm-accounts","pallet-asset-registry","opengov"],"tags":["bridge","ntt","wormhole","hyperevm","hyperliquid","hype","feusd","runbook","cross-chain"],"last_updated":"2026-11-02"}}
---


# Runbook — Bring HyperEVM assets to Hydration over NTT

**TL;DR:** HyperEVM (Wormhole **47**, EVM **999**) is **wired** at sdk d16ff85: `chains/evm/hyperevm.ts`, `configs/evm/hyperevm/`, and `hype` (Hydration asset **1001355**) routes both ways, self-redeem and Executor. HYPE went the **plan-B** way: our **own locking NttManager** on HyperEVM (`0xdcDd08BF…676f`, escrows WHYPE `0x5555…5555`, native HYPE wrapped upfront) with a burning manager on Hydration — **not** a peer of Wormhole's canonical HYPE group, so our `hype` is not fungible with Wormhole HYPE on Base/Solana/Unichain. Track B (a HyperEVM-native stable, feUSD recommended) is **not started**. Sections 1, 3–5 remain the how-to for any further HyperEVM asset.

## 0. Status (d16ff85)

| | Hydration (73) | HyperEVM (47) |
|---|---|---|
| EVM chain id | 222222 | 999 |
| core bridge | `0x3792a6d63c31941B2805181771795D9176fA82A1` | `0x7C0faFc4384551f063e05aee704ab943b8B53aB3` |
| Executor | `0xd633d8d1ceee8c8252196d44857c0f41b8dcb0d9` | `0xd7717899cc4381033Bc200431286D0AC14265F78` |
| `nttExecutor` shim | `0xd3Dda7c8608Ea251C42c6E0A2A686aDc5e9C0C03` | `0xBc275e094e031e990b060134AbbDa00132f9A163` |
| HYPE NttManager | burning (asset 1001355) | **locking** `0xdcDd08BFB7b3B9381149A852Ba5F69Bd5d87676f` |
| HYPE transceiver | — | `0xe3eD8E1217AB419123D54E2F0850BC736b595D68` |
| RPC | `https://rpc.coke.hydration.cloud` | `https://rpc.hyperliquid.xyz/evm` |

Routes: Hydration → HyperEVM `viaNttTemplate(hype, hype, hyperevm)` + `viaNttExecutorTemplate(...)`; HyperEVM → Hydration `toHydrationViaNttTemplate(hype|whype, hype)`, `toHydrationViaNttExecutorNativeTemplate(hype, hype)`, `toHydrationViaNttExecutorTemplate(whype, hype)`. `hype` and `whype` share one deployment; `hype` is registered first so a VAA from the shared transceiver resolves to Hydration's key. No mainnet Token Bridge on HyperEVM at sdk-base 6.1.4 — NTT is the only path.

## 1. The two rules that decide everything

**Peering is owner-gated.**

```solidity
// native-token-transfers/evm/src/NttManager/NttManager.sol:104
function setPeer(uint16 peerChainId, bytes32 peerContract, uint8 decimals, uint256 inboundLimit)
    public onlyOwner

// native-token-transfers/evm/src/Transceiver/WormholeTransceiver/WormholeTransceiverState.sol:153
function setWormholePeer(uint16 peerChainId, bytes32 peerContract) external payable onlyOwner
```

The transceiver defers to the manager's owner — `Transceiver.getNttManagerOwner()` returns `IOwnableUpgradeable(nttManager).owner()`. Peers are **local config, bilateral, no cross-chain message**: A→C needs A to know C *and* C to know A. Other chains in the group don't need to know about us unless we want those legs too.

**One hub.** Exactly one chain runs `mode = LOCKING`; every other chain runs `BURNING`. Upstream: *"locking mode which preserves the original token supply on a single chain"*. This is why `usdc_wh` / `usdt_wh` **cannot** be extended to HyperEVM — their hub is the Ethereum locking manager, and a second locking manager would give one burn two escrows to drain. Anything inbound from HyperEVM needs a **new asset key**.

Corollary worth internalising: **locking mode needs nothing from the token issuer.** The Wormhole EVM guide states tokens *"only need to be ERC-20 compliant"* for locking, and the manager is set as minter **only on spoke chains**. Issuer buy-in is a canonicity question, not a deployment blocker.

## 2. Track A — HYPE (done, own manager)

Wormhole's canonical HYPE group is hubbed on HyperEVM (manager `0x452A2613…`, `NttManagerWethUnwrap`, owner = Wormhole `GeneralPurposeGovernance`, peering is an ask with no public process). We did **not** join it: Hydration's `hype` comes from our own locking manager escrowing WHYPE. Consequences:

- **Not fungible** with Wormhole HYPE on Base / Solana / Unichain; moving between the two is a swap, not a bridge.
- **We** hold the escrow and own upgrade, pause and rate-limit config on the HyperEVM side — add it to [[wiki/note-emergency-levers\|note-emergency-levers]] / the NTT ownership table once its owner is read on chain.
- If we later want canonical HYPE, it is a second asset key (one hub per group), not a migration of `hype`.

## 3. Track B — a HyperEVM-native stable (a deploy)

**None of the HyperEVM stablecoins has an NTT deployment on any chain.** The market is also ~97% USDC, which is CCTP and redundant with our `usdc_wh`. Everything not-USDC and not-already-LayerZero adds up to roughly **$95M** — that is the entire addressable pool.

| Token | HyperEVM address | Size | Existing standard | NTT verdict |
|---|---|---|---|---|
| **feUSD** (Felix) | `0x02c6a2fa58cc01a18b8d9e00ea48d65e4df26c70` | ~$75M | **none** — Liquity-v2-fork CDP, HyperEVM/HyperCore only | **best.** Plain ERC-20, zero conflicting commitments |
| **USDhl** (Felix Labs / M0) | `0xb50A96253aBDF803D85efcDce07Ad8becBc52BD5` | ~$15M | Hyperlane (M0 Portal Lite) | good. M0 is already a Wormhole NTT customer for `$M`; backing sits on Ethereum, so only nominally native |
| USDM (Monetrix) | UNKNOWN | ~$2.4M | none specified | possible, tiny, very new |
| USDe (Ethena) | `0x5d3a1ff2b6bab83b63cd9ad0787074081a52ef34` | ~$15.5M here | LayerZero OFT | poor — canonical mint is Ethereum, locking = bridge-on-a-bridge |
| USDT0 | `0xB8CE59FC3717ada4C02eaDF9682A9e934F625ebb` | ~$100–155M here | LayerZero OFT, **Ethereum lockbox** | dead |
| USDC (Circle) | `0xb88339CB7199b77E23DB6E890353E22632Ba630f` | ~$5B | CCTP V2 | impossible + redundant |
| USDH | `0x111111a1a0667d36bd57c0a9f569b98057111111` (low confidence) | →0 | custom CoreRouter | dead — **sunset 17 Jul 2026**, Coinbase took the brand |

**Recommendation: feUSD.** Only asset that is simultaneously genuinely HyperEVM-native with no upstream lock, meaningfully sized, and free of a competing bridge commitment. Felix is a DeFi-native team with no regulated-issuer gate, so their blessing is plausible — worth getting for canonicity even though it is not technically required.

USDhl is the better *relationship* play (Felix Labs runs it, M0 already ships NTT on Ethereum/Arbitrum/Base) but is 5× smaller and M0 may prefer extending its own portal.

Topology: **HyperEVM = locking hub** (we deploy and own it), **Hydration = burning spoke**.

Risk to name explicitly in any proposal: a permissionless locking hub means **we** hold the escrow, **we** own upgrade and rate-limit config, and if Felix later runs an official NTT group ours becomes the non-canonical one.

## 4. Hydration-side checklist (both tracks)

1. Register the asset in [[wiki/pallet-asset-registry\|pallet-asset-registry]] → note the id; the erc20 precompile address is `0x00000000000000000000000000000001{assetId:08X}`.
2. Deploy a **burning** `NttManager` + `WormholeTransceiver` on Hydration against that precompile (same shape as the existing burning entries in `chains/polkadot/hydration.ts`).
3. `EVMAccounts.set_ntt_minter(asset_id, manager)` — call index 7, **`ControllerOrigin`**, so it goes through [[wiki/opengov\|opengov]]. Registry is per-asset and single-valued; overwrites without a prior-value check. Emergency stop is `clear_ntt_minter` under the deliberately faster `NttEmergencyOrigin`.
4. `setPeer(47, …)` + `setWormholePeer(47, …)` on our manager/transceiver.
5. `chains/evm/hyperevm.ts` — add the asset to `assetsData` (chain def exists).
6. Registry entries under `wormhole.ntt` on **both** chain defs, keyed by the **same asset key** (registry rule 1 in `packages/xc/docs/ntt.md` — chain-local key variants silently kill the redeem callback).
7. Routes in `configs/polkadot/hydration/index.ts` via `viaNttTemplate` **and** `viaNttExecutorTemplate`; reverse legs in `configs/evm/hyperevm/index.ts`.
8. `NttRateLimitValidation` and `NttCustodyValidation` pick it up once both ends are registered.

## 5. Gotchas

- **Decimals.** WHYPE is 18 on HyperEVM, the Solana mint is 9, and NTT trims to `min(8, decimals)`. EVM amounts must be floored to wire precision or the manager reverts `TransferAmountHasDust`.
- **`NttManagerWethUnwrap`.** The HYPE hub unwraps WHYPE → native HYPE on delivery. Good UX outbound; means the escrow is WHYPE, not native.
- **The shim still won't work from Hydration.** `NttManagerWithExecutor` approves `type(uint256).max` and our erc20 precompile carries a u128 balance. Hydration→HyperEVM uses `transferViaExecutor()` (two calls we own), same as every other outbound leg.
- **Executor cost is paid in WETH on Hydration** (`EVM.call { value }` debits asset 20) whatever the destination's gas token; `HydrationEvmValueValidation` checks the fee-currency side.
- **Executor sequence race.** `Executor.requestExecution` reads `nextMessageSequence()` at build time; another transfer through the same manager landing in between leaves the request pointing at the wrong message. Nothing relays, transfer stays hand-claimable.
- **Wormhole calls HyperEVM experimental** — verbatim, *"its node software is not open source. Use Wormhole messaging on HyperEVM with caution."* Belongs in the risk memo, and argues for a conservative initial `inboundLimit` rather than the uint64 sentinel HYPE uses.
- **HyperCore linking is optional and separate.** Only needed if the ERC-20 should be tradeable on HyperCore: HIP-1 spot deploy, `ntt hype link --token-index`, `szDecimals + 5 <= weiDecimals`, genesis mint to `0x2000000000000000000000000000000000000{tokenIndex:04x}`. Deploy Spot steps are permanent.
- **Big blocks.** Contract deploys on HyperEVM need the big-block path (30M gas, ~1min), not small blocks.
- **NTT Launchpad won't help** — it covers 9 chains and HyperEVM is not one. CLI only, following the Hyperliquid guide.

## 6. Open questions

- Owner / pauser and rate limits of **our** HyperEVM HYPE manager `0xdcDd08BF…676f` (not yet read on chain).
- Whether Felix would sanction an NTT group for feUSD, and whether they'd want to own it instead.
- Whether the Ethereum-side 4-of-7 Safe and the `pauser == owner` collapse are intended. The seven owners are now recorded in [[wiki/whm-cross-chain-governance\|whm-cross-chain-governance]]; the same Safe is the vetoer on every WHM `GovernanceExecutor`.

## Sources

- [[wiki/wormhole\|wormhole]]
- [[wiki/xc-cfg\|xc-cfg]]
- [[wiki/xc-core\|xc-core]]
- [[wiki/pallet-evm-accounts\|pallet-evm-accounts]]
- [[wiki/source-sdk-codebase\|source-sdk-codebase]]
- Off-vault: [native-token-transfers](https://github.com/wormhole-foundation/native-token-transfers) (`evm/src/NttManager/NttManager.sol`, `evm/src/Transceiver/WormholeTransceiver/WormholeTransceiverState.sol`, `evm/src/wormhole/Governance.sol`), [NTT overview](https://wormhole.com/docs/products/token-transfers/native-token-transfers/overview/), [Deploy NTT to Hyperliquid](https://wormhole.com/docs/products/token-transfers/native-token-transfers/guides/deploy-to-hyperliquid/), [HyperEVM live on Wormhole](https://wormhole.com/blog/hyperliquids-hyperevm-is-now-live-on-wormhole), [HYPE goes multichain with NTT](https://wormhole.com/blog/hype-goes-natively-multichain-with-wormhole-ntt), [Circle USDC on HyperEVM](https://www.circle.com/multi-chain-usdc/hyperevm), [Felix USDhl docs](https://usefelix.gitbook.io/usdhl/advanced/developers), [USDT0 deployments](https://docs.usdt0.to/technical-documentation/deployments)
