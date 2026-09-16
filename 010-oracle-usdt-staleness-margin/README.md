# 010 — Oracle: add jitter margin to USDT staleness bound

**Status:** ✅ **Executed on mainnet** — 2026-09-16 11:44:47 UTC. Scheduled via DAO Safe
nonce 19 on 2026-09-14 10:04:59 UTC (tx
`0x838950378422d1988b28335eed20aed86d0cc9059d389a34ca227ddc5c45a994`); the 48h delay
elapsed 2026-09-16 10:04:59 UTC and the operation was executed ~1h40m later.
**Operation id:** `0xa2e19cb9b60ce55ee8cf1fc86dcf7cfffcb1118ed18b87f6e802ef3e17ab4c6e`
(recomputable via `hashOperation` with the predecessor and salt below — recomputed and
verified against a mainnet fork).

## What it changes

One call to `Oracle.updateUsdFeedInfo(address token, address feed, uint256 staleness)` on
`0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26`, **re-passing the existing feed address** for
USDT and changing only `stalenessInterval`:

| Token | Feed (unchanged) | Staleness: before → after | Chainlink heartbeat |
|---|---|---|---|
| USDT `0xdAC17F958D2ee523a2206206994597C13D831ec7` | `0x3E7d1eAB13ad0104d2750B8863b489D65364e32D` (USDT/USD) | `86400` → **`88200`** | 86400 (24h) |

Because the feed address is unchanged, `_upsertFeed` takes the update path (not the
first-add path): the call emits `UsdStalenessIntervalUpdated(USDT, 86400, 88200)` only — no
re-check of feed decimals, no membership change (`isTokenSupported(USDT)` stays `true`
throughout).

**What it does NOT do:** touches no other registry key, no role, no feed *address* (only
the staleness number), no other token ([008](../008-oracle-staleness-margin/)'s
WETH/`address(0)`/USDC margins are a separate, already-scheduled proposal). It does not
touch `StrategyManager.isSupportedERC20(USDT)` (set by [007](../007-strategy-manager-supported-usdt/),
already executed).

## Why

[006](../006-oracle-usdt-feed/) registered the USDT/USD feed with `stalenessInterval` set
to **exactly** the feed's own heartbeat (`86400`, 24h), with zero jitter margin — the same
gap [008](../008-oracle-staleness-margin/) is fixing for WETH, native ETH, and USDC. A
Chainlink round that lands even a second past its nominal 24h heartbeat (ordinary
block-time variance, not a stalled feed) makes `getUsdPrice(USDT)` /
`getUsdPriceWithStalenessCheck(USDT)` revert with `OracleStalePrice` even though the feed
is healthy. This proposal adds the same `+1800` (30 min) margin 008 chose for USDC's
23h-heartbeat feed — proportionate to USDT's 24h heartbeat, and enough to absorb normal
jitter without materially loosening the check (a feed that actually stops updating still
fails closed well within the added margin).

**Timing:** proposed independent of any strategy activity — USDT is whitelisted on the
StrategyManager ([007](../007-strategy-manager-supported-usdt/)) but no strategy routes it
yet, so this only changes how tolerant an already-live feed is to normal timing variance,
not what it prices or whether it's supported.

## No predecessor

`updateUsdFeedInfo` checks only: caller holds `ADMIN_ROLE`, `_priceFeed != address(0)`,
`_stalenessInterval != 0`, and (only on a feed-address change, which this is not)
`feed.decimals() <= 18`. It does not depend on any other timelock operation, so
`predecessor` is `0x00…00`.

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
| `data` | `0x8eff1c3c000000000000000000000000dac17f958d2ee523a2206206994597c13d831ec70000000000000000000000003e7d1eab13ad0104d2750b8863b489d65364e32d0000000000000000000000000000000000000000000000000000000000015888` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0x801e90834d72db816c7d5768fe725fe5d14f3ea7cb5a30e91410234aa42a70b5` |
| `delay` | `172800` (48h) |

`data` decodes to `updateUsdFeedInfo(address,address,uint256)` (selector `0x8eff1c3c`) with
`_token = USDT`, `_priceFeed = 0x3E7d…e32D` (unchanged), `_stalenessInterval = 88200`.

The salt derives from `keccak256("everstrat/oracle/usd-feed/usdt-staleness/2026-09-13")` —
a real, distinct per-proposal salt. Predecessor and salt must be reused **byte-for-byte**
at execute time.

## Verification performed (forked mainnet, before signing, 2026-09-13)

Fork at block 25970961. Starting state: `getUsdFeedInfo(USDT) == (0x3E7d…e32D, 86400)`,
`isTokenSupported(USDT) == true`, `Registry.hasRole(ADMIN_ROLE, timelock) == true`, op id
unused on-chain (`getOperationState == 0` Unset). Feed probed live on mainnet:
`description() == "USDT / USD"`, `decimals() == 8`, `latestRoundData` ~1h8m old (well
within the 24h heartbeat), price `~0.9997`.

1. `hashOperation(Oracle, 0, <data>, 0, salt)` reproduces the operation id above.
2. Direct `updateUsdFeedInfo(USDT, …)` from an unrelated EOA — **reverts**
   `RegistryClientMissingRole(ADMIN_ROLE)` (`0x4d616cff`).
3. DAO Safe → `schedule` with the exact calldata above — **success**, 57,276 gas.
   `CallScheduled` + `CallSalt` emitted for the operation id above.
4. `execute` before the delay — **reverts** `TimelockUnexpectedOperationState` (`0x5ead8eb5`).
5. Warp 48h + 1s. `execute` **from an unrelated EOA** — **success**, 61,868 gas — cheap:
   same feed address, staleness-only update, no `_validateFeedDecimals` re-check.
   `UsdStalenessIntervalUpdated(USDT, 86400, 88200)` emitted.
6. Post-state: `getUsdFeedInfo(USDT) == (0x3E7d…e32D, 88200)`. `isTokenSupported(USDT)`
   unchanged (`true`) throughout.
7. Re-`execute` — reverts `TimelockUnexpectedOperationState` (`0x5ead8eb5`); a direct
   identical re-update (same feed, same staleness) from the timelock reverts
   `OracleNothingToUpdate` (`0xf00155ca`).

`getUsdPriceWithStalenessCheck(USDT)` reverts on the warped fork immediately after
execute — the 48h time jump outruns even the widened bound. Fork artifact only, same as
[008](../008-oracle-staleness-margin/#verification-performed-forked-mainnet-before-signing):
on mainnet the underlying Chainlink round is fresh (verified `latestRoundData` ~1h8m old
at proposal time).

## On-chain schedule (mainnet)

| Field | Value |
|---|---|
| Scheduled via | DAO Safe `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9`, nonce 19 |
| Schedule tx | `0x838950378422d1988b28335eed20aed86d0cc9059d389a34ca227ddc5c45a994` |
| Executed (Safe) | 2026-09-14 10:04:59 UTC — 3/4 signatures |
| Operation state | `Waiting` (re-checked 2026-09-15 12:40 UTC) |
| Executable from | 2026-09-16 10:04:59 UTC |
| Execute (permissionless) | `02-execute.json` — anyone with gas, once `Ready` |

## On-chain execution (mainnet)

| Field | Value |
|---|---|
| Executed (UTC) | 2026-09-16 11:44:47 |
| Transaction | `0xfa6d1d560c00bc60ee0d3a479b8b25bd821e9173bc91293f2ceaaff0e6023e3a` |
| Block | 25989820 |
| Gas | 61,868 |
| Caller | `0x046E01eE…a899D7` (permissionless — `EXECUTOR_ROLE` is `address(0)`, no role needed) |

**Effect verified on-chain after execution** — `Oracle.getUsdFeedInfo(USDT)`:

| Field | Now | Before |
|---|---|---|
| Price feed | `0x3E7d1eAB13ad0104d2750B8863b489D65364e32D` (unchanged) | same |
| `stalenessInterval` | **88200** | 86400 |

## Cancelling

Either the DAO Safe or the Security Safe may call
`cancel(0xa2e19cb9b60ce55ee8cf1fc86dcf7cfffcb1118ed18b87f6e802ef3e17ab4c6e)` on the timelock
at any point before execution. After execution, `updateUsdFeedInfo` again to change the
feed or staleness further.
