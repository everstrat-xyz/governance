# 012 — Allow the Mimic smart account as executor caller on both keeper executors

**Status:** ✅ **Executed on mainnet** — 2026-09-17 12:02:59 / 12:04:23 UTC. Both executors now
report `isExecutorCaller(0x41153f0C36f8Fb5c38396573dDB487AC95a98256) == true` and
`executorCallerCount() == 1`, so the Mimic smart account can drive keeper automation.
Proposed as DAO Safe nonce 21 through the off-chain Safe delegate
`0x1483E048a76A93a3A59bBfA6d60471eA4990e922`, executed 2026-09-15 11:42:47 UTC with 3/4
signatures (tx `0x41e5a64096c938a4f98c57da0e2edec0856adfed802812523d48a2cecc515211`, block
25982636); the 48h delay elapsed 2026-09-17 11:42:47 UTC.
**Operation id (QueueKeeperExecutor):** `0xa4c9f4d382fa7196d669dc24528d5ac92950c4b8b31b43dcdec5569f505f6e40`
**Operation id (StrategyKeeperExecutor):** `0xb716df945dcc361695182bfad5553ffb7b6dd2cebfba54b7f4b249ac06b880b5`
**Predecessor (both ops):** `0x4b6a7a40449523fa21d7f268807beddb8a670319b4bce25ca142d9696fcd9fd6` — [011](../011-strategy-manager-add-unicl-weth-usdt-strategy/), WETH/USDT UniCL strategy registration

## What it changes

Binds the Mimic automation smart account `0x41153f0C36f8Fb5c38396573dDB487AC95a98256` as an
allowed executor caller on **both** keeper executor contracts:

| # | Target | Call | Effect |
|---|---|---|---|
| 1 | QueueKeeperExecutor `0xb7D76E4334e9E23B6edFF77e0C05B07E938a090B` | `allowExecutorCaller(0x4115…8256)` | Mimic may drive redemption-queue upkeep (`perform(uint8,bytes)`), actions `PriceBatch` / `ProcessRequests` / `AdvanceCursor` |
| 2 | StrategyKeeperExecutor `0xE94F714fbBF85c421b6FB80D3Ce0DFA20110EA29` | `allowExecutorCaller(0x4115…8256)` | Mimic may drive strategy upkeep (`perform(uint8)`), actions `Rebalance` / `WithdrawShortfall` / `DepositExcess` / `HarvestPerformanceFees` / `Sync` / `ProvideExitLiquidity` |

Both contracts currently have `executorCallerCount() == 0`, which makes every `perform()` call
revert `KeeperExecutorNoAllowedCallers()` (`0x0e1c8dbc`) — the executors are **inert** until this
proposal lands. It is the final on-chain step of the keeper-automation rollout; the
deployment runbook (`DeployKeeperExecutors` NatSpec) lists it as step 3, after the Mimic
functions are deployed and their triggers created.

This is an **allowlist only**: it grants no role on the Registry. Both executors already hold
`KEEPER_ROLE` (`Registry.hasRole(KEEPER_ROLE, executor) == true`); no role grant, no code
change, no fund movement. The executors hold **zero** token balances and cannot move protocol
funds beyond the actions their `perform()` entrypoints already permit, all of which
re-validate against live state — the perform payload is untrusted and amounts are never taken
from it.

## Why

The DAO is switching keeper automation on: the Mimic smart account runs the off-chain
functions (off-chain `checker()` read → calldata relay for the strategy keeper; queue
deep-scan for the queue keeper) that submit `perform()` calls. Without an allowlisted
caller the executors reject every upkeep call, so the queue and the strategies stay
manual. See the `DeployKeeperExecutors` NatSpec steps 1–4 and
`../contracts` `automation/KeeperExecutorBase.sol`.

**Ordering is intentional and enforced on-chain.** Both operations carry 011's operation id as
`predecessor`, so `execute` reverts `TimelockUnexecutedPredecessor` (`0x90a9a618`) until the
WETH/USDT strategy registration is `Done`. Automation therefore cannot be turned on before the
strategy it is meant to service is live. **011 executed on 2026-09-16 11:46:59 UTC, so this gate is
now cleared** — 012's own 48h delay closes at 2026-09-17 11:42:47 UTC, after which nothing but gas
stands between the queued batch and live automation.

## Transactions

DAO Safe schedule batch (2 entries, wrapped by `MultiSendCallOnly`
`0x9641d764fc13c8B624c04430C7356C1C7C8102e2`), then permissionless execution.

| # | Call | Target | Function | Op id |
|---|---|---|---|---|
| 1 | `schedule` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` | `schedule(QueueKeeperExecutor, 0, allowExecutorCaller(Mimic), 011-opid, salt, 172800)` | `0xa4c9f4d3…5f6e40` |
| 2 | `schedule` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` | `schedule(StrategyKeeperExecutor, 0, allowExecutorCaller(Mimic), 011-opid, salt, 172800)` | `0xb716df94…06b880b5` |

### Parameters

| Parameter | Value |
|---|---|
| Inner selector | `allowExecutorCaller(address)` = `0xb76fa138` |
| Inner calldata | `0xb76fa13800000000000000000000000041153f0c36f8fb5c38396573ddb487ac95a98256` |
| `value` | `0` |
| `delay` | `172800` (48h, the enforced minimum) |
| `salt` | `0x1e9fefcfe014e4103ee70beacf32f3d393bb3ca8535b28716d4e9370ddae1cce` = `keccak256("everstrat/automation/allow-executor-caller/mimic-smart-account/2026-09-15")` |
| `predecessor` | `0x4b6a7a40…9fd6` (011 op id) — **not** `0x00…00` |
| Caller allowlisted | `0x41153f0C36f8Fb5c38396573dDB487AC95a98256` (Mimic smart account, `isContract = true`) |

Files: `01-schedule.json` (ABI-resolved builder form, 2 entries), `01-schedule-raw.json`
(raw-calldata variant, selector `0x01d5062a`), `02-execute.json` (permissionless execute).

## Verification performed

All 12 steps run on an anvil fork of Ethereum mainnet at block `25982247`
(`anvil --fork-url https://ethereum-rpc.publicnode.com`), via cast.

| Step | Action | Result |
|---|---|---|
| 0 | `perform(0,0x)` on both executors while allowlist empty | revert `KeeperExecutorNoAllowedCallers()` `0x0e1c8dbc` |
| a | `allowExecutorCaller(Mimic)` from a role-less EOA | revert `RegistryClientMissingRole(ADMIN_ROLE)` `0x4d616cff` |
| b | DAO Safe `schedule` ×2 | OK, gas `56591` each; `getOperationState == 1` |
| b′ | `hashOperation(target,0,inner,predecessor,salt)` | equals both recorded op ids exactly |
| c | `execute` before the delay | revert `TimelockUnexpectedOperationState` `0x5ead8eb5` |
| d | `evm_increaseTime 172801` + mine | both ops `getOperationState == 2` (Ready) |
| e | `execute` with predecessor 011 not yet `Done` | revert `TimelockUnexecutedPredecessor` `0x90a9a618` |
| f | `execute` 011 (permissionless, from role-less EOA) | OK, gas `349592`; 011 `Done` |
| g | `execute` both 012 ops (role-less EOA) | OK, gas `117480` / `117469` |
| h | getters | `isExecutorCaller(Mimic) == true` on both; `executorCallerCount() == 1` on both |
| h′ | `perform(0,0x)` from a role-less EOA, allowlist now non-empty | revert `KeeperExecutorUnauthorizedCaller` `0x5e6ac3a8` (strict per-caller gate is active) |
| h″ | `perform(0,0x)` from the Mimic itself | passes the caller gate; rejected inside by `KeeperExecutorUnknownAction()` `0x16a2ea8f` (action `None` is not a valid action) — confirms the allowlist is what gates entry |
| i | re-`execute` an executed op | revert `TimelockUnexpectedOperationState` `0x5ead8eb5` |
| i′ | duplicate `allowExecutorCaller(Mimic)` from the timelock | OK, gas `32269`, **no** `ExecutorCallerAllowed` event, `executorCallerCount()` unchanged at `1` — the function is idempotent (`if (_executorCallers.add(c))`, no revert on duplicates) |

Source checks in `../contracts` at `origin/main`: `allowExecutorCaller` is
`onlyAuthRole(Auth.ADMIN_ROLE)` with no `whenNotPaused`; it reverts only on
`address(0)` (`KeeperExecutorUnauthorizedCaller`). The `onlyExecutorCaller` modifier checks the
allowlist only — the empty-set guard fires before the per-caller check, which is why step 0 and
step h′ return different selectors.

## Risks

- **Automation can be armed without further signatures, but not before 011 is Done.**
  Once the 48h elapses *and* 011 is `Done`, `execute` is callable by anyone
  (`EXECUTOR_ROLE = address(0)`). The predecessor is the safety interlock: until then the
  proposal cannot take effect even if it is scheduled.
- **Mimic is an off-chain dependency.** The allowlist makes the executors *callable* by
  Mimic; it does not fund Mimic credits or create its triggers (`DeployKeeperExecutors`
  step 4 covers funding). If Mimic lapses, upkeep stops — the executors simply stay idle,
  they cannot be drained by being allowlisted.
- **Removal is instant.** `removeExecutorCaller(Mimic)` is a single ADMIN call, and it is
  also available to nobody else; the `SECURITY_ROLE` path to stop automation is
  `pause()` on the executors (`pause()` is `onlyEitherAuthRole(ADMIN_ROLE, SECURITY_ROLE)`),
  with no timelock. That is the break-glass if a caller misbehaves.
- Allowlisting is per-contract; a second smart account or a rotation requires a new proposal.

## Dependency on 011

`predecessor` is set to 011's operation id, so `TimelockUnexecutedPredecessor` blocks both
operations until the WETH/USDT strategy is registered. This is the deliberate "turn automation
on right after the strategies are live" sequencing. Alternative if the DAO prefers earlier
queue automation: the QueueKeeperExecutor op is technically independent of strategy
registration (queue upkeep does not touch strategies) and could be scheduled with
`predecessor = 0x00…00`. That split would need a separate proposal — this record keeps the
two calls coupled, as requested.

## Provenance

Built from the task in the DAO chat (bind the Mimic smart account
`0x41153f0C36f8Fb5c38396573dDB487AC95a98256` to both keepers, gated on 011). Every value in
this record is derived programmatically at build time from the timelock ABI, the executor
addresses in the root README, and the 011 operation id; no hex is transcribed by hand. Fork
simulation and selectors recorded as observed.

## On-chain schedule (mainnet)

| Field | Value |
|---|---|
| Proposed by | off-chain Safe delegate `0x1483E048a76A93a3A59bBfA6d60471eA4990e922` ("Everhomie"); `proposer` recorded as its delegator `0x4A2D30c7b9f7907D580f9A1902D42dd78B21F0d2` |
| Scheduled via | DAO Safe `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9`, nonce 21 (batch of 2 `schedule` ops) |
| Signatures | 3 of 4 — `0x4A2D…F0d2` 11:28:06 · `0xe9BE…dc4a` 11:38:46 · `0x1Efb…9a46` 11:42:13 UTC |
| Execute (Safe) | 2026-09-15 11:42:47 UTC — tx `0x41e5a64096c938a4f98c57da0e2edec0856adfed802812523d48a2cecc515211`, block 25982636 |
| Operation state | `Waiting` for both ops (re-checked 2026-09-15 12:40 UTC) |
| Executable from | 2026-09-17 11:42:47 UTC — after 011 is `Done` (`predecessor`) |
| Effect | both executors report `isExecutorCaller(0x4115…8256) == true`, `executorCallerCount() == 1` |

## On-chain execution (mainnet)

Executed on 2026-09-17, one call per operation, both from `0x046E01eE…a899D7`
(permissionless — `EXECUTOR_ROLE` is `address(0)`). 011 was already `Done`, so the
`predecessor` gate was satisfied.

| Operation | Transaction | Block | Executed (UTC) | Gas |
|---|---|---|---|---|
| QueueKeeperExecutor `0xa4c9f4d3…5f6e40` | `0x0441b1f863b9e70a25c52c9eecc50c25fcfd970520bcd41b2000fa25cd788049` | 25997085 | 2026-09-17 12:02:59 | 117,480 |
| StrategyKeeperExecutor `0xb716df94…06b880b5` | `0x08760b797858c72e59a5b52c3b60ffdf85896f6614a9205be777c76775e0df3f` | 25997092 | 2026-09-17 12:04:23 | 117,469 |

Gas matches the fork simulation (117,480 / 117,469) exactly.

**Getters after execution** — the state change the proposal was for:

| Executor | `isExecutorCaller(Mimic)` | `executorCallerCount()` | `isExecutorCaller(0x1111…1111)` |
|---|---|---|---|
| QueueKeeperExecutor `0xb7D76E43…090B` | **true** | **1** | false |
| StrategyKeeperExecutor `0xE94F714f…EA29` | **true** | **1** | false |

**Live gate exercised** — `perform(...)` (`0x16092623` / `0x16d9fdd2`) against current mainnet
state, both executors:

| Caller | Result |
|---|---|
| Mimic `0x41153f0C…8256` | revert `KeeperExecutorUnknownAction()` `0x16a2ea8f` — the caller gate **passes**, execution then stops because action `None` is not a real action |
| Stranger `0x1111…1111` | revert `KeeperExecutorUnauthorizedCaller(0x1111…1111)` `0x5e6ac3a8` — rejected at the gate |
| *(before this proposal)* any caller | revert `KeeperExecutorNoAllowedCallers()` `0x0e1c8dbc` |

So the allowlist is not just stored — it is the thing gating entry, and only the Mimic passes it.

## Cancelling

Before execution: `cancel(opId)` from the DAO Safe or the Security Safe
`0x7c128C1CF39822B4133F8E067F4bF14999c49846`, or simply leave it unexecuted. After execution:
`removeExecutorCaller(Mimic)` on the affected executor (48h via ADMIN), or `pause()` on the
executor (instant, ADMIN or SECURITY).
