# 021 — Drain and remove the UniCL UNI/WETH v2 strategy

**Status:** 📝 **Draft — PR opened, not yet queued in the DAO Safe.** Calldata built and simulated
2026-10-20, re-verified against the live chain 2026-10-21 (block 26,240,182). Nothing is scheduled:
the next DAO Safe nonce is **31** (the queue holds only the dead 015 at nonce 24 and an ancient
nonce-5 entry), so this batch becomes the nonce-31 Safe transaction.
**Operation id:** `0xa8bf3f7281633b88cfe2f5c14e59b149afff1820a0b57a657b7935c42a7f9656` — recomputed
locally and reproduced by the mainnet Timelock's own `hashOperationBatch`; `getOperationState == 0`
(Unset).

## What it changes

One atomic 48h timelock batch of four calls, all against the registered UniCL **UNI/WETH 0.3% v2**
strategy `0xfc888a499c4172e393a258fbc6cad1475a7e4129` (S4, weight 10/10):

1. `Controller.withdrawFromStrategy(S4, type(uint256).max)` — pulls the strategy's whole NAV to the
   Controller. StrategyManager caps the amount at `strategy.maxWithdrawal()` (= `navInETH()` while the
   strategy is unpaused), so the call drains whatever the strategy holds **at execution time** — no
   amount goes stale over the 48h delay. It settles the strategy's pending performance fee first.
2. The same call again — sweeps any residue a full unwind leaves behind (same reasoning as 020).
   When the first sweep already emptied the strategy it is a no-op: `min(max, 0) = 0` returns without
   calling the strategy.
3. `UniCLStrat(S4).pause()` — closes the strategy to deposits and rebalances while it is being deleted.
   This is the fix for the window found in the 10-19 execution: a keeper `DepositExcess` that landed
   *before* a `forceRemoveStrategy` in the same block deposited 0.378161667 ETH into a strategy that
   was removed 112 transactions later.
4. `StrategyManager.forceRemoveStrategy(S4)` — deregisters the strategy and revokes its
   `CONVERTER_CALLER_ROLE`. Emits `StrategyForceRemoved(S4, reportedNAV, navReverted)`, which records
   exactly how much dust, if any, was written off.

Funds move strategy → Controller only; they stay inside the protocol and in NAV. The keeper's normal
`DepositExcess` path then re-deposits the Controller's ETH above `controllerReserveETH` (0.05 ETH,
[016](../016-strategy-keeper-exit-settlement-funding/)) into the remaining three v2 strategies by
weight. **After this batch the registered set is three strategies whose weights are 40/15/45 — a sum of
exactly 100** (the current four sum to 110 nominally; the StrategyManager normalises by the running
sum, so 110 is bookkeeping, not a defect).

Why `forceRemoveStrategy` rather than `removeStrategy`: `removeStrategy` requires
`navInETH() ≤ MAX_NAV_RESIDUE` = 10 wei, and whether a full unwind meets that is decided by the market
at the moment of execution (see [020](../020-strategy-manager-drain-remove-unicl-v1-strategies/README.md)
§ "Why `forceRemoveStrategy`"). `forceRemoveStrategy` makes the removal deterministic and records any
dust in the event.

## Why

Requested by **Arseny** (2026-10-20, Telegram thread 5599): *"what's 21? Removing of UNI strategy?
then yes"*. The strategic rationale beyond the request itself is not recorded in the project's memory —
what follows is the measured evidence, not an argument invented after the fact.

Measured, live, block 26,240,178 (2026-10-21 09:49 UTC), `performanceFeeBps` 1500, ETH $2,461.65:

- **It is the only strategy whose realized ETH-terms return was negative** in the v1 book. Window
  2026-09-17 13:11:35 → 2026-10-19 11:49:35 (v1 strategies, 28.0–38.8 d):

  | Strategy | Deposited | Withdrawn | Realized | Return |
  |---|---|---|---|---|
  | S1 WETH/USDC 0.3% | 1.774476 ETH | 1.851563 ETH | +0.077087 ETH | **+4.344 %** |
  | S2 WETH/USDC 0.01% | 0.665428 ETH | 0.693550 ETH | +0.028122 ETH | **+4.226 %** |
  | S3 WETH/USDT 0.3% | 1.996285 ETH | 2.082981 ETH | +0.086696 ETH | **+4.343 %** |
  | S4 UNI/WETH 0.3% | 0.623436 ETH | 0.615957 ETH | −0.007478 ETH | **−1.200 %** |

- **It is the only strategy with a non-stable paired leg** (UNI), and the only one whose existence
  depends on [013](../013-add-uni-supported-token/) (UNI/USD oracle feed + `addSupportedERC20(UNI)`).
  No other strategy in the set carries a governance-token leg.
- The v2 copy is currently the book's smallest position (0.444343 ETH of 4.889295 ETH = **9.1 %**) and
  its realized window is short (12.3 d, +0.247 %, not annualisable).
- Pool-level forward baseline **today** (GeckoTerminal 24h, `feeAPR = vol24h × fee ÷ TVL`, full-range
  pool baseline — **not** this position's realized return): UNI/WETH 0.3 % **30.66 %** (TVL $14.61M,
  vol24h $4.09M) vs WETH/USDT 38.49 %, WETH/USDC 0.3 % 21.92 %, WETH/USDC 0.01 % 21.71 %. On this
  measure the pool is mid-pack today, and was the weakest of the four in the 2026-10-20 snapshot
  (12.80 %) — it moves with the pool's volume, so it is not by itself an argument either way.

Alternative considered: **zero the deposit weight and let exits drain it over time.** Rejected —
it leaves a fourth `UniCLStrat` on the books indefinitely, keeps the UNI-leg unwind path live for every
future withdrawal, and still needs a removal proposal at the end, exactly as with the v1 set in 020.

## Transactions

One entry in the DAO Safe — `Timelock.scheduleBatch`, operation **Call** (not MultiSend):

| # | Target | Value | Call | Selector |
|---|---|---|---|---|
| 0 | `0x9097868dcbda5a630729015DE1ddeCd66d74052c` (Controller) | 0 | `withdrawFromStrategy(0xfc88…4129, type(uint256).max)` | `0xb53d0958` |
| 1 | `0x9097868dcbda5a630729015DE1ddeCd66d74052c` (Controller) | 0 | `withdrawFromStrategy(0xfc88…4129, type(uint256).max)` | `0xb53d0958` |
| 2 | `0xfc888a499c4172e393a258fbc6cad1475a7e4129` (UniCLStrat S4) | 0 | `pause()` | `0x8456cb59` |
| 3 | `0x94916ab93C669E7c734F844DB019CE9449a3b5C9` (StrategyManager) | 0 | `forceRemoveStrategy(0xfc88…4129)` | `0x428ea195` |

## Parameters

| Name | Value |
|---|---|
| `predecessor` | `0x00…00` — no on-chain dependency (018 and 019 are already `Done`; see below) |
| `salt` | `0xc9735ac119706fa807fd289be4ca15f7755821e569c25c255d2c9a2cb83c21d8` = `keccak256("everstrat/strategy/drain-remove-uni-weth/2026-10-20")` |
| `delay` | `172800` (48 h) |
| target strategy | `0xfc888a499c4172e393a258fbc6cad1475a7e4129` — UniCLStrat **v2**, UNI/WETH 0.3 %, weight 10/10, registered by [019](../019-strategy-manager-register-unicl-v2-strategies/) |

`01-schedule-raw.json` carries the same batch as raw calldata (`scheduleBatch` = `0x8f2a0bb0`,
2186 bytes) for when the Safe UI cannot resolve the timelock ABI; `02-execute.json` carries the
`executeBatch` calldata (`0xe38335e5`) for the T+48h step, which is permissionless
(`EXECUTOR_ROLE = address(0)`).

## Verification performed

Live mainnet reads, 2026-10-21 09:48–09:50 UTC, block **26,240,182** (no fork needed — the
authorization path is resolved by calling as the Timelock):

1. **Operation id reproduced by the chain's own view.** `Timelock.hashOperationBatch(targets, values,
   payloads, salt, predecessor)` returns `0xa8bf3f72…42a7f9656`, identical to the locally computed id
   from `01-schedule-raw.json`. `getOperationState(id)` = **0 (Unset)** — nothing scheduled yet, as
   expected for a Safe transaction that has not executed.
2. **Every inner call simulated with `from` = the Timelock** (`0xF0911198…91774`) — so
   `onlyAuthRole(ADMIN_ROLE)` resolves for real:

   | Call | Result |
   |---|---|
   | `Controller.withdrawFromStrategy(S4, max)` | **OK** — returns `0.444342931414988172` ETH (the strategy's whole NAV) |
   | `UniCLStrat(S4).pause()` | **OK** — empty return |
   | `StrategyManager.forceRemoveStrategy(S4)` | **OK** — empty return |

   Control: the same calls sent from an EOA instead revert `RegistryClientCallerHasNoneOfRoles(bytes32,bytes32)`
   (`0x958c6bdf`) — the simulations above are meaningful, not a permissive fallback.
3. **Prerequisites are executed, not merely scheduled.** `Controller.version() == 1.1.0` (018 Done
   2026-10-12 12:13:23 UTC, op `0x8cc3710f…c98a`) and the four v2 strategies are registered with the v1
   set drained (019 + 020 Done 2026-10-19 11:49:35–11:50:35 UTC). Without 018, `withdrawFromStrategy`
   reverts `RegistryClientMissingRole(keccak256("KEEPER_ROLE"))` and this batch would be dead on arrival.
4. **The strategy's live state supports a clean drain.** `paused()` = false ·
   `navInETH()` = `maxWithdrawal()` = **0.444342931414988172 ETH** · `totalDeposited()` = 0.443248706688
   ETH · `maxDeposit()` = 249.4 ETH · registry `hasRole(ADMIN_ROLE, Timelock)` = true.
5. **No ERC-20 residue that a `forceRemove` would strand.** `balanceOf(UNI)` = **0** on the strategy, on
   the StrategyManager, on the Controller and on the Converter; the strategy's own ETH balance = 0. The
   Controller holds exactly its 0.05 ETH reserve. (A UNI residue of 0.004997 was observed on 10-20 and
   is no longer there.)
6. **Nothing else is pending.** DAO Safe nonce 31, threshold 3, five owners; the Tx Service queue holds
   only the dead 015 (nonce 24, 0 confirmations) and an ancient nonce-5 entry.

## Risks

- **Unwinding realizes the position's IL and stops its fee accrual.** This is a deliberate exit, not a
  free operation: the 0.4443 ETH comes back as ETH (the swap leg pays `swapSlippageBps` = 100 and the
  pool's fee), and the strategy stops earning.
- **Liveness — the batch can revert and stay `Ready`.** The two withdrawals revert `ConverterSwapFailed`
  while the UNI/WETH pool is dislocated from its 30-min TWAP beyond the strategy's calm bound. Nothing
  is lost: an op that reverts stays `Ready`, and `executeBatch` can be re-sent while the pool is calm.
  Unlike 020 (four independent batches), this is a **single** batch, so all four calls land together or
  none of them do.
- **Re-check immediately before executing.** If a rebalance between now and execution leaves a UNI
  residual on the strategy, `forceRemoveStrategy` still deletes it and the token would sit in a
  deregistered contract (the known off-book ERC-20 gap). Confirm `UniCLStrat.balanceOf(UNI) == 0` and
  `navInETH()` after the drain, and that `paused()` is false *before* the batch runs (a paused strategy
  makes `withdrawFromStrategy` silently no-op while `forceRemoveStrategy` still removes it).
- **`pause()` is inside the batch, after the withdrawals** — so the deposit-window race is closed only
  for the interval between the drain and the removal, which is the interval that matters. The residual
  risk is a `DepositExcess` in the same block *before* the drain; the keeper's weight set still
  includes S4 until the removal confirms.

## Dependency on 018 / 019

Both **Executed**: 018 (Controller v1.0.0 → v1.1.0) on 2026-10-12, 019 (register the four v2
strategies) on 2026-10-19. `predecessor` is therefore `0x00…00` — there is no pending op to gate on.
The batch targets the **v2** strategy `0xfc88…4129`; its v1 predecessor `0x956fE55DB50527E75c1fcA679adE509AD89cAFDd`
was drained and force-removed by [020](../020-strategy-manager-drain-remove-unicl-v1-strategies/).

## Cancelling

`Timelock.cancel(opId)` is callable by the DAO Safe or the Security Safe before execution. Reverting
this proposal before the Safe transaction executes needs no cancellation at all: the op does not exist
until then (`state == Unset`).

## Provenance

Built 2026-10-20 from live mainnet reads, the `contracts` sources (`UniCLStrat.withdraw`,
`StrategyManager.forceRemoveStrategy`, `Controller.withdrawFromStrategy`), and the 019/020 records;
re-verified end-to-end against the live chain 2026-10-21. The requester's approval is on record; the
strategic rationale is marked above as not recorded rather than reconstructed.
