# 015 — StrategyManager: register the UniCL UNI/WETH 0.3% strategy

**Status:** ❌ **Withdrawn — never scheduled.** Filed in the DAO Safe queue at nonce 24
(2026-09-21 16:29:08 UTC) and withdrawn 11 minutes later, before any owner signed. A Safe
rejection at the same nonce executed on 2026-09-22 13:39:11 UTC (tx
`0x06eed20458f42cbafacdcf4fedcd7ac2969c2d709020b33a7c09809e58e3113d`, block 26033382), so the
015 Safe tx is permanently unexecutable. Nothing reached the timelock
(`getOperationState == 0` Unset) and `isStrategyRegistered(0x2c3A…715d) == false`.
**Operation id (never scheduled):** `0x489a556a585fbae6efeb04d3f1627d9168124c63a2b089e29b2e5ba42201cd07`

## What it would have changed

`StrategyManager.addStrategy(0x2c3AEFaCb41065810f43F23d0eBa8dF20350715d, 10, 10)` — wire the
newly deployed UNI/WETH 0.3% concentrated-liquidity strategy into `StrategyManager` so it can
receive deposits and be valued in protocol NAV, with deposit and withdrawal weights of 10.

Deployed by Arseny on 2026-09-21 (tx `0x9481e0e2…5113d68`, 5.88M gas). Read back from the
contract:

| Param | Value |
|---|---|
| `pool` | `0x1d42064Fc4Beb5F8aAF85F4617AE8b3b5B8Bd801` — UNI/WETH, fee 3000, tickSpacing 60 |
| `pairedToken` | UNI `0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984` |
| `token0` / `token1` | UNI / WETH |
| `swapAdapter` | `0x0844580a121124CAEc6Cf4A933aac401813cCde5` ([005](../005-converter-allow-univ3-adapter/)) — `isAdapterAllowed` true |
| `registry` | `0x46AA1bd55c19be90d8767e0C22732A7DD31D993D` |
| `positionWidth` | 18 (±1080 ticks ≈ ±10.8%) |
| `rebalanceTickThreshold` | 480 |
| `maxTickDeviation` | 60 |
| `twapInterval` / `shortTwapInterval` | 1800 / 60 |
| `maxTotalNAV` | **250 ETH** (smallest of the four strategies) |
| `paused` | false |
| `totalDeposited` / `navInETH` | 0 / 0 — empty |

The pool's `observationCardinality` is 274, above the 150 minimum the 1800 s TWAP needs.

## Why it was withdrawn

Withdrawn by **Arseny** on 2026-09-21 16:40 UTC: *"it seems that the strategy is not completely
ready and we didn't enable Uni oracle."* Two open items at the time:

1. **UNI was not priceable.** [013](../013-add-uni-supported-token/) (UNI/USD feed +
   `addSupportedERC20(UNI)`) had not executed — see [Dependency on 013](#dependency-on-013).
   *Resolved:* 013 executed 2026-09-22 23:16:47 UTC, and UNI is now priceable and supported.
2. **The deployed binary is not attributable** — see [Risks](#risks).

## Why weights 10 / 10

Weights are normalised across eligible strategies, so this would dilute the existing three
rather than add a fourth share: 40/15/45/10 → **36.4 / 13.6 / 40.9 / 9.1 %**. 10 is the most
conservative weight in the set: the smallest cap of the four (250 ETH vs 1000 / 140 / 4500); UNI
is the first paired token that is neither ETH nor a USD stable, so the position carries
idiosyncratic single-asset risk; and the strategy is unproven in production.

## Dependency on 013

`predecessor` was set to **013's `addSupportedERC20(UNI)` op**
(`0xcd443da64d612cc5bc96157197e5ab62c5207d1d45d402be1d2e5170a5f35ae9`). `addStrategy` itself
succeeds with UNI unsupported — it only checks role, pause state, code presence and duplicates —
but a strategy holding UNI is unpriceable until 013 lands, and that is a protocol-wide hazard.
Measured on a fork pinned to block 26027017, registering first and then giving the strategy 1 UNI:

| Step | Result |
|---|---|
| `addStrategy` with 013 unexecuted | OK (249,565 gas) — registration alone is fine |
| `StrategyManager.totalNAVInETH()`, strategy empty | works (strategy is worth 0) |
| `strategy.navInETH()` with 1 UNI held | revert `OracleTokenNotSupported()` `0x868fd74e` |
| `StrategyManager.totalNAVInETH()` with 1 UNI held | revert `0x868fd74e` — **NAV frozen protocol-wide** |
| …then execute 013 op1 + op2 | both calls work again |
| `strategy.navInETH()` after 013 | 3,257,672,436,673,538 wei ≈ 0.0032577 ETH for 1 UNI |

The predecessor makes that ordering impossible to get wrong. Alternative: `predecessor = 0x00…00`
and sequence manually — the operation id changes, and the hazard above becomes an operational
rule instead of an on-chain one. If 013 were ever cancelled, this op would have to be re-scheduled
with a new predecessor.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` with `delay = 172800` — **withdrawn, do not sign** |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` — never applicable |

`01-schedule-raw.json` is the same transaction as raw calldata (selector `0x01d5062a`).

### Parameters

| Field | Value |
|---|---|
| `target` | StrategyManager `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` |
| `value` | `0` |
| `data` | `addStrategy(address,uint8,uint8)` (`0xdca0c48f`) — `0xdca0c48f0000000000000000000000002c3aefacb41065810f43f23d0eba8df20350715d000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000a` |
| `predecessor` | `0xcd443da64d612cc5bc96157197e5ab62c5207d1d45d402be1d2e5170a5f35ae9` (013 op2) |
| `salt` | `0x04c997170365fd7c8ae41d6545e71cd8636caacda545cb6c146a8e7b81dc40b2` = `keccak256("everstrat/strategy-manager/add-strategy/unicl-uni-weth-0.3pct/2026-09-21")` |
| `delay` | `172800` |

## Verification performed

Fork (anvil, mainnet pinned to 26027017), full path through the timelock:

| Step | Result |
|---|---|
| direct `addStrategy` from a non-ADMIN account | revert `RegistryClientMissingRole` `0x4d616cff` |
| `schedule` from the DAO Safe | OK, 57,420 gas |
| `execute` before the 48h delay | revert `TimelockUnexpectedOperationState` `0x5ead8eb5` |
| `execute` after 48h with 013 op2 unexecuted | revert `TimelockUnexecutedPredecessor` `0x90a9a618` |
| `execute` after 013 op2 is Done | OK, **269,246 gas**; registered, `strategyCount` 3 → 4, weights 10/10, `CONVERTER_CALLER_ROLE` granted |
| re-`execute` | revert `0x5ead8eb5` |
| `hashOperation(...)` | reproduces `0x489a556a…2201cd07` |

## Risks

**Bytecode provenance — open item.** This deployment is not reproducible from any committed
revision of `../contracts`, and it was compiled with different settings than the three strategies
already registered:

| Binary | Runtime size | CBOR metadata |
|---|---|---|
| `0x2c3AEF…0715d` (this one) | **24,570 B** | 10 B — solc 0.8.30, **IPFS hash stripped** |
| WETH/USDT strategy (registered) | 24,391 B | 51 B — IPFS hash present |
| `origin/main` (49c0e63) build | 24,391 B | 51 B — IPFS hash present |

The `origin/main` build reproduces the registered binaries (differences confined to immutable
slots). This contract is **+179 bytes**, differing in 20,592 bytes across 568 runs — a different
build, not different constructor arguments. Other build profiles don't reach 24,570 B either
(default / `via_ir` 24,391 B; `bytecode_hash = none` 24,350 B; `optimizer_runs = 1000000`
32,912 B), so the delta is a **source** difference. The ABI surface is identical (61 of 61
dispatcher selectors match the registered strategy), so there is no new external or privileged
entry point — but the audit's coverage cannot be claimed for it. **Before any re-proposal the
deployer should supply the commit and build profile used.**

## Withdrawal (mainnet)

| Field | Value |
|---|---|
| Filed | DAO Safe nonce 24, safeTxHash `0x49019656754da2fb6e5146e0b5163bd2770ba603b882abb9cddcbd4c78f7c1f9`, submitted 2026-09-21 16:29:08 UTC by delegate `0x1483E048…0E922` (proposer `0x4A2D…F0d2`), origin `everstrat/015-strategy-manager-add-unicl-uni-weth-strategy`; stored `data` equal to `01-schedule-raw.json` |
| Signatures on 015 | none |
| Rejection | Safe → itself, `value 0`, `data 0x`, nonce 24, safeTxHash `0xe51719fe3911bd8d37f5b56939f17a415d9fbf73437a28b8caa3d8dacabff734`, submitted 2026-09-21 16:40:52 UTC |
| Rejection signatures | 3 of 4 — `0x4A2D…F0d2` 2026-09-21 16:43:01 · `0x1Efb…9a46` 2026-09-22 12:44:59 · `0xe9BE…dc4a` 13:12:05 UTC |
| Rejection executed | 2026-09-22 13:39:11 UTC — tx `0x06eed20458f42cbafacdcf4fedcd7ac2969c2d709020b33a7c09809e58e3113d`, block 26033382; `ExecutionSuccess(0xe51719fe…abff734)`; relayed by `0x4A2D…F0d2` via the MetaMask `DelegationManager` |
| Result | Safe nonce advanced 24 → 25; the 015 Safe tx can never execute. `getOperationState(0x489a556a…2201cd07) == 0`, `isStrategyRegistered(0x2c3A…715d) == false` (re-checked 2026-09-22 23:03 UTC) |

There was never a timelock operation, so no `cancel(bytes32)` was involved: the Safe
Transaction Service does not expose deletion of a queued tx, and a same-nonce rejection is the
enforceable way to kill one.

## Cancelling

Nothing to cancel. 013 has executed, so once the binary is attributed a re-proposal can re-file
`01-schedule-raw.json` at the Safe's current nonce — the inner calldata, predecessor and salt are
unchanged, so the operation id stays `0x489a556a…2201cd07`. A re-proposal after the strategy is
redeployed needs new calldata, a new salt and a new directory.
