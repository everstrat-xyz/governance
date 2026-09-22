# 014 — StrategyManager: set `performanceFeeBps` 0 → 1500 (15%)

**Status:** ⏳ **Scheduled on mainnet** (tx
`0x4d9828d1a877b12144fd149f6dc81a448e7077118247496282e686d965a5e881`, block 26033351,
2026-09-22 13:32:59 UTC, DAO Safe nonce 23). Ready at **2026-09-24 13:32:59 UTC**
(`getTimestamp` = `1790256779`); `getOperationState == 1` (Waiting) as of 2026-09-22 23:03 UTC.
No predecessor. `performanceFeeBps()` is still `0` until someone calls `execute`.
**Operation id:** `0xc4302e528a0a993eb9cc433dfc35ddace37bfef0cc3d2e7b97207d397f74ea46`
(`hashOperation(StrategyManager, 0, setPerformanceFeeBps(1500), 0x00…00, salt)` — recomputed and
verified on mainnet).

## What it changes

`StrategyManager.setPerformanceFeeBps(1500)` — the protocol's performance fee on strategy LP fees
goes from **0 to 15%**. Fees accrue to `daoTreasury()`, which is
`0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9` — **the DAO Safe itself**, not an ops address.
Emits `PerformanceFeeBpsChanged(0, 1500)`.

Bounds: `setPerformanceFeeBps` is `onlyAuthRole(ADMIN_ROLE)` (48h timelock — never SECURITY or
KEEPER) and accepts `0 – MAX_PERFORMANCE_FEE_BPS` (`2000`, read on mainnet).

## Why

Requested by **Arseny** (2026-09-21): *"set it to 15 and open a proposal on safe"*, after
Vadyusha reported that `performanceFeeBps` is still `0`, so the performance fee **accrues but
cannot be harvested**. A first draft of this proposal used 1000 (10%); it was re-cut to 1500
before filing, and only 1500 was ever signed.

With `performanceFeeBps = 0` the fee share still accrues but `StrategyManager` has no way to
settle it. Setting a non-zero rate is what unlocks collection — and per audit finding **M15**
(`_setPerformanceFeeBps` does not settle outstanding fees first) the new rate applies to the
**entire previously uncharged LP-fee base**, so the retroactive period becomes collectable too.

## No predecessor

The fee setter has no on-chain dependency on any other scheduled operation, so
`predecessor = 0x00…00`.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` with `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` after the delay |

`01-schedule-raw.json` is the same transaction as raw calldata (selector `0x01d5062a`).

### Parameters

| Field | Value |
|---|---|
| Timelock | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (48h) |
| `target` | StrategyManager `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` |
| `value` | `0` |
| `data` | `setPerformanceFeeBps(1500)` = `0x9f0caac900000000000000000000000000000000000000000000000000000000000005dc` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0x2115f78dfb605f8591296be39bd99ddad345076ecbe6c2e21ec2d320472598c0` = `keccak256("everstrat/sm/performance-fee-bps/2026-09-21")` |
| `delay` | `172800` (48h, the enforced minimum) |

## Verification performed

Mainnet fork (anvil, real deployed contracts), 2026-09-21:

| Step | Result |
|---|---|
| non-admin calls `setPerformanceFeeBps(1500)` directly | revert `RegistryClientMissingRole` `0x4d616cff` |
| DAO Safe calls `schedule(...)` | OK, gas **56,071** |
| `execute(...)` before the 48h delay | revert `TimelockUnexpectedOperationState` `0x5ead8eb5` |
| `execute(...)` from an unrelated EOA after the delay | OK, gas **75,111** |
| `performanceFeeBps()` after | **0 → 1500** |
| `execute(...)` again | revert `0x5ead8eb5` (already done) |
| `hashOperation(...)` | reproduces `0xc4302e52…f74ea46` exactly |

## Risks

- **Retroactive.** The whole previously uncharged fee base becomes collectable at 15% (audit
  M15). A later *decrease* is likewise retroactive, so if the rate is ever lowered, settle first.
- Reversible only through the same 48h `ADMIN_ROLE` path — there is no SECURITY override.

## On-chain schedule (mainnet)

| Field | Value |
|---|---|
| Proposed by | off-chain Safe delegate `0x1483E048a76A93a3A59bBfA6d60471eA4990e922`; `proposer` recorded as its delegator `0x4A2D30c7b9f7907D580f9A1902D42dd78B21F0d2`; submitted 2026-09-21 16:02:21 UTC |
| Scheduled via | DAO Safe `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9`, nonce 23 (single `schedule`) |
| safeTxHash | `0x4fba6636d0eb3a605e38ef34a4b78274bffe725c00bb3b6bdbc9055a31dd0004` |
| Signatures | 3 of 4 — `0x4A2D…F0d2` 12:26:38 · `0x1Efb…9a46` 12:44:28 · `0xF412…E149` 13:26:32 UTC (2026-09-22) |
| Execute (Safe) | 2026-09-22 13:32:59 UTC — tx `0x4d9828d1a877b12144fd149f6dc81a448e7077118247496282e686d965a5e881`, block 26033351, gas 191,994; relayed by owner `0x4A2D…F0d2` through the MetaMask `DelegationManager` `0xdb9B1e94…47dB3` (`redeemDelegations`) |
| Events | `CallScheduled` + `CallSalt` for `0xc4302e52…f74ea46`; `ExecutionSuccess(0x4fba6636…dd0004)` |
| Calldata check | Safe tx `data` equals `01-schedule-raw.json` byte-for-byte |
| Ready at | `getTimestamp` = `1790256779` = **2026-09-24 13:32:59 UTC** |
| Operation state | `1 (Waiting)` (re-checked 2026-09-22 23:03 UTC, block 26036183) |
| Execute (permissionless) | `02-execute.json` — anyone with gas, once `Ready` |

## Cancelling

Before execution, either the DAO Safe or the Security Safe may call
`cancel(0xc4302e528a0a993eb9cc433dfc35ddace37bfef0cc3d2e7b97207d397f74ea46)` on the timelock.
After execution, `setPerformanceFeeBps(<new>)` via a new 48h `ADMIN_ROLE` proposal.
