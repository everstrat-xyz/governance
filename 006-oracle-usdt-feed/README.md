# 006 — Oracle USD price feed: add USDT

**Status:** 📝 **Draft — not yet submitted.** Payloads below are simulated against forked
mainnet but have not been signed or scheduled.
**Operation id:** `0x3aa9f59be4381618b4c684ca8155b837fa1718db26ef96c021ba4d775fe21a9a`
(`hashOperation(Oracle, 0, updateUsdFeedInfo(USDT, USDT/USD, 86400), predecessor, salt)`).

## What it changes

`Oracle.updateUsdFeedInfo(0xdAC17F958D2ee523a2206206994597C13D831ec7, 0x3E7d1eAB13ad0104d2750B8863b489D65364e32D, 86400)`
on the price oracle `0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26` — registers the Chainlink
`USDT / USD` reference feed for USDT and adds USDT to the Oracle's supported-token set.

| Field | Value |
|---|---|
| token | USDT `0xdAC17F958D2ee523a2206206994597C13D831ec7` (6 decimals) |
| feed | `0x3E7d1eAB13ad0104d2750B8863b489D65364e32D` — Chainlink `USDT / USD` (`description()` verified), 8 decimals |
| `stalenessInterval` | `86400` (24 h) |

`86400` is the Chainlink `USDT / USD` deviation/heartbeat interval exactly — the feed is
only guaranteed to write once per 24 h absent a price move, so a tighter bound would
reject a normally-cadenced round as stale. This is intentionally looser than the `82800`
[003](../003-oracle-usd-feeds/) chose for `USDC / USD`; the two feeds are configured
independently.

Before this change, `Oracle.getUsdFeedInfo(USDT)` and `getUsdPrice(USDT)` revert
`OracleTokenNotSupported` (`0x868fd74e`) — no feed is configured.

**What it does NOT do:** touches no registry key, no role, no other token, no pair feed. It
is a single first-time `UsdFeedAdded`; it rewrites nothing. It does **not** whitelist USDT
on the StrategyManager — that is a separate `addSupportedERC20(USDT)` proposal (the analog
of [004](../004-strategy-manager-supported-usdc/)), which this feed is a prerequisite for.

## Why

The protocol plans to run the UniCL strategy against the **WETH/USDT** Uniswap V3 pool in
addition to WETH/USDC. `UniswapV3ConverterAdapter` cross-checks every TWAP quote against
the Oracle, so a WETH↔USDT route needs the Oracle to price USDT in USD — exactly as the
WETH/USDC route needs the USDC feed from 003. `address(0)` (native ETH) is already priced
(Chainlink `ETH / USD`, set at deployment), so this feed completes the USDT side.

The feed is Chainlink's canonical mainnet `USDT / USD` aggregator; the staleness is set to
that feed's own heartbeat (see *What it changes*).

**Timing:** proposed while no strategy routes USDT yet, so registering the feed now has no
effect on any position or NAV.

## No predecessor

`updateUsdFeedInfo` checks only: caller holds `ADMIN_ROLE`, `_priceFeed != address(0)`,
`_stalenessInterval != 0`, and `feed.decimals() <= 18`. It does not depend on any other
timelock operation, so `predecessor` is `0x00…00`.

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
| `target` | `0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26` (Oracle) |
| `value` | `0` |
| `data` | `0x8eff1c3c000000000000000000000000dac17f958d2ee523a2206206994597c13d831ec70000000000000000000000003e7d1eab13ad0104d2750b8863b489d65364e32d0000000000000000000000000000000000000000000000000000000000015180` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0x5c5ed1cb0184b918a42a3209e35dacf347197d609fe10ee6327d60cea78496dd` |
| `delay` | `172800` (48h — the enforced minimum) |

`data` decodes to `updateUsdFeedInfo(address,address,uint256)` (selector `0x8eff1c3c`) with
`_token = USDT`, `_priceFeed = 0x3E7d…e32D`, `_stalenessInterval = 86400`.

The salt derives from `keccak256("everstrat/oracle/usd-feed/usdt/2026-09-09")` — a real
per-proposal salt (003 used a zero salt; this restores the convention). Predecessor and
salt must be reused **byte-for-byte** at execute time.

## Verification performed (forked mainnet, before signing)

Fork at block 25942019. `Oracle.isTokenSupported(USDT) == false`,
`getSupportedTokens() == [address(0), USDC, WETH]`,
`Registry.hasRole(ADMIN_ROLE, timelock) == true`. Feed probed on mainnet:
`description() == "USDT / USD"`, `decimals() == 8`, `latestRoundData` fresh (~1.0000).

1. `hashOperation(Oracle, 0, <data>, 0, salt)` reproduces the operation id above.
2. Direct `updateUsdFeedInfo(USDT, …)` from an unrelated EOA — **reverts**
   `RegistryClientMissingRole(ADMIN_ROLE)` (`0x4d616cff`).
3. DAO Safe → `schedule` with the exact calldata above — **success**, 57,276 gas.
   `CallScheduled` + `CallSalt` emitted; `isOperationPending` true, ready-at `scheduled + 172800`.
4. `execute` before the delay — **reverts** `TimelockUnexpectedOperationState` (`0x5ead8eb5`).
5. Warp 48h + 1s. `execute` **from an unrelated EOA** — **success**, 157,434 gas.
   `UsdFeedAdded(USDT, 0x3E7d…e32D, 86400)` (`0x29cc2d51…`) + `CallExecuted` emitted, confirming
   execution is permissionless.
6. Post-state: `isTokenSupported(USDT) == true`, `getUsdFeedInfo(USDT) == (0x3E7d…e32D, 86400)`,
   `getSupportedTokens() == [address(0), USDC, WETH, USDT]`.
7. Re-`execute` and re-`schedule` the same op — both revert `TimelockUnexpectedOperationState`;
   a direct re-add with identical args reverts `OracleNothingToUpdate` (`0xf00155ca`).

`getUsdPrice(USDT)` on the warped fork reverts `OracleStalePrice` (`0xa9f73445`) — the
48 h time jump makes the real Chainlink round older than the 24 h bound. Not a defect: on
mainnet the feed round is fresh (verified `getUsdPrice(USDC)` returns `~1.0` live against
the same code path).

## Cancelling

Either the DAO Safe or the Security Safe may
`cancel(0x3aa9f59be4381618b4c684ca8155b837fa1718db26ef96c021ba4d775fe21a9a)` on the timelock
any time before execution. After execution: `updateUsdFeedInfo` again to change the feed or
staleness, or `removeToken(USDT)` (`ADMIN_ROLE`, 48h) to drop it.
