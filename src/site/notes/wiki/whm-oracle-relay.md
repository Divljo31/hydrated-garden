---
{"dg-publish":true,"permalink":"/wiki/whm-oracle-relay/","title":"WHM — Oracle relay (Solana / Ethereum / Robinhood → Hydration oracles)","tags":["whm","oracle","wormhole","solana","kamino-scope","jito","wsteth","apyusd","spy","robinhood","hdx","security"],"dgShowBacklinks":true,"dgShowLocalGraph":true,"dgShowInlineTitle":true,"dgShowFileTree":true,"dgShowToc":true,"dg-note-properties":{"type":"package","title":"WHM — Oracle relay (Solana / Ethereum / Robinhood → Hydration oracles)","repo":"whm","paths":["docs/oracle/spec.md","contracts/src/oracles/OracleEmitter.sol","contracts/src/oracles/OracleReceiver.sol","contracts/src/oracles/adapters/AggregatorAdapter.sol","contracts/src/oracles/adapters/AggregatorV3Adapter.sol","contracts/src/MessageReceiver.sol","crates/solana/programs/oracle-emitter/src","deployments/prod/oracle-relay-solana.json","deployments/prod/oracle-relay-ethereum.json","deployments/prod/oracle-relay-robinhood.json","deployments/prod/oracle-relay-hydration.json","migrations/envs/prod","agents/broadcaster/src/apps","agents/relayer/src/apps/oracle"],"key_exports":["OracleEmitter","OracleReceiver","AggregatorAdapter","AggregatorV3Adapter","send","send_price","send_rate","registerFeed","setOracle","setMaxPriceAge","receiveMessage","maxPriceAge","PRICE_SCALE_DIVISOR"],"tags":["whm","oracle","wormhole","solana","kamino-scope","jito","wsteth","apyusd","spy","robinhood","hdx","security"],"last_updated":"2026-11-02"}}
---


# WHM — Oracle relay

**TL;DR:** Pushes external prices/rates into Hydration's EVM oracle contracts over Wormhole, no intermediate hop. A source emitter reads the feed and publishes a 128-byte VAA `(action, assetId, value 18-dec, timestamp)`; a per-source `OracleReceiver` on Hydration verifies it, rejects stale/out-of-order updates, divides by `1e10` (→ 8 dec) and calls `setPrice(int256)` on the mapped oracle. **All four prod relays are frozen**: receivers and EVM emitters have renounced ownership (and with it UUPS upgradeability); the Solana program's upgrade authority is revoked. This is the source of the money market's non-DIA prices for PRIME, SOL, jitoSOL, wstETH, apyUSD and SPY.

## Live feeds into Hydration (as committed in `deployments/prod/`, 2026-09/10)

Receiver maps one oracle per assetId; the later `*-checked` step **overwrote** the earlier mapping, so the checked oracle is the live target where one exists.

| Asset | Source chain → emitter | What is read | Hydration `OracleReceiver` | Live target oracle (Hydration EVM) | Superseded oracle |
|---|---|---|---|---|---|
| PRIME | Solana `oracle-emitter` `2knjvg…sdn` | Kamino Scope price, index **383** (`ScopePrices 3t4JZc…chNH`) | `0x582e2fac…69c3` | `0x09221057Cf7E75953D199FB319E606972A6A82Cd` (checked) | `0x6e3E9403…C84e` |
| SOL | Solana, same program | Kamino Scope price, index **1** | `0x582e…69c3` | `0xf832dc4268Ac29C9C0B16De1784382BEee801Fb8` | — |
| jitoSOL | Solana, same program | SPL stake pool `Jito4APyf642JPZPx3hGc6WWJ8zPKtRbRs4P815Awbb` `total_lamports / pool_token_supply` (rate, action 2) | `0x582e…69c3` | `0x64b7BbAC63E5aDcA0ec35C4AEBf9937Fc1d79C1D` (checked) | `0xFcEd56d8…6CeE` |
| wstETH | Ethereum `OracleEmitter` `0xfbf68264…b4b9` | `wstETH.stEthPerToken()` (stETH per wstETH, not USD) | `0x69137704…1083` | `0xBE6B91cCb5b41e68426Dcb584830A360093c2775` (checked) | `0x35bEe055…C9FD` |
| apyUSD | Ethereum, same emitter | `apyUSD.convertToAssets(1e18)` (apxUSD per share, ERC-4626) | `0x6913…1083` | `0x80FaaC4Da72FCcABDa5B276D9A931252D6242Caf` (checked) | `0x6a738A5B…44Bd` |
| SPY | Robinhood (Wormhole 72) `OracleEmitter` `0x9fbd9f16…a9de` | Chainlink `SPY/USD` `0x319724394D3A0e3669269846abE664Cd621f9f6A` via `AggregatorAdapter` `0xb01d3f48…3649` (`maxAge 0`) | `0x3b6e3469…39ce` | `0x802893C9CDD5AaC833E52783DF39e572fe8f8538` | — |

**Outbound (Hydration → Robinhood):** HDX price from Hydration's EMA oracle precompile `0x0000010200000000000000000000000a00000000` (decoded per [[wiki/hydration-precompiles\|hydration-precompiles]]: period byte `02` = TenMinutes, source bytes all zero, asset A = 10, asset B = 0 — source decoding unverified) via `AggregatorAdapter` `0xbc675831…856f` → Hydration `OracleEmitter` `0x3f5cc441…d066` → Robinhood `OracleReceiver` `0x060f1ef6…b958` → oracle `0xD1a2198CBbC7cbfB602DF581D8614eD31682901D`. Supports the HDX/HOLLAR NTT spokes on Robinhood ([[wiki/whm-contracts\|whm-contracts]]).

assetId scheme: Solana = canonical mint pubkey (bytes32); EVM sources = `keccak256(symbol)`.

## Flow

1. **Publish** (permissionless, caller pays the Wormhole fee): Solana `send_price(asset)` / `send_rate(asset)`; EVM `send(assetId)` does a `staticcall` to the registered source and reads one 18-dec `uint256`. Triggered by [[wiki/whm-agents\|whm-agents]] `broadcaster` on a `thresholds.json` change-detect loop + full-refresh interval.
2. **Relay** (permissionless): `relayer/oracle` submits the VAA to `OracleReceiver.receiveMessage()`. Failed writes revert the whole call; the VAA stays retryable.
3. **Receive** (`OracleReceiver._handleOracleUpdate`): emitter check + replay by VAA hash (`MessageReceiver`) → monotonic per asset (`vaaTimestamp > latest`) → absolute freshness `vaaTimestamp + maxPriceAge >= block.timestamp` (`maxPriceAge = 300` s, fixed) → `OracleNotSet` if unmapped → scale `÷1e10`, reject zero → `oracle.setPrice()`.

## Trust & admin

| Component | Can push prices into Hydration? | Upgrade | Pause | Admin today |
|---|---|---|---|---|
| Solana `oracle-emitter` | Yes (via VAA) | **None** — upgrade authority revoked 2026-09-22 (was `6db8TCH5…uiq`) | none | `Config.owner = 6db8TCH51fK4Pq2LkBdMaMFPfXD17GjfLD8TaMPWNuiq` (deployer key) can only **create** new feed PDAs (`init` on `[price_feed, asset_id]`); existing feeds can't be rebound, and receivers ignore unmapped assetIds |
| Ethereum / Robinhood / Hydration `OracleEmitter` (UUPS) | Yes | **Impossible** — `_authorizeUpgrade` is `onlyOwner`, owner = `0x0` | none | none (renounced); feed bindings final |
| `OracleReceiver` ×4 (UUPS) | **Calls `setPrice`** | **Impossible** (owner `0x0`) | none | none — `maxPriceAge`, emitter, assetId→oracle mapping all frozen |
| Hydration target oracle contracts (6 above) | Final write | **Not in this repo** | ? | **Unknown — the only remaining kill switch.** Each authorizes its receiver to call `setPrice`; revoking that authorization (or freezing the MM reserve) is the only way to stop a bad feed |
| Wormhole guardians (13/19) | Attest every VAA | — | — | Wormhole |

Trust roots per price: **Kamino Scope** (PRIME, SOL), **Jito stake pool accounting** (jitoSOL), **Lido wstETH** (wstETH), **apyUSD ERC-4626 vault** (apyUSD — `convertToAssets` is a live share price, so vault-donation / share-price manipulation flows straight into Hydration within one VAA; whether the "checked" oracle bounds the jump is not visible here), **Chainlink on Robinhood** (SPY), **Hydration EMA** (HDX outbound).

## Gotchas

- Rates are published in **native units** (stETH per wstETH, apxUSD per apyUSD, SOL per jitoSOL); denomination to USD is the consuming oracle's job.
- The stale check uses the VAA's observation time, not the payload `timestamp`.
- An oracle serving two sources must authorize both receivers; adding a source = new parallel receiver, never a reconfiguration.
- Relay of oracle VAAs is listed as "not implemented" in `docs/oracle/spec.md`, but `agents/relayer/src/apps/oracle` and `oracle-robinhood` exist and `sh/print-agents.sh` tracks them — the doc is stale.

## Sources

- [[wiki/source-whm-codebase\|source-whm-codebase]], [[wiki/whm-contracts\|whm-contracts]], [[wiki/whm-agents\|whm-agents]]
- [[wiki/hydration-precompiles\|hydration-precompiles]] (EMA oracle precompile addressing), [[wiki/hydration-borrow\|hydration-borrow]], [[wiki/note-dependency-registry\|note-dependency-registry]]
