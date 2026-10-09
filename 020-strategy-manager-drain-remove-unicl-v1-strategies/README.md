# 020 — Drain and remove the four UniCL v1 strategies

**Status:** 📝 Draft — not yet submitted.
**Operation ids** (four independent `scheduleBatch` operations, each `predecessor` = 019):

| v1 strategy | Operation id |
|---|---|
| USDC/WETH 0.3% `0x5E12FFD5e0F69d37B581D06b68ad6c53609cac38` | `0x0867987d3b1031f96c2b1fe55e7c6ee16c98052725f8795a17afb28e24e8e06e` |
| USDC/WETH 0.01% `0x59C476c1817b23791d96F35b9f468E6d9A242252` | `0x15bd85e34ae24c4027895dac31be3b0acad8c9741efb1f8ff595cafe13eb2a14` |
| WETH/USDT 0.3% `0x3Fb6B9174427CA4FF38728398F4F4CB526F66501` | `0x4cb3334f7677057a07bf052fdbf7415816f1458f2af4d4a7ed26e979595a69ae` |
| UNI/WETH 0.3% `0x956fE55DB50527E75c1fcA679adE509AD89cAFDd` | `0xfb251769634d731c258f9a28038d21a373753303b030e2cb4e9319857548a11b` |

All four recomputed locally and reproduced by the mainnet timelock's own `hashOperationBatch`;
`getOperationState == 0` (Unset).

## What it changes

For each v1 strategy, one atomic timelock batch of four calls:

1. `Controller.withdrawFromStrategy(old, type(uint256).max)` — pulls the strategy's whole NAV to
   the Controller. StrategyManager caps the amount at `strategy.maxWithdrawal()` (= `navInETH()` while
   unpaused), so the call drains whatever the strategy holds **at execution time**; no amount goes
   stale over the 48h delay. It settles the strategy's pending performance fee first.
2. The same call again — sweeps any residue the v1 code leaves after a full withdrawal (below).
   When the first sweep already emptied the strategy it is a no-op: `min(max, 0) = 0` returns
   without calling the strategy.
3. `old.investIdleETH()` — the **pause guard**. It is `ADMIN_ROLE` + `whenNotPaused`, and after the
   drain the strategy holds no idle native ETH, so it returns 0 and does nothing. If the strategy is
   paused, it reverts `EnforcedPause` (`0xd93c0665`) and the whole batch with it.
4. `StrategyManager.forceRemoveStrategy(old)` — deregisters the strategy and revokes its
   `CONVERTER_CALLER_ROLE`. Emits `StrategyForceRemoved(old, reportedNAV, navReverted)`, which records
   exactly how much dust, if any, was written off.

Funds move strategy → Controller only; they stay inside the protocol and in NAV. With the v2 set
from [019](../019-strategy-manager-register-unicl-v2-strategies/) registered, the keeper's normal
`DepositExcess` path then deposits the Controller's ETH above `controllerReserveETH` (0.05 ETH,
[016](../016-strategy-keeper-exit-settlement-funding/)) into the v2 strategies by weight. After all
four batches, `strategies()` is exactly the v2 set.

## Why

Requested by **Vadyusha** (2026-10-09): the second half of the v1 → v2 migration; see 019 for what
v2 changes. Measured reasons to drain rather than leave v1 running beside v2:

- **The v1 withdraw path can leave value behind on a full unwind.** Whether it does depends on
  market state at the moment of withdrawal (see the residue table below) — consistent with the v1
  swap-sizing bug v2 fixes (`d714231`: v1 sizes swaps on the strategy pool, not the route it actually
  swaps through). Keeping v1 registered means large exits keep paying it.
- **Cost is small.** The whole migration — drain v1, re-deposit into v2 — moved `totalNAVInETH` by
  **−0.021% to −0.048%** across two fork runs an hour apart.

The alternative — zero the v1 deposit weights and let exits drain them over time — avoids the swap
cost now but keeps the v1 unwind behaviour for every withdrawal, runs eight strategies for an
open-ended period, and still needs a removal proposal at the end.

## Why `forceRemoveStrategy` instead of raising `MAX_NAV_RESIDUE`

`removeStrategy` requires `navInETH() ≤ MAX_NAV_RESIDUE` = **10 wei**. What a v1 full withdrawal
leaves behind is not stable — the same sweep, an hour apart:

| v1 strategy | Fork block 26154906: after 1 sweep | after 2 sweeps | Fork block 26155030: after 1 sweep |
|---|---|---|---|
| USDC/WETH 0.3% | 1,127,535,313,261,125 wei (0.087%) | 687,179,459,330 wei | 0 |
| USDC/WETH 0.01% | 367,746,021,708,073 wei (0.076%) | 223,932,453,917 wei | 0 |
| WETH/USDT 0.3% | 1,372,956,351,431,870 wei (0.094%) | 827,595,663,203 wei | 0 |
| UNI/WETH 0.3% | 0 | 0 | 0 |

So a `[withdraw, removeStrategy]` batch would execute or revert depending on prices at execution time.

Raising the bound was considered and rejected:

- **It is a `constant`.** Changing it means a new StrategyManager implementation, review, deployment
  and a separate 48h UUPS upgrade that 020 would also have to wait for — the contract that holds all
  strategy accounting, upgraded to accommodate a one-off legacy migration. v2 does not need it: v2
  strategies drain to exactly 0.
- **No absolute bound fits.** The residue scales with NAV. At today's ≈1.3 ETH per strategy it
  ranged 0 – 1.4 × 10¹⁵ wei after one sweep; the WETH/USDT cap is 4500 ETH, where the same 0.09% is
  ≈4 ETH. A bound loose enough for v1 at scale stops guarding anything; a tight one makes the batch
  a coin flip.
- **`forceRemoveStrategy` exists for exactly this case**, records the dropped NAV in its event, and
  the guard below restores the protection that matters.

**What `removeStrategy`'s check was really protecting** is "never deregister a strategy that still
holds funds". Within this batch that can only happen if the withdrawals silently move nothing, and
the only way for that while `navInETH() > 0` is a **paused** strategy (`maxWithdrawal()` returns 0
when paused; a strategy pauses only via `pause()`, by ADMIN or the Security Safe). Step 3 makes the
batch revert in exactly that case. Otherwise each withdrawal either completes for the full
`navInETH()` (net of swap costs) or reverts the batch. With both sweeps done, `forceRemoveStrategy` writes off at most the second-sweep residue
— ≤ 8.3 × 10¹¹ wei per strategy in every run so far, 0 in the latest.

## Dependency on 019 and 018

- **019 is the `predecessor`** of all four operations (`0xf558cda6…fc9ed6a6`), so `executeBatch` reverts
  `TimelockUnexecutedPredecessor` (`0x90a9a618`) until the v2 set is registered. Without it, 020 could
  leave the protocol with no strategies at all.
- **018 is required but not encoded.** `Controller.withdrawFromStrategy` is `KEEPER_ROLE`-only until
  [018](../018-controller-upgrade-v1.1.0/) (Controller v1.1.0) executes; before that every 020 batch
  reverts `RegistryClientMissingRole(KEEPER_ROLE)` (`0x4d616cff`) and simply stays `Ready` — a failed
  `executeBatch` does not consume the operation. 018 is scheduled (ready 2026-10-11 12:07:59 UTC),
  before 020 can be. Only one predecessor fits; 019 was chosen because skipping it is the harmful
  case, while skipping 018 only reverts.
- The four 020 operations are independent of each other and may execute in any order.

Alternative: `predecessor = 0x00…00` on each, sequencing 018 → 019 → 020 by hand — the operation ids
change.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | 4 × `timelock.scheduleBatch(...)` (4 calls each), `delay = 172800` — wrapped by `MultiSendCallOnly` |
| 2 | `02-execute.json` | anyone | 4 × `timelock.executeBatch(...)` after 018 and 019 are done |

`01-schedule-raw.json` is the same four transactions as raw calldata (selector `0x8f2a0bb0`); both
encode to identical bytes. **Prefer the raw file** for batches (see 019).

### Parameters

| Field | Value |
|---|---|
| `targets` | `[Controller 0x9097868dcbda5a630729015DE1ddeCd66d74052c, Controller, <old strategy>, StrategyManager 0x94916ab93C669E7c734f844dB019Ce9449a3b5C9]` |
| `values` | `[0, 0, 0, 0]` |
| `payloads` | `[withdrawFromStrategy(old, 2²⁵⁶−1), withdrawFromStrategy(old, 2²⁵⁶−1), investIdleETH(), forceRemoveStrategy(old)]` — selectors `0xb53d0958`, `0xb53d0958`, `0xf06d084b`, `0x428ea195` |
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

**Mechanics, exact `01-schedule-raw.json` bytes from the DAO Safe** (block 26155075): each
`scheduleBatch` OK, gas **79,228**, 4 × `CallScheduled` + 1 × `CallSalt`; after the 48h warp,
`executeBatch` of 020 before 019 is `Done` reverts `TimelockUnexecutedPredecessor` `0x90a9a618`; a
role-less EOA calling `forceRemoveStrategy` directly reverts `RegistryClientMissingRole(ADMIN_ROLE)`
`0x4d616cff`. All four v1 strategies expose `investIdleETH()` (`0xf06d084b`); an `eth_call` of it as
the timelock returns 0 on mainnet today.

**Effects, the whole migration** (block 26155078; fork-only `updateDelay(0)` so no warp stales the
feeds — `delay` is not part of the operation id, so these are the same operations):

| Step | Result |
|---|---|
| 020 before 019 | revert `0x90a9a618` |
| 019 `executeBatch` | OK, gas 823,913 — `strategyCount()` 4 → 8 |
| 020 before 018 (Controller still 1.0.0) | revert `RegistryClientMissingRole(KEEPER_ROLE)` `0x4d616cff` |
| 018's inner call applied (scheduled on mainnet, not yet `Ready`) | Controller `version() == "1.1.0"` |

| 020 batch | Gas | NAV before (ETH) | ETH to Controller | vs NAV | Written off | After |
|---|---|---|---|---|---|---|
| USDC/WETH 0.3% | 2,097,271 | 1.295418 | 1.294680 | −0.057% | 0 | unregistered, `isCaller == false`, op `Done` |
| USDC/WETH 0.01% | 1,956,661 | 0.486760 | 0.486536 | −0.046% | 0 | same |
| WETH/USDT 0.3% | 1,770,720 | 1.458232 | 1.457293 | −0.064% | 0 | same |
| UNI/WETH 0.3% | 1,679,249 | 0.116050 | 0.115985 | −0.057% | 0 | same |

After all four: `strategies()` = the four v2 addresses; `totalNAVInETH` 3.404493 ETH (from 3.406463
before), all on the Controller. Fee settlement minted **6.9602 EVE** (≈ 0.0034 ETH at
`eveBasePriceInETH` 0.0004936) to the DAO treasury — the 15% performance fee
([014](../014-strategy-manager-performance-fee-bps/)) on LP fees not yet charged, which any harvest
would mint anyway. Re-executing a batch reverts `0x5ead8eb5`. Then, as the keeper,
`depositToStrategies(3.3545 ETH)` into the v2 set (gas 3,779,313) → `totalNAVInETH` **3.404828 ETH —
−0.001635 ETH (−0.048%)** end to end; `withdrawFromStrategies(0.05 ETH)` (gas 4,235,940) and
`checkAndRebalanceStrategies()` (gas 523,805) on the v2 set both succeeded.

An earlier run at block 26154937, with the batch then lacking step 3, found residues after two sweeps
of 2.3 × 10¹², 7.4 × 10¹¹, 2.1 × 10¹² and 0 wei written off, the three original v1 strategies
delivering +0.13–0.15% over reported NAV (TWAP-priced `navInETH()` vs spot unwind), and −0.021% end
to end.

**The guard** (block 26155018):

| Case | Result |
|---|---|
| Security Safe `pause()`s USDC/WETH 0.3% v1 (it still reports `navInETH()` 1.295682 ETH, `maxWithdrawal()` 0), then `executeBatch` | revert `EnforcedPause` `0xd93c0665`; strategy still registered; operation stays `2 (Ready)` |
| Same state with the batch lacking step 3 | would have force-removed it, dropping 1.295682 ETH from NAV |
| Someone donates 1 wei of native ETH to USDC/WETH 0.01% v1 first | batch OK, residue 0 — the sweeps spend idle ETH first, so step 3 still finds none |

## Risks

- **Liveness of a batch.** Each batch is all-or-nothing. If a sweep reverts at execution time (e.g.
  price dislocation beyond the strategy's tick-deviation or swap-slippage bound), or the strategy is
  paused, the batch reverts and stays `Ready`; retry once the cause clears. If one ever became
  permanently unexecutable, cancel it and re-propose that strategy.
- **`forceRemoveStrategy` skips the residue check.** Bounded by steps 1–3 as above; the dropped
  amount, if any, is in each `StrategyForceRemoved` event.
- **Idle ETH until the keeper re-deposits.** ≈ 3.35 ETH sits on the Controller between 020 and the
  next `DepositExcess` tick: counted in NAV, available for exit settlement, earning nothing. The
  single re-deposit took 3.78M gas on the fork; the keeper may split it across ticks.
- **Performance fee settlement** mints EVE (≈ 7 EVE on the fork) when the batches run — expected,
  not a cost of the migration.
- **v1 deposits in the 019 → 020 window.** v1 deposit weights stay non-zero until removal, so the
  keeper may still deposit into v1 meanwhile; the batch drains whatever is there. Execute 020 right
  after 019.
- Removal is reversible only by re-adding the v1 address (`addStrategy`, 48h) — not a goal.

## Cancelling

Before execution, either the DAO Safe or the Security Safe may `cancel(opId)` any of the four
operations independently. If 019 is cancelled, all four become permanently unexecutable. After
execution, there is nothing to undo short of re-registering a v1 address via a new proposal.
