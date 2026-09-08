# 004 — StrategyManager: whitelist USDC as a supported ERC-20

**Status:** ✅ **Executed on mainnet** (tx `0x1a97bcc41c357bcabde9975a4f7d79a396d090ff76d3cb1717e9c4828d8ed431`, block 25933686, 2026-09-08 15:57:47 UTC) — via the DAO Safe after the 48h delay and after 003's USDC feed. `StrategyManager.isSupportedERC20(USDC) == true`, `supportedERC20() == [USDC]`. Scheduled 2026-09-06 00:09:11 UTC (tx `0x60bec29dc01dc658741ebe883697502da81e8046de1761abc3018ea6a2543f57`, block 25914620).
**Operation id:** `0x602fe6598633e3e8ab93387234aa64ffa49fae9edcf95215b2cfc968093871cb`
(`hashOperation(StrategyManager, 0, addSupportedERC20(USDC), predecessor, salt)`, with the
predecessor and salt below — recomputed and verified on mainnet).

## What it changes

`StrategyManager.addSupportedERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48)` — adds USDC
to the StrategyManager's supported-ERC-20 set (`EnumerableSet`, checked by
`isSupportedERC20` / listed by `supportedERC20()`).

Effect: once USDC is in the set, any **non-zero** USDC balance held by the StrategyManager
is priced into NAV via the Oracle (`_supportedERC20sNAVInETH` → `Oracle.convert(USDC, ETH, …)`).
A zero balance is skipped entirely, so adding the token now — while the balance is `0` —
changes no accounting.

**What it does NOT do:** grants no role, registers no feed, touches no strategy, no AMM or
registry parameter. It does not move or swap any funds; ERC-20 recovery back to native ETH
via the Converter is a separate, later change.

## Why

The strategy the protocol is preparing to run pairs ETH with USDC. On
`IStrategy.emergencyExit()` the strategy unwinds and forwards **both** legs to the
StrategyManager — so USDC can land on the StrategyManager during an emergency, while the
protocol is paused. `addSupportedERC20` is deliberately **not** `whenNotPaused` for exactly
this reason: the whitelist must be fixable mid-incident. Whitelisting USDC ahead of time
means a stranded USDC balance is counted in NAV immediately rather than being invisible
until someone reacts.

USDC must be **priceable by the Oracle** before it can be added — `addSupportedERC20`
reverts `StrategyManagerERC20NotPriceable` otherwise. That is the dependency on 003.

**Timing:** proposed while the protocol is unbootstrapped and no strategy is live, so the
StrategyManager's USDC balance is `0` and this has no effect on NAV or any position until
a real strategy emergency-exits.

## Dependency on 003

`addSupportedERC20` checks `Oracle.isTokenSupported(USDC)`, which only becomes true once
**003's USDC feed operation** (`0xd57312a1b34a92fa9799b8467c6e49733f334d0a83aefbfad9980f1a8b7ed5a1`)
is executed. This proposal wires that in explicitly: its timelock **predecessor** is set to
that operation id, so `execute` reverts `TimelockUnexecutedPredecessor` until 003's USDC
feed is done. Scheduling can happen any time (predecessor is only enforced at execute).

To decouple — schedule 004 without waiting on 003's exact operation id — set `predecessor`
to `0x00…00` and sequence the two executes manually. The operation id changes if you do.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` with `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` after the delay **and** after 003's USDC feed |

`01-schedule-raw.json` is the same transaction as raw calldata (selector `0x01d5062a`), for
when the Safe UI cannot resolve the timelock ABI.

### Parameters

| Field | Value |
|---|---|
| `to` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (admin timelock) |
| `target` | `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` (StrategyManager) |
| `value` | `0` |
| `data` | `0xd73acee5000000000000000000000000a0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` |
| `predecessor` | `0xd57312a1b34a92fa9799b8467c6e49733f334d0a83aefbfad9980f1a8b7ed5a1` (003 USDC feed op) |
| `salt` | `0xd35f034b2bc943f2f73bd90d03ca4da612af83712e2dedbf26571e73f16b1825` |
| `delay` | `172800` (48h — the enforced minimum) |

`data` decodes to `addSupportedERC20(address)` (selector `0xd73acee5`) with
`_token = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48`.

The salt derives from `keccak256("everstrat/strategy-manager/supported-erc20/usdc/2026-09-06")`
and restores the per-proposal salt convention that 003 dropped. Predecessor and salt must
be reused **byte-for-byte** at execute time.

## Verification performed (forked mainnet, before signing)

Fork at block 25914427, `StrategyManager.isSupportedERC20(USDC) == false`,
`Oracle.isTokenSupported(USDC) == false`, `Registry.hasRole(ADMIN_ROLE, timelock) == true`.

1. Direct `timelock → StrategyManager.addSupportedERC20(USDC)` with USDC not yet priceable —
   **reverts** `StrategyManagerERC20NotPriceable(USDC)` (`0x0eb217c5`). Confirms the 003 dependency.
2. DAO Safe → `schedule` with the exact calldata above — **success**, 56,591 gas.
   `CallScheduled` + `CallSalt` emitted for op `0x602fe659…093871cb`.
3. `isOperationPending` `true`, `isOperationReady` `false`, ready-at exactly `scheduled + 172800`.
4. `execute` before the delay — **reverts** `TimelockUnexpectedOperationState` (`0x5ead8eb5`, state 4 = Waiting).
5. Warp 48h + 1s. `execute` while 003's USDC feed op is still pending — **reverts**
   `TimelockUnexecutedPredecessor(0xd57312a1…)` (`0x90a9a618`).
6. Execute 003's USDC feed op from an unrelated EOA — success; `Oracle.isTokenSupported(USDC)` → `true`.
7. `execute` 004 **from an unrelated EOA** — success, 141,777 gas. `SupportedERC20Added(USDC)`
   emitted (`0xf7e6a5cd…994054`), confirming execution is permissionless.
8. Post-state: `isSupportedERC20(USDC) == true`, `supportedERC20() == [USDC]`.
9. Re-`execute` and re-`schedule` the same op — both revert `TimelockUnexpectedOperationState`;
   a direct re-add reverts `StrategyManagerERC20AlreadySupported` (`0x04a77d9d`).

Recomputing `hashOperation` from the fields as rendered in the Safe UI reproduces the
operation id above.

## On-chain schedule (mainnet)

Scheduled 2026-09-06 00:09:11 UTC in tx
`0x60bec29dc01dc658741ebe883697502da81e8046de1761abc3018ea6a2543f57` (block 25914620),
DAO Safe → timelock `schedule`. Confirmed against mainnet:

- The tx calldata to the timelock matches `01-schedule.json` byte-for-byte (target
  StrategyManager, `data` `0xd73acee5…3606eb48`, predecessor `0xd57312a1…8b7ed5a1`, salt
  `0xd35f034b…16b1825`, delay `172800`).
- The tx calldata matched `01-schedule.json` byte-for-byte.
- `getTimestamp` → `1788826151` = **2026-09-08 00:09:11 UTC** (ready-at).

## On-chain execution (mainnet)

Executed 2026-09-08 15:57:47 UTC in tx
`0x1a97bcc41c357bcabde9975a4f7d79a396d090ff76d3cb1717e9c4828d8ed431` (block 25933686),
DAO Safe → timelock `execute` (Safe nonce 10). Prerequisites were both met on-chain first:

- 003's USDC feed op `0xd57312a1…8b7ed5a1` executed 2026-09-07 12:02:11 UTC
  (tx `0xe324753e7cc9d004c5348fa91ecbd80017fce4d955f7f84a6bfa7e3dfd06942c`), so the
  predecessor check passed.
- 48h delay elapsed 2026-09-08 00:09:11 UTC.

Confirmed after execution:

- `getOperationState(0x602fe659…093871cb)` → `3` (Done).
- `CallExecuted` + `SupportedERC20Added(USDC)` (`0xf7e6a5cd…994054`) emitted.
- `StrategyManager.isSupportedERC20(USDC)` → `true`; `supportedERC20()` → `[USDC]`.
- `totalNAVInETH()` / `totalNAVInUSD()` → `0` (USDC balance is `0`, so no NAV effect).

## Risks

- **NAV freeze while stranded.** Once USDC is supported, a non-zero USDC balance on the
  StrategyManager with a stale/invalid USD feed makes `_supportedERC20sNAVInETH` revert and
  freezes NAV (fail-closed, by design). Mitigation exists: `removeSupportedERC20(USDC)` —
  `SECURITY_ROLE` can call it with no delay, `ADMIN_ROLE` via the 48h path — and it makes no
  external calls, so a bricked token cannot block its own removal.
- **Native-ETH pricing must be complete.** NAV pricing of a USDC balance goes
  `Oracle.convert(USDC, address(0), …)`, i.e. it also needs ETH priced. 003 registers WETH,
  not the zero address; confirm the Oracle's native-ETH path is configured before any USDC
  can actually reach the StrategyManager. Not required for `addSupportedERC20` itself
  (the add only checks set membership of the USDC feed).
- **Predecessor coupling.** If 003's USDC feed op is cancelled and re-scheduled under a
  different salt, its operation id changes and this proposal must be re-scheduled with the
  new predecessor.

## Cancelling

No longer possible — the operation is executed. Before execution, either the DAO Safe or
the Security Safe could `cancel(0x602fe6598633e3e8ab93387234aa64ffa49fae9edcf95215b2cfc968093871cb)`
on the timelock. To reverse the effect now, `removeSupportedERC20(USDC)` — `SECURITY_ROLE`
with no delay, or `ADMIN_ROLE` via the 48h path.
