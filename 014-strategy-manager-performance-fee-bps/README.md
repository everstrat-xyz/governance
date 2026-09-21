# 014 — Set StrategyManager `performanceFeeBps` to 1000 (10%)

**Status:** 🟡 **PREPARED — not yet submitted to the Safe.** All payloads are built, fork-simulated
and the SafeTx is signed, but the Safe Transaction Service refused the submission
(`429 Monthly quota exceeded`, `x-ratelimit-remaining: 0`, reset in ~10 days). Submit via the
Safe UI (see [How to submit](#how-to-submit)) — the transaction service quota is per-IP, so an
owner's browser is unaffected. Requested by **Arseny** (2026-09-21).

**Operation id:** `0xdca0bcc486e9c3506a0430e87b5573988c18dc398f070ccd5efc6c403b409b45`

## What it changes

One call, one operation:

| # | Target | Call | Effect |
|---|---|---|---|
| 1 | StrategyManager `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` | `setPerformanceFeeBps(1000)` | Performance fee switches from **0 → 1000 bps (10%)** |

After this executes, `harvestPerformanceFeeFromStrategies()` can for the first time mint the
accrued performance fee to `daoTreasury()` — which is the **DAO Safe itself**
(`0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9`), so the fee accrues to the DAO treasury.

## Why

Today `performanceFeeBps()` reads `0` on mainnet — the fee is **disabled**, so the already-accrued
LP-fee base cannot be collected (вадюша, 2026-09-21: "они копятся, но собрать пока нельзя"). Setting
a non-zero rate:

- turns fee collection on;
- and, because the rate applies to the **whole uncharged base**, also makes the performance fee
  accrued over the entire preceding period harvestable.

⚠️ **This is retroactive.** Per audit finding **M15**: `_setPerformanceFeeBps` does not settle
outstanding fees before the rate change, so the new rate charges the full uncharged base. That is
the intended behaviour here (Arseny explicitly wants the prior period collectable), but it is the
reason the rate is a policy decision and not a mechanical one.

**Value chosen: 1000 bps (10%).** The parameter was unset at deploy and no rate is recorded in the
vault — only the contract range (`0 – MAX_PERFORMANCE_FEE_BPS = 2000`). 1000 bps is the
conservative, industry-common choice. **Confirm the rate before collecting signatures** — changing
it only requires a new schedule at the current nonce, not a cancel.

## Transactions

| # | Safe tx | Timelock call |
|---|---|---|
| 1 | `schedule(StrategyManager, 0, setPerformanceFeeBps(1000), 0x00…00, salt, 172800)` | `StrategyManager.setPerformanceFeeBps(1000)` |

### Parameters

| Field | Value |
|---|---|
| `target` | `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` (StrategyManager) |
| `value` | `0` |
| `data` | `0x9f0caac900000000000000000000000000000000000000000000000000000000000003e8` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0x2115f78dfb605f8591296be39bd99ddad345076ecbe6c2e21ec2d320472598c0` |
| `delay` | `172800` (48h) |
| operation id | `0xdca0bcc486e9c3506a0430e87b5573988c18dc398f070ccd5efc6c403b409b45` |

`salt = keccak256("everstrat/sm/performance-fee-bps/2026-09-21")`. No predecessor — this action has
no on-chain dependency on any other scheduled op.

### `setPerformanceFeeBps` preconditions (read from `../contracts/src`)

- `onlyAuthRole(ADMIN_ROLE)` → verified held by the admin timelock `0xF091…` (`hasRole` = true) and
  **not** by the DAO Safe (`hasRole` = false).
- Emits `PerformanceFeeBpsChanged(previous, new)`.
- No feed read, no NAV dependency → the add-only path simulates cleanly on a time-warped fork.
- `MAX_PERFORMANCE_FEE_BPS = 2000` → 1000 is within range.

## Verification performed

Mainnet fork (`anvil --fork-url https://ethereum-rpc.publicnode.com`), 2026-09-21:

| Step | Result |
|---|---|
| (a) `setPerformanceFeeBps(1000)` from non-admin EOA | revert `RegistryClientMissingRole` `0x4d616cff` ✅ |
| (b) `performanceFeeBps()` before | `0` ✅ |
| (c) DAO Safe → `schedule(…)` | success, `gasUsed 56,071` ✅ |
| (d) `execute(…)` before delay | revert `TimelockUnexpectedOperationState` `0x5ead8eb5` ✅ |
| (e) warp +48h | — |
| (f) `execute(…)` from unrelated EOA (`EXECUTOR_ROLE = address(0)`) | success, `gasUsed 75,111` ✅ |
| (g) `performanceFeeBps()` after | **`1000`** ✅ |
| (h) re-`execute` | revert `0x5ead8eb5` ✅ |

On-chain state check (2026-09-21, `https://ethereum-rpc.publicnode.com`):

| Check | Result |
|---|---|
| `StrategyManager.performanceFeeBps()` | `0` (no-op until this executes) |
| `StrategyManager.daoTreasury()` | `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9` (DAO Safe) |
| `Timelock.getOperationState(op id)` | `0` — Unset, nothing scheduled ✅ |
| `Registry.hasRole(ADMIN_ROLE, timelock)` | `true` ✅ |

## How to submit

The Tx Service is the only path that records a *proposal*; it is quota-blocked for this agent's IP.
An owner can create it in the Safe app in ~1 minute (their IP has its own quota):

**Tx Builder** → paste target `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9`, leave
`contractMethod` as `schedule(...)` on the Timelock, and fill the parameters table above; **or**
use `01-schedule-raw.json` (raw `0x01d5062a` calldata) if the UI cannot resolve the Timelock ABI.

Alternatively, wait for the quota to reset (~2026-10-01) and the agent can submit it directly.

## Cancelling

Nothing is scheduled, so there is nothing to cancel. If the rate should differ, discard this and
schedule the new value — the operation id simply differs by the `data`/`salt` inputs.
