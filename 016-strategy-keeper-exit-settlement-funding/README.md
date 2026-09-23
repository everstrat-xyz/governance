# 016 — StrategyKeeperExecutor: `minWithdrawETH` 0.01 → 0.0001 ETH, `controllerReserveETH` 0 → 0.05 ETH

**Status:** 🟡 **Proposed** — queued in the DAO Safe at **nonce 25**, 1 of 3 confirmations, not
executed. `safeTxHash = 0x3fe47bc2e2575bece7f2b2a7afaade84b43ed0925dcc05582d087d2cfdd8f637`,
submitted 2026-09-23 00:50:49 UTC by owner `0xF412F1A5d22f08FBD406D3B2B52e80336fa8E149` (who also
gave the first confirmation). It is a `multiSend` delegatecall to `MultiSendCallOnly`
`0x9641d764fc13c8B624c04430C7356C1C7C8102e2`, value 0. Its two inner calls, decoded from the raw
stored bytes, are plain calls to the timelock, and each one's calldata matches
`01-schedule-raw.json` **byte-for-byte**. Neither operation is scheduled yet
(`getOperationState == 0` for both at block 26036723).
**Operation id (`minWithdrawETH`):** `0xc9c3bc8473e7a53e7cd69095ef3bc8e9f6d6453f292711c5636f1d195e5f5725`
**Operation id (`controllerReserveETH`):** `0x826b1bdb0c36a7b9128c42a3261c21ba98f7920034dc00e8b81282f8566a02d5`
(both `hashOperation(StrategyKeeperExecutor, 0, <setter>, 0x00…00, salt)`, recomputed locally and
reproduced by the mainnet timelock's own `hashOperation`)

## What it changes

Two policy knobs on the StrategyKeeperExecutor `0xE94F714fbBF85c421b6FB80D3Ce0DFA20110EA29`, both
of which decide how queued exits get their ETH:

| # | Call | Before | After | Event |
|---|---|---|---|---|
| 1 | `setMinWithdrawETH(1e14)` | `0.01` ETH | **`0.0001` ETH** | `MinWithdrawETHChanged(1e16, 1e14)` |
| 2 | `setControllerReserveETH(5e16)` | `0` | **`0.05` ETH** | `ControllerReserveETHChanged(0, 5e16)` |

- **`minWithdrawETH`** is the smallest Controller **shortfall** (in-window priced exit liability
  minus Controller ETH) that makes the keeper recommend `WithdrawShortfall`, i.e. pull ETH out of
  the strategies. A smaller shortfall is not funded automatically. It is the largest residual left
  unfunded.
- **`controllerReserveETH`** is ETH the keeper leaves idle on the Controller instead of depositing
  it (`_idleExcess = balance − (reserve + needsETH)`). It is **not** a spending lock:
  `QueueKeeperExecutor._affordableRequests` budgets against the full `controller.balance`, so the
  reserve is a **settlement float**. Any priced batch costing ≤ the reserve settles straight from
  the Controller, with no strategy withdrawal.

Out of scope: **`exitLiquidityTargetETH` stays `0`**, so no ETH is pushed to the AMM for immediate
exits yet. Every exit keeps going through the queue.

Both setters are `onlyAuthRole(ADMIN_ROLE)` (48h timelock). Neither setter has a precondition or
bound: `setMinWithdrawETH` accepts any value including `0` (unlike `minDepositETH` /
`minHarvestETH` / `minExitLiquidityTopUpETH`, which reject `0`), and `setControllerReserveETH`
accepts any value. No role change, no code change, no fund movement at execution.

## Why

Proposed by **Vadyusha** (2026-09-22/23).

**`minWithdrawETH` → 0.0001.** At `0.01` it was 10× `AMM.minBatchExitETH` (`0.001`). With the
Controller empty, around ten minimum-size queued exits had to pile up before the keeper would
fund any of them. Once a batch is priced its liability is committed: NAV deducts it, supply
deducts the escrow, and the user cannot cancel within `MAX_BATCH_PROCESSING_TIME` (3 days). A
shortfall under the threshold is therefore not deferred but left unpaid. The request runs out
the 3-day window, `pullRequest` reverts `ExitQueueBatchExpired`, and the user has to
`closeRequest` to get their EVE back, with no ETH. The proposer's reasons for `0.0001`:

- `minWithdrawETH` *is* the residual the protocol tolerates leaving unfunded. A smaller number
  means a smaller permitted residual, which is safer.
- It makes the manual fix for a stuck shortfall cheaper. `Controller.receive()` is unguarded, so
  anyone can unblock a stranded batch by sending the missing ETH straight to the Controller. That
  takes no role and no timelock, and the queue keeper settles on its next tick. At `0.0001`, that
  top-up is never more than 0.0001 ETH.

**`controllerReserveETH` → 0.05.** Once the keeper funds more shortfalls automatically, the
reserve decides how often that funding has to touch the LP positions. It keeps ordinary exits off
the strategies entirely. The 0.05 ETH figure came out of review. The effective idle band is
`reserve … reserve + minDepositETH` (0.05–0.15 ETH, about 4–11% of NAV). The Controller already
holds 0.06 ETH idle today (excess 0.06 < `minDepositETH` 0.1), so this makes the current state
official rather than adding new drag.

**Why the reserve comes before exit liquidity.** `ProvideExitLiquidity` ranks above
`DepositExcess` in the checker, and both draw from the same `_idleExcess`. With a zero reserve,
turning exit liquidity on would move everything above the *currently priced* liability into the
AMM. Nothing would be left to settle the *next* batch once it is priced. With the reserve set
first, the AMM top-up can only take ETH above it.

Mainnet state at block 26036344: NAV 1.3081 ETH; Controller 0.06 ETH; AMM `freeBalance` 0;
strategies' withdrawal weights 40 / 15 / 45; `minBatchAge` 1 day; batches 1–3 priced with no
requests, batch 4 open.

## No predecessor

The two setters write separate storage slots and do not depend on each other or on any other
scheduled operation, so `predecessor = 0x00…00`. They can execute in either order. Unrelated to
the pending [014](../014-strategy-manager-performance-fee-bps/).

## Transactions

A DAO Safe schedule batch with 2 entries, wrapped by `MultiSendCallOnly`
`0x9641d764fc13c8B624c04430C7356C1C7C8102e2`, followed by permissionless execution.

| # | Call | Target | Function | Op id |
|---|---|---|---|---|
| 1 | `schedule` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` | `schedule(StrategyKeeperExecutor, 0, setMinWithdrawETH(1e14), 0x0, salt, 172800)` | `0xc9c3bc84…5e5f5725` |
| 2 | `schedule` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` | `schedule(StrategyKeeperExecutor, 0, setControllerReserveETH(5e16), 0x0, salt, 172800)` | `0x826b1bdb…566a02d5` |

`02-execute.json` holds the matching two `execute` calls. `01-schedule-raw.json` is the same
batch as raw calldata (selector `0x01d5062a`).

### Parameters

| Field | Value |
|---|---|
| Timelock | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (48h) |
| `target` (both) | StrategyKeeperExecutor `0xE94F714fbBF85c421b6FB80D3Ce0DFA20110EA29` |
| `value` | `0` |
| `data` #1 | `setMinWithdrawETH(100000000000000)` = `0x8797794600000000000000000000000000000000000000000000000000005af3107a4000` |
| `data` #2 | `setControllerReserveETH(50000000000000000)` = `0x4c5808dc00000000000000000000000000000000000000000000000000b1a2bc2ec50000` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` (shared) | `0x61b4eac54f186ab4517968691742a035c777a967ba43bb4754b66f67293f53f4` = `keccak256("everstrat/keeper/exit-settlement-funding/2026-09-23")` |
| `delay` | `172800` (48h, the enforced minimum) |

## Verification performed

Mainnet fork (anvil at block ~26036303, real deployed contracts), 2026-09-23:

| Step | Result |
|---|---|
| non-admin EOA calls either setter directly | revert `RegistryClientMissingRole(ADMIN_ROLE)` `0x4d616cff` (both) |
| DAO Safe calls `schedule(...)` ×2 | OK, gas **56,027** each (measured as separate txs, not through MultiSend); `CallScheduled` + `CallSalt` emitted; state `1` (Waiting) |
| `execute(...)` before the 48h delay | revert `TimelockUnexpectedOperationState` `0x5ead8eb5` |
| after `evm_increaseTime 172801` | state `2` (Ready) for both |
| `execute(...)` ×2 from an unrelated EOA | OK, gas **52,670** (`minWithdrawETH`) / **69,638** (`controllerReserveETH`, zero → non-zero slot) |
| events | `MinWithdrawETHChanged(1e16, 1e14)` and `ControllerReserveETHChanged(0, 5e16)`, decoded from the logs |
| getters after | `minWithdrawETH()` **1e16 → 1e14**; `controllerReserveETH()` **0 → 5e16**; state `3` (Done) |
| `execute(...)` again / `schedule(...)` again | revert `0x5ead8eb5` (both) |
| JSON | all three files parse; each `01-schedule-raw.json` `data` equals a fresh `cast calldata` built from `01-schedule.json`; `02-execute.json` inputs match the schedules and rehash to both op ids |

**Checker honours the reserve** (separate fork, no time warp so the Chainlink feeds stay fresh;
timelock impersonated to set the reserve):

| Controller | Reserve | `strategyUpkeepStatus()` |
|---|---|---|
| 0.06 (live) | 0 | `None` |
| 0.2 | 0 | `DepositExcess`, **0.2** |
| 0.2 | 0.05 | `DepositExcess`, **0.15** |
| 0.12 | 0.05 | `None` (excess 0.07 < `minDepositETH` 0.1) |

**Dust `WithdrawShortfall` works.** The StrategyKeeperExecutor (which holds `KEEPER_ROLE`) was
impersonated to call `Controller.withdrawFromStrategies(x)` with an explicit 15M gas limit:

| `x` | Status | Gas | Controller receives | Per strategy (40 / 15 / 45) |
|---|---|---|---|---|
| 0.0001 | OK | 2,703,206 | 0.0001 exactly | all three `FundsWithdrawnFromStrategy` |
| 0.001 | OK | 2,709,523 | 0.001 exactly | all three `FundsWithdrawnFromStrategy` |
| 0.01 | OK | 2,696,203 | 0.01 exactly | all three `FundsWithdrawnFromStrategy` |

Gas is flat (~2.70M) regardless of amount, because each strategy withdraw with no idle ETH does
a full remove → convert → rebalance → re-add of its position. NAV rose ~0.00042 ETH in every
case, about the same whatever the amount. That is LP fees surfacing from the withdraw's position
poke, which hides the swap cost, so churn cost cannot be isolated from NAV here.

**Mainnet keeper history.** Every `DepositExcess` Mimic has run through this executor (read from
`StrategyUpkeepPerformed` logs, 2026-09-17 → 09-22). Each one deposited into all three strategies,
and none emitted `StrategyDepositFailed`:

| tx | Gas limit (Mimic) | Gas used | Headroom | Deposited |
|---|---|---|---|---|
| `0xd19f18b2e6fdaab5899c0ba913bba0b6f5c4bf9060f71bbd58ed2ce34ac1b38b` | 4,426,615 | 3,414,082 | 1.30× | 0.4345 ETH |
| `0x01a87751fc3b611d59bc7bcce4083f1e3fd9fa4b1d215e047f41be703c694b9c` | 4,402,449 | 3,103,323 | 1.42× | 0.3197 ETH |
| `0xf9d5ab12010a8e70dc0f5dd14105b3ef349a3e032bdede7565fe92fc8920a391` | 4,123,810 | 2,873,087 | 1.44× | 0.3200 ETH |
| `0x655c071788fca0568ff6172944c73bbb3317013fe3c633586822edfc9a69f943` | 4,393,219 | 3,048,384 | 1.44× | 0.2038 ETH |

## Risks

- **More LP churn, bounded.** Any shortfall ≥ 0.0001 ETH that the Controller cannot cover now
  triggers a withdraw across every strategy with a non-zero withdrawal weight. That costs ~2.7M
  gas (≈ 0.0002 ETH at the 0.08 gwei base fee seen on 2026-09-23, paid by the Mimic account) plus
  a remove/re-add of all three positions, whatever the size of the shortfall. `needsETH` only
  grows when a batch is priced (`minBatchAge` = 1 day), which limits how often this can happen.
  The reserve damps it: shortfalls only arise once a day's priced exits exceed the float.
- **Tight gas limits starve the last strategy. Seen on a fork, not in production.** When the gas
  limit is within a few percent of what the call needs (as with `cast`'s own estimate), the
  **last** strategy in StrategyManager's loop (WETH/USDT `0x3Fb6…6501`, weight 45) runs out of gas
  inside the `try/catch`. On withdraw the tx still succeeds but under-delivers:
  `StrategyWithdrawFailed` with an empty reason, and the Controller got 55% of the request (one
  run reverted outright). On deposit it is worse. After the `catch`, StrategyManager keeps only
  ~1/64 of the gas, which is too little to emit `StrategyDepositFailed` and return the leftover ETH,
  so its own frame runs out of gas and the whole tx reverts. With a 15M limit every run succeeded.
  Production does not show this. Mimic sets `perform` gas limits with **1.30–1.44× headroom** over
  gas used, and all four mainnet `DepositExcess` runs filled all three strategies (see
  *Mainnet keeper history* above). `WithdrawShortfall` has never run on mainnet because no exit
  has been queued. It needs less gas (~2.7M) than those deposits (2.9–3.4M), so the same headroom
  should cover it. Only transactions that emitted `StrategyUpkeepPerformed` were checked. Reverted
  `perform` calls through Mimic's shared contract were not searched.
- **Idle-ETH drag.** 0.05–0.15 ETH (~4–11% of NAV) sits outside the strategies. This is roughly
  today's actual idle level, but it is now deliberate.
- Reversible only through the same 48h `ADMIN_ROLE` path; there is no SECURITY override.

## Cancelling

Before execution, either the DAO Safe or the Security Safe may call `cancel(opId)` on the
timelock for either operation (`0xc9c3bc84…5e5f5725` and/or `0x826b1bdb…566a02d5`). They are
independent, so one can be cancelled and the other left to execute. After execution, revert with
`setMinWithdrawETH(1e16)` / `setControllerReserveETH(0)` through a new 48h `ADMIN_ROLE` proposal.
