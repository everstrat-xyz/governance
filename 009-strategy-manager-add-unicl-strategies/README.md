# 009 — StrategyManager: register the two UniCL USDC/WETH strategies

**Status:** 📝 **Draft — not yet submitted.** Payloads below are simulated against forked
mainnet but have not been signed or scheduled.
**Operation ids:**
- 0.3% pool strategy: `0x345dfba96cc9da6d54b34e07a99bdc594eac68248b26cb292a6c0d18a5e7eca8`
- 0.01% pool strategy: `0xf340bad15fd7770a796eb38047ff1170e898b4f5541a656cd5a0bb828ea50990`

Both recomputable via `hashOperation`, verified on mainnet.

## What it changes

Two calls to `StrategyManager.addStrategy(address strategy, uint8 depositWeight, uint8 withdrawalWeight)`
(selector `0xdca0c48f`) on `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9`, registering the two
`UniCLStrat` instances already deployed against USDC/WETH Uniswap V3 pools:

| Strategy | Pool (fee tier) | `depositWeight` | `withdrawalWeight` | `maxTotalNAV` |
|---|---|---|---|---|
| `0x5E12FFD5e0F69d37B581D06b68ad6c53609cac38` | `0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D8` — USDC/WETH **0.3%** | `40` | `40` | 1000 ETH |
| `0x59C476c1817b23791d96F35b9f468E6d9A242252` | `0xE0554a476A092703abdB3Ef35c80e0D76d32939F` — USDC/WETH **0.01%** | `15` | `15` | 140 ETH |

`addStrategy` adds the strategy to `_strategies`, sets both weights (capped at
`MAX_DEPOSIT_WEIGHT` / `MAX_WITHDRAWAL_WEIGHT` = 100 each), and calls
`Converter.grantCallerRole(strategy)` so the strategy can call the Converter's swap
functions — no extra role grant needed since `grantCallerRole` is gated to the
StrategyManager contract itself (`onlyAuthContract(Auth.STRATEGY_MANAGER)`).

**What it does NOT do:** deploys no code (both `UniCLStrat` instances already exist on
mainnet — bytecode-only deploys via `DeployUniCLStrat.s.sol`, verified below), moves no
funds (`totalDeposited() == 0`, `navInETH() == 0` on both), registers no Oracle feed,
changes no Converter allowlist entry, touches no other strategy.

## Why

Both strategies were deployed for exactly this pairing: USDC (already priced and
StrategyManager-supported via [003](../003-oracle-usd-feeds/)/[004](../004-strategy-manager-supported-usdc/))
and WETH, routed through the whitelisted `UniswapV3ConverterAdapter`
([005](../005-converter-allow-univ3-adapter/)). `addStrategy` is the final go-live step —
per `DeployUniCLStrat.s.sol`'s own deployment-order comment, it always runs on the 48h
admin timelock, after the adapter allowlist and deploy steps, never through a bootstrap key.

Weights: `40`/`40` for the 0.3% pool, `15`/`15` for the 0.01% pool — roughly proportional
to their `maxTotalNAV` caps (1000 ETH vs 140 ETH), giving the deeper, higher-fee-tier pool
the larger share of both deposit and withdrawal routing.

**Timing:** both strategies report `totalDeposited() == 0` and are unregistered — this is
the first time either becomes reachable by the protocol; no existing position is affected.

## No predecessor — but a strong recommendation

`addStrategy` has no on-chain dependency on [008](../008-oracle-staleness-margin/) (the
Oracle staleness-margin proposal): it only checks `ADMIN_ROLE`, `whenNotPaused`, non-zero
address, has-code, and not-already-registered. Neither `predecessor` is set to 008's
operation ids, and this proposal can execute before, after, or independent of it.

That said, 008 is **strongly recommended** to land first (or alongside). Both strategies'
swap path cross-checks its Uniswap TWAP quote against the Oracle's `USDC / USD` and
native-ETH price within `MAX_QUOTE_DEVIATION_BPS` — and both of those Oracle feeds
currently sit at **exactly** their Chainlink heartbeat with zero jitter margin (the
[003](../003-oracle-usd-feeds/) issue 008 fixes). Once these strategies hold real capital,
an Oracle-side `OracleStalePrice` revert on ordinary heartbeat jitter would block a
deposit, withdrawal, or rebalance that has nothing to do with either strategy. Landing 008
first removes that failure mode before it can bite; it is not required for `addStrategy`
to succeed and is not encoded as a `predecessor` here.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` ×2 (one batch), `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` ×2 after the delay |

The two `schedule` calls are meant to go out as a single Safe batch (Transaction Builder /
`MultiSendCallOnly`), same shape as [008](../008-oracle-staleness-margin/)'s three-op batch.
`01-schedule-raw.json` is the same two transactions as raw calldata (selector `0x01d5062a`).

### Parameters

Common to both:

| Field | Value |
|---|---|
| `to` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (admin timelock) |
| `target` | `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` (StrategyManager) |
| `value` | `0` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `delay` | `172800` (48h) |

Per-operation `data` (`addStrategy(address,uint8,uint8)`, selector `0xdca0c48f`) and salt:

| Operation | `data` | `salt` |
|---|---|---|
| 0.3% pool strategy | `0xdca0c48f0000000000000000000000005e12ffd5e0f69d37b581d06b68ad6c53609cac3800000000000000000000000000000000000000000000000000000000000000280000000000000000000000000000000000000000000000000000000000000028` | `0xf6f07d85a2e33d9c21291a61176ceea72ec2c84314aeb767fa2787e2f45a45d3` |
| 0.01% pool strategy | `0xdca0c48f00000000000000000000000059c476c1817b23791d96f35b9f468e6d9a242252000000000000000000000000000000000000000000000000000000000000000f000000000000000000000000000000000000000000000000000000000000000f` | `0x83732cd5a6911a31a8bbbddb1405a52850ef90f550974a3ae21657997919908a` |

Salts derive from
`keccak256("everstrat/strategy-manager/add-strategy/unicl-usdc-weth-<0.3pct|0.01pct>/2026-09-12")`.
Predecessor (zero) and salt must be reused **byte-for-byte** at execute time, per operation.

## Verification performed (forked mainnet, before signing)

Fork at block 25963186. Starting state: neither strategy registered,
`StrategyManager.paused() == false`, `strategyCount() == 0`.

**Deployed-strategy sanity, mainnet:**

1. Both `0x5E12FFD5…cac38` and `0x59C476c1…242252` byte-identical (outside immutables) to
   the compiled `UniCLStrat` artifact — 24,391 bytes, matches exactly.
2. `token0() == USDC`, `token1() == WETH`, `pairedToken() == USDC`, `weth() == WETH`,
   `factory()` == canonical Uniswap V3 factory, for both.
3. `pool()` verified against `factory.getPool(USDC, WETH, pool.fee())`: strategy 1 →
   `0x8ad599c3…6e6D8` at fee `3000` (0.3%, `tickSpacing = 60`); strategy 2 →
   `0xE0554a47…2939F` at fee `100` (0.01%, `tickSpacing = 1`). Both match the addresses
   given for registration exactly.
4. Both pools' `observationCardinality` (1440 and 8192) comfortably exceed
   `MIN_OBSERVATION_CARDINALITY = 150` at the strategies' configured `twapInterval = 1800`
   / `shortTwapInterval = 60` (both at the contract's floor values).
5. `swapAdapter()` on both == `0x0844580a121124CAEc6Cf4A933aac401813cCde5`, the
   [005](../005-converter-allow-univ3-adapter/) adapter — confirmed still whitelisted
   (`Converter.isAdapterAllowed == true`).
6. Both `paused() == false`, `totalDeposited() == 0`, `navInETH() == 0` — fresh, unfunded.

**Timelock simulation:**

1. `hashOperation` for both (target StrategyManager, value 0, respective `data`, zero
   predecessor, respective salt) reproduces the two operation ids above.
2. Direct `addStrategy(strategy1, 40, 40)` from an unrelated EOA — **reverts**
   `RegistryClientMissingRole(ADMIN_ROLE)` (`0x4d616cff`).
3. DAO Safe → `schedule` ×2 with the exact calldata above — **both succeed**;
   `CallScheduled` + `CallSalt` emitted for each op id.
4. `execute` (strategy 1) before the delay — **reverts** `TimelockUnexpectedOperationState`
   (`0x5ead8eb5`).
5. Warp 48h + 1s. `execute` **from an unrelated EOA** for both — **both succeed**
   (349,592 / 266,541 gas respectively).
6. Post-state: `isStrategyRegistered` → `true` for both; `strategyCount() == 2`;
   `depositWeight`/`withdrawalWeight` == `(40, 40)` and `(15, 15)` respectively;
   `Registry.hasRole(CONVERTER_CALLER_ROLE, strategy)` → `true` for both.
7. Re-`execute` (strategy 1) — reverts `TimelockUnexpectedOperationState` (`0x5ead8eb5`); a
   direct re-add from the timelock reverts `StrategyManagerStrategyAlreadyRegistered`
   (`0x5dfc84ba`).

## Cancelling

Either the DAO Safe or the Security Safe may `cancel(<operation id>)` on the timelock for
either operation independently, any time before that operation executes:

- `cancel(0x345dfba96cc9da6d54b34e07a99bdc594eac68248b26cb292a6c0d18a5e7eca8)` — 0.3% pool strategy
- `cancel(0xf340bad15fd7770a796eb38047ff1170e898b4f5541a656cd5a0bb828ea50990)` — 0.01% pool strategy

After execution, `removeStrategy` / `forceRemoveStrategy` (`ADMIN_ROLE`) unwind a
registered strategy; `setStrategyWeights` / `setDepositWeight` / `setWithdrawalWeight`
(`ADMIN_ROLE`) retune weights without removing it.
