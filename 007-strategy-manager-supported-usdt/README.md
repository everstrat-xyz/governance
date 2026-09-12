# 007 — StrategyManager: whitelist USDT as a supported ERC-20

**Status:** ⏳ **Scheduled on mainnet** (tx `0x83ddbe36f5af0e3365a4ca25459f9a78fb6bd1afaadf7de03b1d307036283c75`, block 25956365, 2026-09-11 19:50:59 UTC). Timelock state Waiting. Executable from **2026-09-13 19:50:59 UTC** — **and** only once [006](../006-oracle-usdt-feed/)'s USDT feed operation is executed (see *Dependency on 006*).
**Operation id:** `0xd98323ea4ce022263fda34fe855afa99119621065da39310f13701106a7f9274`
(recomputable via `hashOperation` with the predecessor and salt below — recomputed and verified on mainnet).

## What it changes

`StrategyManager.addSupportedERC20(0xdAC17F958D2ee523a2206206994597C13D831ec7)` — adds USDT
(Tether) to the StrategyManager's supported-ERC-20 set (`EnumerableSet`, checked by
`isSupportedERC20` / listed by `supportedERC20()`).

Effect: once USDT is in the set, any **non-zero** USDT balance held by the StrategyManager
is priced into NAV via the Oracle (`_supportedERC20sNAVInETH` → `Oracle.convert(USDT, ETH, …)`).
A zero balance is skipped entirely, so adding the token now — while the balance is `0` —
changes no accounting.

**What it does NOT do:** grants no role, registers no feed, touches no strategy, no AMM or
registry parameter. It does not move or swap any funds.

## Why

Symmetry with **004 (USDC)**: the protocol's strategy pairs ETH with stablecoins. On
`IStrategy.emergencyExit()` the strategy unwinds and forwards **both** legs to the
StrategyManager — so USDT can land on the StrategyManager during an emergency while the
protocol is paused. `addSupportedERC20` is deliberately **not** `whenNotPaused` for exactly
this reason: the whitelist must be fixable mid-incident. Whitelisting USDT ahead of time
means a stranded USDT balance is counted in NAV immediately rather than invisible until
someone reacts.

**Timing:** proposed while the protocol is unbootstrapped and no strategy is live, so the
StrategyManager's USDT balance is `0` and this has no effect on NAV or any position until a
real strategy emergency-exits.

## Dependency on 006

`addSupportedERC20` checks `Oracle.isTokenSupported(USDT)`, which only becomes true once
**006's USDT feed operation** (`0x3aa9f59be4381618b4c684ca8155b837fa1718db26ef96c021ba4d775fe21a9a`)
is executed. This proposal wires that in explicitly: its timelock **predecessor** is set to
that operation id, so `execute` reverts `TimelockUnexecutedPredecessor` until 006's USDT
feed is done. Scheduling can happen any time (predecessor is only enforced at execute).

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` with `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` after the delay **and** after 006 |

`01-schedule-raw.json` is the same transaction as raw calldata (selector `0x01d5062a`).

### Parameters

| Field | Value |
|---|---|
| `to` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (admin timelock) |
| `target` | `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` (StrategyManager) |
| `value` | `0` |
| `data` | `0xd73acee5000000000000000000000000dac17f958d2ee523a2206206994597c13d831ec7` |
| `predecessor` | `0x3aa9f59be4381618b4c684ca8155b837fa1718db26ef96c021ba4d775fe21a9a` (006 USDT feed) |
| `salt` | `0x5e625924673f6ec94b6bbb021a8ea4199b020fcf09bfc370183373d3d5a85cf4` |
| `delay` | `172800` (48h) |

`data` decodes to `addSupportedERC20(address)` (selector `0xd73acee5`) with
`_token = 0xdAC17F958D2ee523a2206206994597C13D831ec7`.

The salt derives from `keccak256("everstrat/strategy-manager/supported-erc20/usdt/2026-09-11")`.

## Verification performed (forked mainnet, 2026-09-11)

Fork at mainnet head, `StrategyManager.isSupportedERC20(USDT) == false`,
`Oracle.isTokenSupported(USDT) == false`, 006's USDT-feed op already **scheduled on-chain**
(`getOperationState == 1 Waiting`, ready-at `1789300487`). Full `test/SimExec007.t.sol` run:

1. DAO Safe (`0x1780C78e…`) → `timelock.schedule(SM, 0, addSupportedERC20(USDT), pred, salt, 172800)`
   — **success**; `CallScheduled` + `CallSalt` emitted for op
   `0xd98323ea4ce022263fda34fe855afa99119621065da39310f13701106a7f9274`.
2. `execute` after the 48h delay but **before 006** — **reverts**
   `TimelockUnexecutedPredecessor(0x3aa9f59b…)` (`0x90a9a618`). Confirms the coupling.
3. Warp past 006's ready-at → execute 006's feed op (already scheduled on-chain) —
   **success**; `Oracle.isTokenSupported(USDT) → true`.
4. `execute` 007 from an unrelated EOA (`0xDEAD`) — **success**, 279,815 gas.
5. Post-state: `StrategyManager.isSupportedERC20(USDT) == true`.

Recomputing `hashOperation` from the fields as rendered reproduces the operation id above.

## On-chain schedule (mainnet)

Scheduled 2026-09-11 19:50:59 UTC in tx
`0x83ddbe36f5af0e3365a4ca25459f9a78fb6bd1afaadf7de03b1d307036283c75` (block 25956365),
DAO Safe → timelock `schedule`. Confirmed against mainnet:

- The `schedule` calldata in the tx matches `01-schedule-raw.json` byte-for-byte (target
  StrategyManager, `data` `0xd73acee5…31ec7`, predecessor `0x3aa9f59b…5fe21a9a`, salt
  `0x5e625924…5a85cf4`, delay `172800`).
- `getOperationState(0xd98323ea…6a7f9274)` → `1` (Waiting); `isOperationPending` → `true`.
- `getTimestamp` → `1789329059` = **2026-09-13 19:50:59 UTC** (ready-at).
- Predecessor `0x3aa9f59b…5fe21a9a` (006's USDT feed) is still `1` (Waiting) — `execute`
  will revert `TimelockUnexecutedPredecessor` until 006's USDT feed is executed.

## Cancelling

Either the DAO Safe or the Security Safe may call
`cancel(0xd98323ea4ce022263fda34fe855afa99119621065da39310f13701106a7f9274)` on the timelock
at any point before execution.
