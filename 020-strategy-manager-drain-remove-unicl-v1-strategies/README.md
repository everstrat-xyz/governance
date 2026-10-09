# 020 — Drain and remove the four UniCL v1 strategies

**Status:** 📝 Draft — not yet submitted.
**Operation ids** (four independent `scheduleBatch` operations, each `predecessor` = 019):

| v1 strategy | Operation id |
|---|---|
| USDC/WETH 0.3% `0x5E12FFD5e0F69d37B581D06b68ad6c53609cac38` | `0xfa9e716cf5547d760c68e8758a4900a0644df7490af3fbee8a9ccf56ed7493d5` |
| USDC/WETH 0.01% `0x59C476c1817b23791d96F35b9f468E6d9A242252` | `0xe9438cb7438f47c22b1e74477c013d86870cb08f7080080844d3532dacef75c8` |
| WETH/USDT 0.3% `0x3Fb6B9174427CA4FF38728398F4F4CB526F66501` | `0xe680e96963678090ac5854516999bf7c770284e622e66c014e04c689b9265bd9` |
| UNI/WETH 0.3% `0x956fE55DB50527E75c1fcA679adE509AD89cAFDd` | `0x93ee86d5f12f2b739647a886d51e7dce3baeb77c02809d4c0d10fcf3b81f4cd2` |

All four recomputed locally and reproduced by the mainnet timelock's own `hashOperationBatch`;
`getOperationState == 0` (Unset) at block 26154876.

## What it changes

For each v1 strategy, one atomic timelock batch of three calls:

1. `Controller.withdrawFromStrategy(old, type(uint256).max)` — pulls the strategy's whole NAV to
   the Controller. StrategyManager caps the amount at `strategy.maxWithdrawal()` (= `navInETH()` while
   unpaused), so the call drains whatever the strategy holds **at execution time**; no amount goes
   stale over the 48h delay. It also settles the strategy's pending performance fee first.
2. The same call again — sweeps the residue the v1 code leaves after a full withdrawal (below).
3. `StrategyManager.forceRemoveStrategy(old)` — deregisters the strategy and revokes its
   `CONVERTER_CALLER_ROLE`. Emits `StrategyForceRemoved(old, reportedNAV, navReverted)`, which records
   exactly how much dust was written off.

Funds move strategy → Controller only; they stay inside the protocol and in NAV. With the v2 set
from [019](../019-strategy-manager-register-unicl-v2-strategies/) registered, the keeper's normal
`DepositExcess` path then deposits the Controller's ETH above `controllerReserveETH` (0.05 ETH,
[016](../016-strategy-keeper-exit-settlement-funding/)) into the v2 strategies by weight. After all
four batches, `strategies()` is exactly the v2 set.

## Why

Requested by **Vadyusha** (2026-10-09): the second half of the v1 → v2 migration; see 019 for what
v2 changes. Two measured reasons to drain rather than leave v1 running beside v2:

- **The v1 withdraw path leaves value behind on every full unwind.** On a fork, one full
  `withdrawFromStrategy` on the three original v1 strategies left 0.076–0.094% of their NAV in the
  contract (as WETH/USDC plus LP dust); v2's `4e782d7` fixes this. Keeping v1 registered means every
  large exit pays that.
- **Cost is small.** The whole migration — drain v1, re-deposit into v2 — moved `totalNAVInETH`
  from 3.402225 to 3.401513 ETH on the fork: **−0.000712 ETH (−0.021%)**.

The alternative — zero the v1 deposit weights and let exits drain them over time — avoids the swap
cost now but keeps the v1 unwind shortfall for every withdrawal, runs eight strategies for an
open-ended period, and still needs a removal proposal at the end.

## Why `forceRemoveStrategy`, and why two sweeps

`removeStrategy` requires `navInETH() ≤ MAX_NAV_RESIDUE` = **10 wei**. The v1 code cannot meet it:

| v1 strategy | NAV before | after 1 sweep | after 2 sweeps | `removeStrategy` |
|---|---|---|---|---|
| USDC/WETH 0.3% | 1.293611 ETH | 1,127,535,313,261,125 wei | 687,179,459,330 wei | revert `StrategyManagerStrategyNAVResidueTooHigh` `0x3b3fa6e8` |
| USDC/WETH 0.01% | 0.486186 ETH | 367,746,021,708,073 wei | 223,932,453,917 wei | revert `0x3b3fa6e8` |
| WETH/USDT 0.3% | 1.456202 ETH | 1,372,956,351,431,870 wei | 827,595,663,203 wei | revert `0x3b3fa6e8` |
| UNI/WETH 0.3% | 0.116228 ETH | 0 | 0 | OK |

So a `[withdraw, removeStrategy]` batch would revert forever for three of the four. The second sweep
recovers ≈ 99.9% of the first sweep's residue (≈ 0.0029 ETH across the three); `forceRemoveStrategy`
writes off what is left — about **5.2 × 10¹² wei (0.0000052 ETH) in total**. Because each batch is
atomic, `forceRemoveStrategy` only ever runs after both sweeps succeeded, so it cannot drop a strategy
that still holds real funds. UNI/WETH (the 2026-09-23 build from 017) drains to 0, and its second
sweep is a no-op (`min(max, maxWithdrawal() = 0) = 0` returns without calling the strategy); it uses
the same batch shape so all four behave alike. The written-off dust stays in the removed contracts.

## Dependency on 019 and 018

- **019 is the `predecessor`** of all four operations (`0xf558cda6…fc9ed6a6`), so `executeBatch` reverts
  `TimelockUnexecutedPredecessor` (`0x90a9a618`) until the v2 set is registered. Without it, 020 could
  leave the protocol with no strategies at all.
- **018 is required but not encoded.** `Controller.withdrawFromStrategy` is `KEEPER_ROLE`-only until
  [018](../018-controller-upgrade-v1.1.0/) (Controller v1.1.0) executes; before that every 020 batch
  reverts `RegistryClientMissingRole(KEEPER_ROLE)` (`0x4d616cff`) and simply stays `Ready` — a failed
  `executeBatch` does not consume the operation. 018 is scheduled (ready 2026-10-11 12:07:59 UTC),
  well before 020 can be. Only one predecessor fits; 019 was chosen because skipping it is the
  harmful case, while skipping 018 only reverts.
- The four 020 operations are independent of each other and may execute in any order.

Alternative: `predecessor = 0x00…00` on each, sequencing 018 → 019 → 020 by hand — the operation ids
change.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | 4 × `timelock.scheduleBatch(...)` (3 calls each), `delay = 172800` — wrapped by `MultiSendCallOnly` |
| 2 | `02-execute.json` | anyone | 4 × `timelock.executeBatch(...)` after 018 and 019 are done |

`01-schedule-raw.json` is the same four transactions as raw calldata (selector `0x8f2a0bb0`); both
encode to identical bytes. **Prefer the raw file** for batches (see 019).

### Parameters

| Field | Value |
|---|---|
| `targets` | `[Controller 0x9097868dcbda5a630729015DE1ddeCd66d74052c, Controller, StrategyManager 0x94916ab93C669E7c734f844dB019Ce9449a3b5C9]` |
| `values` | `[0, 0, 0]` |
| `payloads` | `[withdrawFromStrategy(old, 2²⁵⁶−1), withdrawFromStrategy(old, 2²⁵⁶−1), forceRemoveStrategy(old)]` — selectors `0xb53d0958`, `0xb53d0958`, `0x428ea195` |
| `predecessor` | `0xf558cda6b3831a4fb393278e368ff3b319c04696e37ebb034d31c463fc9ed6a6` (019) |
| `delay` | `172800` |

| v1 strategy | `salt` | Preimage |
|---|---|---|
| USDC/WETH 0.3% | `0xca5a07a0b6e53a95f818a74c68e6580bf7f914a6efe768891c7e9553a914b38f` | `everstrat/strategy-manager/drain-remove/unicl-v1-usdc-weth-0.3pct/2026-10-09` |
| USDC/WETH 0.01% | `0xfc137cde715162c3c8d120d0f0bc2232160e9387d7b9f7ba0d4830133d83524c` | `everstrat/strategy-manager/drain-remove/unicl-v1-usdc-weth-0.01pct/2026-10-09` |
| WETH/USDT 0.3% | `0x7b093cde77ceb92cd28fd9720ec0a5cc75a64756a33c19f462de212f03664c2e` | `everstrat/strategy-manager/drain-remove/unicl-v1-weth-usdt-0.3pct/2026-10-09` |
| UNI/WETH 0.3% | `0x623ba74160b472b961951bed443b3efca4ce980ef46601cd310c860244f7b1a7` | `everstrat/strategy-manager/drain-remove/unicl-v1-uni-weth-0.3pct/2026-10-09` |

## Verification performed

Mainnet forks (anvil, `ethereum-rpc.publicnode.com`), 2026-10-09.

**Mechanics, exact `01-schedule-raw.json` bytes from the DAO Safe** (block 26154963): each
`scheduleBatch` OK, gas **73,610**, 3 × `CallScheduled` + 1 × `CallSalt`; after the 48h warp,
`executeBatch` of 020 before 019 is `Done` reverts `TimelockUnexecutedPredecessor` `0x90a9a618`; a
role-less EOA calling `forceRemoveStrategy` directly reverts `RegistryClientMissingRole(ADMIN_ROLE)`
`0x4d616cff`.

**Effects, the whole migration** (block 26154937; fork-only `updateDelay(0)` so no warp stales the
feeds — `delay` is not part of the operation id, so these are the same operations):

| Step | Result |
|---|---|
| 020 before 019 | revert `0x90a9a618` |
| 019 `executeBatch` | OK, gas 823,913 — `strategyCount()` 4 → 8 |
| 020 before 018 (Controller still 1.0.0) | revert `RegistryClientMissingRole(KEEPER_ROLE)` `0x4d616cff` |
| 018's inner call applied (it is scheduled on mainnet, not yet `Ready`) | Controller `version() == "1.1.0"` |

| 020 batch | Gas | NAV before | ETH to Controller | vs NAV | Written off | After |
|---|---|---|---|---|---|---|
| USDC/WETH 0.3% | 3,425,837 | 1.293608 | 1.295557 | +0.15% | 2,307,201,237,520 wei | unregistered, `isCaller == false`, op `Done` |
| USDC/WETH 0.01% | 3,397,718 | 0.486187 | 0.486811 | +0.13% | 742,200,963,586 wei | same |
| WETH/USDT 0.3% | 2,945,781 | 1.456200 | 1.458262 | +0.14% | 2,127,289,800,635 wei | same |
| UNI/WETH 0.3% | 1,651,594 | 0.116228 | 0.116013 | −0.19% | 0 | same |

After all four: `strategies()` = the four v2 addresses; Controller 3.406643 ETH; `totalNAVInETH`
3.406643 ETH (from 3.402225 before). The three original v1 strategies delivered slightly **more** than
their reported NAV. `navInETH()` prices inventory at the TWAP, while the unwind executes at spot and
likely also realises LP fees sitting uncollected in the positions; this record does not separate the two. The fee-settlement step minted **7.1755 EVE** (≈ 0.0035 ETH at
`eveBasePriceInETH` 0.0004936) to the DAO treasury — the 15% performance fee
([014](../014-strategy-manager-performance-fee-bps/)) on LP fees not yet charged, which any harvest
would mint anyway. Re-executing a batch reverts `0x5ead8eb5`.

Then, as the keeper: `depositToStrategies(3.3566 ETH)` into the v2 set (gas 3,668,221) →
`totalNAVInETH` **3.401513 ETH** — −0.000712 ETH (−0.021%) end to end vs before the migration.
`withdrawFromStrategies(0.05 ETH)` and `checkAndRebalanceStrategies()` on the v2 set both succeeded.

## Risks

- **Liveness of a batch.** Each batch is all-or-nothing. If a sweep reverts at execution time (e.g.
  the pool price dislocates past the strategy's `maxTickDeviation` or swap slippage bound), the batch
  reverts and stays `Ready`; retry when the market settles. If one ever became permanently
  unexecutable, cancel it and re-propose that strategy as `[withdrawFromStrategy, forceRemoveStrategy]`.
- **`forceRemoveStrategy` skips the residue check.** Bounded by atomicity (it runs only after two full
  sweeps succeeded); the dropped amount is in each `StrategyForceRemoved` event. ≈ 5.2 × 10¹² wei on
  the fork.
- **Idle ETH until the keeper re-deposits.** ≈ 3.36 ETH sits on the Controller between 020 and the
  next `DepositExcess` tick: counted in NAV, available for exit settlement, earning nothing. The
  single re-deposit took 3.67M gas on the fork; the keeper may split it across ticks.
- **Performance fee settlement** mints EVE (≈ 7.2 EVE on the fork) when the batches run — expected,
  not a cost of the migration.
- **v1 deposits in the 019 → 020 window.** v1 deposit weights stay non-zero until removal, so the
  keeper may still deposit into v1 meanwhile; the batch drains whatever is there. Execute 020 right
  after 019.
- Removal is reversible only by re-adding the v1 address (`addStrategy`, 48h) — not a goal.

## Cancelling

Before execution, either the DAO Safe or the Security Safe may `cancel(opId)` any of the four
operations independently. If 019 is cancelled, all four become permanently unexecutable. After
execution, there is nothing to undo short of re-registering a v1 address via a new proposal.
