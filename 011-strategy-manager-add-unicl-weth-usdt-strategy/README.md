# 011 — StrategyManager: register the UniCL WETH/USDT 0.3% strategy

**Status:** 📝 **Draft — not yet submitted.** Payload below is simulated against forked
mainnet but has not been signed or scheduled.
**Operation id:** `0x4b6a7a40449523fa21d7f268807beddb8a670319b4bce25ca142d9696fcd9fd6`
(recomputable via `hashOperation` with the predecessor and salt below — recomputed and
verified against a mainnet fork).

## What it changes

One call to `StrategyManager.addStrategy(address strategy, uint8 depositWeight, uint8 withdrawalWeight)`
(selector `0xdca0c48f`) on `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9`, registering the
already-deployed `UniCLStrat` instance against the USDT/WETH Uniswap V3 pool:

| Strategy | Pool (fee tier) | `depositWeight` | `withdrawalWeight` | `maxTotalNAV` |
|---|---|---|---|---|
| `0x3Fb6B9174427CA4FF38728398F4F4CB526F66501` | `0x4e68Ccd3E89f51C3074ca5072bbAC773960dFa36` — WETH/USDT **0.3%** | `45` | `45` | 4500 ETH |

`addStrategy` adds the strategy to `_strategies`, sets both weights (capped at
`MAX_DEPOSIT_WEIGHT` / `MAX_WITHDRAWAL_WEIGHT` = 100 each), and calls
`Converter.grantCallerRole(strategy)` so the strategy can call the Converter's swap
functions — no extra role grant needed since `grantCallerRole` is gated to the
StrategyManager contract itself (`onlyAuthContract(Auth.STRATEGY_MANAGER)`).

**What it does NOT do:** deploys no code (the `UniCLStrat` instance already exists on
mainnet — bytecode-only deploy via `DeployUniCLStrat.s.sol`, verified below), moves no
funds (`totalDeposited() == 0`, `navInETH() == 0`), registers no Oracle feed, changes no
Converter allowlist entry, touches no other strategy (including
[009](../009-strategy-manager-add-unicl-strategies/)'s two WETH/USDC strategies, which
this proposal is independent of).

## Why

This strategy pairs USDT (whitelisted on the StrategyManager via
[007](../007-strategy-manager-supported-usdt/), Oracle-priced via
[006](../006-oracle-usdt-feed/)) with WETH, routed through the whitelisted
`UniswapV3ConverterAdapter` ([005](../005-converter-allow-univ3-adapter/)) — the same
pattern [009](../009-strategy-manager-add-unicl-strategies/) used for the WETH/USDC pair.
`addStrategy` is the final go-live step — per `DeployUniCLStrat.s.sol`'s own
deployment-order comment, it always runs on the 48h admin timelock, after the adapter
allowlist and deploy steps, never through a bootstrap key.

Weights: `45`/`45`, between 009's `40`/`40` (WETH/USDC 0.3%, 1000 ETH cap) and a
meaningfully higher share reflecting this strategy's larger `maxTotalNAV` (4500 ETH),
without routing a majority of deposit/withdrawal flow to a single, as-yet-unproven
strategy on day one.

**Timing:** the strategy reports `totalDeposited() == 0` and is unregistered — this is the
first time it becomes reachable by the protocol; no existing position is affected.

## No predecessor — but a strong recommendation

`addStrategy` has no on-chain dependency on [010](../010-oracle-usdt-staleness-margin/)
(the USDT Oracle staleness-margin proposal): it only checks `ADMIN_ROLE`, `whenNotPaused`,
non-zero address, has-code, and not-already-registered. `predecessor` is not set to 010's
operation id, and this proposal can execute before, after, or independent of it.

That said, 010 is **strongly recommended** to land first (or alongside). This strategy's
swap path cross-checks its Uniswap TWAP quote against the Oracle's `USDT / USD` and
native-ETH price within `MAX_QUOTE_DEVIATION_BPS` — and the USDT feed currently sits at
**exactly** its Chainlink heartbeat (`86400`) with zero jitter margin (the
[006](../006-oracle-usdt-feed/) issue 010 fixes). Once this strategy holds real capital, an
Oracle-side `OracleStalePrice` revert on ordinary heartbeat jitter would block a deposit,
withdrawal, or rebalance that has nothing to do with the strategy itself. Landing 010 first
removes that failure mode before it can bite; it is not required for `addStrategy` to
succeed and is not encoded as a `predecessor` here.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` with `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` after the delay |

`01-schedule-raw.json` is the same transaction as raw calldata (selector `0x01d5062a`), for
when the Safe UI cannot resolve the timelock ABI.

### Parameters

| Field | Value |
|---|---|
| `to` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (admin timelock) |
| `target` | `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` (StrategyManager) |
| `value` | `0` |
| `data` | `0xdca0c48f0000000000000000000000003fb6b9174427ca4ff38728398f4f4cb526f66501000000000000000000000000000000000000000000000000000000000000002d000000000000000000000000000000000000000000000000000000000000002d` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0xa20b7d1644726d67c06b27adfa7ab090cae091dba915757b89def8905cab2e5d` |
| `delay` | `172800` (48h) |

`data` decodes to `addStrategy(address,uint8,uint8)` (selector `0xdca0c48f`) with
`_strategy = 0x3Fb6…6501`, `_depositWeight = 45` (`0x2d`), `_withdrawalWeight = 45` (`0x2d`).

The salt derives from
`keccak256("everstrat/strategy-manager/add-strategy/unicl-weth-usdt-0.3pct/2026-09-14")`.
Predecessor (zero) and salt must be reused **byte-for-byte** at execute time.

## Verification performed (forked mainnet, before signing, 2026-09-14)

Fork at block 25971132. Starting state: strategy unregistered,
`StrategyManager.paused() == false`, `strategyCount() == 0`, op id unused on-chain
(`getOperationState == 0` Unset).

**Deployed-strategy sanity, mainnet:**

1. `0x3Fb6…6501` byte-length-identical (outside immutables) to the compiled `UniCLStrat`
   artifact — 24,391 bytes, matches 009's two deployments exactly.
2. `token0() == WETH`, `token1() == USDT`, `pairedToken() == USDT`, `weth() == WETH`,
   `factory()` == canonical Uniswap V3 factory.
3. `pool()` verified against `factory.getPool(WETH, USDT, pool.fee())`:
   `0x4e68Ccd3…dFa36` at fee `3000` (0.3%, `tickSpacing = 60`) — matches the address given
   for registration exactly.
4. Pool `observationCardinality` (360) comfortably exceeds `MIN_OBSERVATION_CARDINALITY =
   150` at the strategy's configured `twapInterval = 1800` / `shortTwapInterval = 60` (both
   at the contract's floor values, same as 009's strategies).
5. `swapAdapter() == 0x0844580a121124CAEc6Cf4A933aac401813cCde5`, the
   [005](../005-converter-allow-univ3-adapter/) adapter — confirmed still whitelisted
   (`Converter.isAdapterAllowed == true`).
6. `paused() == false`, `totalDeposited() == 0`, `navInETH() == 0` — fresh, unfunded.
7. Cross-dependencies confirmed live: `Oracle.isTokenSupported(USDT) == true`
   ([006](../006-oracle-usdt-feed/), executed), `StrategyManager.isSupportedERC20(USDT) ==
   true` ([007](../007-strategy-manager-supported-usdt/), executed).

**Timelock simulation:**

1. `hashOperation(StrategyManager, 0, addStrategy(strategy, 45, 45), 0, salt)` reproduces
   the operation id above.
2. Direct `addStrategy(strategy, 45, 45)` from an unrelated EOA — **reverts**
   `RegistryClientMissingRole(ADMIN_ROLE)` (`0x4d616cff`).
3. DAO Safe → `schedule` with the exact calldata above — **success**, 57,036 gas.
   `CallScheduled` + `CallSalt` emitted for the operation id above.
4. `execute` before the delay — **reverts** `TimelockUnexpectedOperationState`
   (`0x5ead8eb5`).
5. Warp 48h + 1s. `execute` **from an unrelated EOA** — **success**, 349,592 gas (matches
   009's WETH/USDC 0.3% strategy registration cost exactly).
   `DepositWeightUpdated`/`WithdrawalWeightUpdated` (both `0 → 45`) and
   `CONVERTER_CALLER_ROLE` grant emitted.
6. Post-state: `isStrategyRegistered(strategy) == true`; `strategyCount() == 1`;
   `depositWeight`/`withdrawalWeight == (45, 45)`; `Registry.hasRole(CONVERTER_CALLER_ROLE,
   strategy) == true`.
7. Re-`execute` — reverts `TimelockUnexpectedOperationState` (`0x5ead8eb5`); a direct
   re-add from the timelock reverts `StrategyManagerStrategyAlreadyRegistered`
   (`0x5dfc84ba`).

## Cancelling

Either the DAO Safe or the Security Safe may call
`cancel(0x4b6a7a40449523fa21d7f268807beddb8a670319b4bce25ca142d9696fcd9fd6)` on the
timelock at any point before execution.

After execution, `removeStrategy` / `forceRemoveStrategy` (`ADMIN_ROLE`) unwind the
registered strategy; `setStrategyWeights` / `setDepositWeight` / `setWithdrawalWeight`
(`ADMIN_ROLE`) retune weights without removing it.
