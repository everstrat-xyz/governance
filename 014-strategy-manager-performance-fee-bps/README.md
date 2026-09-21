# 014 — Set StrategyManager `performanceFeeBps` to 1500 (15%)

**Status:** 🟡 **PREPARED — not yet submitted to the Safe.** Payloads are built and fork-verified and
the SafeTx is signed, but the Safe Transaction Service returned
`429 {"error_msg":"Monthly quota exceeded"}` on 2026-09-21 (`x-ratelimit-limit: 5000`,
`x-ratelimit-remaining: 0`, reset **2026-10-01 14:12 UTC**). No `SAFE_API_KEY` is configured, and the
legacy host redirects into the same quota. This is the same exhaustion documented for the
safe-watch listener; the fix is a Safe API key, or an owner filing the tx from the Safe app.

**Requested by:** Arseny — *"set it to 15 and open a proposal on safe"* (2026-09-21), after вадюша
reported that `performanceFeeBps` is still `0`, so the performance fee **accrues but cannot be
harvested**.

## Why

The protocol takes a performance fee on the strategy LP fees it earns. With `performanceFeeBps = 0`
the fee share still accrues but **cannot be collected** — `StrategyManager` has no way to settle it.
Setting a non-zero rate is what unlocks collection, and per audit finding **M15**
(`_setPerformanceFeeBps` does not settle outstanding fees first) the new rate applies to the
**entire previously uncharged LP-fee base**, i.e. the retroactive period becomes collectable too.

`performanceFeeBps` is an `ADMIN_ROLE` parameter (48h timelock) — it can only be changed through
governance, never by SECURITY or KEEPER. Bounds: `0 – MAX_PERFORMANCE_FEE_BPS (2000)`.

The fee is not minted to an ops address: `daoTreasury()` is `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9`
— **the DAO Safe itself**, so the fee accrues to the treasury.

## Payload

| | |
|---|---|
| Timelock | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (48h) |
| Inner target | StrategyManager `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` |
| Inner data | `setPerformanceFeeBps(1500)` = `0x9f0caac9…000005dc` |
| `value` | `0` |
| `predecessor` | `0x00…00` (none) |
| `salt` | `0x2115f78dfb605f8591296be39bd99ddad345076ecbe6c2e21ec2d320472598c0` = `keccak256("everstrat/sm/performance-fee-bps/2026-09-21")` |
| `delay` | `172800` (48h) |
| **operation id** | `0xc4302e528a0a993eb9cc433dfc35ddace37bfef0cc3d2e7b97207d397f74ea46` |

`getOperationState(opId)` on mainnet = **0 (Unset)** — nothing is scheduled under this salt yet.

> **op-id derivation gotcha (cost me a cycle):** `hashOperation` is
> `keccak256(abi.encode(target, value, data, predecessor, salt))` — **five** fields. The `delay` is
> *not* part of the hash. Encoding six fields silently yields a well-formed but wrong 32-byte id that
> no explorer will ever match. The id above is cross-checked against the on-chain
> `hashOperation(address,uint256,bytes,bytes32,bytes32)` return value, not just computed locally.

## Transactions

### 1. `01-schedule.json` — Safe → Timelock, `schedule(...)`
`to` `0xF0911198…1774`, `value 0`,
`schedule(0x94916ab9…b5C9, 0, 0x9f0caac9…05dc, 0x00…00, 0x2115f78d…2598c0, 172800)`
SafeTx `safeTxHash = 0x201f9ad4…cdf4452`, **signed** by the registered delegate `0x1483E048…0E922`.

### 2. `02-execute.json` — permissionless
`execute(0x94916ab9…b5C9, 0, 0x9f0caac9…05dc, 0x00…00, 0x2115f78d…2598c0)` — anyone may call once
`Ready`. No Safe needed for this step.

## Verification performed

Simulated on a mainnet fork (anvil, real deployed contracts), 2026-09-21:

| step | result |
|---|---|
| non-admin calls `setPerformanceFeeBps(1500)` directly | reverts `RegistryClientMissingRole` `0x4d616cff` |
| DAO Safe calls `schedule(...)` | succeeds — gas **56,071** |
| `execute(...)` before the 48h delay | reverts `TimelockUnexpectedOperationState` `0x5ead8eb5` |
| `execute(...)` from an unrelated EOA after the delay | succeeds — gas **75,111** |
| `performanceFeeBps()` after | **0 → 1500** ✅ |
| `execute(...)` again | reverts `0x5ead8eb5` (already done) |
| `hashOperation(...)` | reproduces `0xc4302e52…f74ea46` exactly |

## To submit while the API is quota-dead

The quota is **per IP**, so an owner's browser is unaffected. In the Safe app → New transaction →
**Transaction Builder** → custom contract:

- `to`: `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774`
- raw data (`01-schedule-raw.json`), or method `schedule` with:
  `target 0x94916ab93C669E7c734f844dB019Ce9449a3b5C9`, `value 0`,
  `data 0x9f0caac900000000000000000000000000000000000000000000000000000000000005dc`,
  `predecessor 0x0000000000000000000000000000000000000000000000000000000000000000`,
  `salt 0x2115f78dfb605f8591296be39bd99ddad345076ecbe6c2e21ec2d320472598c0`,
  `delay 172800`

Then 3-of-4 confirmations as usual. The Safe's on-chain `nonce()` is **23**; the UI will pick the
right nonce (`nonce()` is RPC-readable and unaffected by the API quota).

## Cancelling

Only while `Waiting`/`Ready` and only via the Timelock: `cancel(opId)` requires `CANCELLER_ROLE`,
which is the DAO Safe — so cancelling before execution is itself a 48h-ish Safe → Timelock dance.
Simplest abort is simply never calling `execute`: the op id is unset until the schedule lands, so
**before submission there is nothing to cancel**.

## Consequences once executed

- `performanceFeeBps` = **1500 (15%)** of strategy LP fees, minted/accrued to `daoTreasury()` (the DAO Safe).
- **Retroactive**: the whole previously uncharged fee base becomes collectable at 15% (audit M15).
- Reducible later through the same 48h path — but a decrease is likewise retroactive, so if the rate
  is ever to be lowered, settle first.
