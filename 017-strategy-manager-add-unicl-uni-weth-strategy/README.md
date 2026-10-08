# 017 — StrategyManager: register the UniCL UNI/WETH 0.3% strategy (re-deployed build)

**Status:** ✅ **Executed on mainnet** (tx
`0x3668b911fe773a6a8dec417a49334cbbbbcd474a5ee4dba264518ff950bd30ac`, block 26061446,
2026-09-26 11:52:11 UTC) — permissionless `execute` after the 48h delay.
`isStrategyRegistered(0x956F…AfDd) == true`, `strategyCount() == 4`, weights 10 / 10. Scheduled
2026-09-23 12:49:47 UTC via DAO Safe nonce 26 (tx
`0x534f557af6b7744329b581e0f6ea0cf9f62142ce5b4a214212376d309e37c2a6`, block 26040285).
**Operation id:** `0x941ba6025a8faaa5670a5ac1802076259d9b22dc5ea85cfb4a7b727f7e035210`

## What it changes

One `schedule` on the 48h admin timelock:
`StrategyManager.addStrategy(0x956FE55Db50527e75C1fca679Ade509AD89cAfDd, 10, 10)`.

That makes the strategy the **4th registered strategy** — `strategyCount()` 3 → 4,
`isStrategyRegistered` false → true, `depositWeight` / `withdrawalWeight` 0 → 10. The
Converter's `CONVERTER_CALLER_ROLE` is granted to it by `addStrategy` itself, not by a
second operation.

This **supersedes the withdrawn [015](../015-strategy-manager-add-unicl-uni-weth-strategy/)**:
same protocol action, different strategy address. 015 targeted `0x2c3AEFaCb41065810f43F23d0eBa8dF20350715d`
and was withdrawn on 2026-09-21 before any owner signed it.

## Why

It wires the UniCL UNI/WETH 0.3% concentrated-liquidity strategy into the protocol so it can
receive deposits, be rebalanced by the keeper, and be counted in NAV.

## Dependency on 013 — resolved, therefore no predecessor

015 had to carry `predecessor = 013 op2`, because registering a strategy that can hold UNI
while UNI had no oracle feed made `navInETH()` **and** `StrategyManager.totalNAVInETH()`
revert `OracleTokenNotSupported` (`0x868fd74e`) — a protocol-wide NAV freeze.

**013 has since executed.** Verified live on 2026-09-23: `Oracle.isTokenSupported(UNI)` =
true, `StrategyManager.supportedERC20()` = `[USDC, USDT, UNI]`, and
`Timelock.getOperationState(013 op2)` = **3 (Done)**.

So the hazard is gone and this operation carries `predecessor = 0x00…00`. Keeping the
predecessor would still pass (a Done predecessor satisfies the check) but would encode a
dependency that no longer exists. Consequence: **this op can execute as soon as its own 48h
elapses** — it is blocked by nothing.

## Transactions

| # | target | value | call |
|---|---|---|---|
| 1 | Timelock `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` | 0 | `schedule(StrategyManager, 0, addStrategy(...), 0x0, salt, 172800)` — selector `0x01d5062a` |
| — | *inner call, executed by the timelock at T+48h* | 0 | `addStrategy(0x956FE55D…, 10, 10)` on `0x94916ab9…b5C9` — selector `0xdca0c48f` |

### Parameters

| parameter | value |
|---|---|
| target | `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` (StrategyManager) |
| value | `0` |
| data | `addStrategy(0x956FE55Db50527e75C1fca679Ade509AD89cAfDd, 10, 10)` |
| predecessor | `0x00…00` (none) |
| salt | `keccak256("everstrat/strategy-manager/add-strategy/unicl-uni-weth-0.3pct/2026-09-23")` = `0x849360ad72bce67bea7ea1a7c3964e6d7b046a7527d4c08abbb7f336eb938c37` |
| delay | `172800` (48h) |

`10/10` matches the scale of the existing strategies (011 used 10/10). The setters revert
above `MAX_DEPOSIT_WEIGHT` / `MAX_WITHDRAWAL_WEIGHT`, and both weights are re-settable later
by governance without touching registration.

## The strategy

| field | value |
|---|---|
| address | `0x956FE55Db50527e75C1fca679Ade509AD89cAfDd` |
| paired token | UNI `0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984` |
| pool | `0x1d42064Fc4Beb5F8aAF85F4617AE8b3b5B8Bd801` (UNI/WETH 0.3%, fee 3000) |
| swapAdapter | `0x0844580a121124CAEc6Cf4A933aac401813cCde5` (Uniswap V3) |
| pairedTokenToWethPath | UNI → 3000 → WETH |
| swapSlippageBps | `100` |
| maxTotalNAV | 250 ETH |
| positionWidth / rebalanceTickThreshold / maxTickDeviation | 18 / 480 / 60 |
| twapInterval / shortTwapInterval | 1800 / 60 |
| paused / isHealthy() | `false` / `true` |
| runtime code | 24,575 B |

The route config is validated in the constructor (`_validateRouteConfig`), so a wrong path
cannot be deployed; the values above were read back from chain and are consistent.

## Verification performed

- **Operation id computed three ways and agreeing**: local `keccak(abi.encode(target, 0, data, 0x0, salt))`,
  on-chain `Timelock.hashOperation(...)`, and the value recorded in the JSON files.
- **Fork simulation** (mainnet fork, `anvil --auto-impersonate`, DAO Safe funded):
  - `schedule` sent from the DAO Safe → success, **57,036 gas**, state **1 (Waiting)**
  - after `evm_increaseTime 172801` → state **2 (Ready)**
  - `execute` → success, **266,541 gas** ⇒ `strategyCount` 3 → **4**, `isStrategyRegistered`
    **true**, `depositWeight` **10**, `withdrawalWeight` **10**, `Converter.isCaller(new)`
    **true**
  - re-`execute` → `TimelockUnexpectedOperationState(op, 0x4)` (Done) — correctly refused
- **Pre-flight on mainnet**: `strategyCount()` = 3, `isStrategyRegistered(new)` = false.
- **Source preconditions read**: `addStrategy` is `onlyAuthRole(ADMIN_ROLE)`, `whenNotPaused`,
  `nonReentrant`, and reverts on zero address, missing code, or double registration.
  `grantCallerRole` is deliberately *not* wrapped in try/catch, so a failure reverts the whole
  call rather than silently registering a strategy with no converter privileges.

## Risks / open items

- **The contract has 1 byte of EIP-170 headroom — 24,575 B of the 24,576 B limit.** It deploys
  and runs correctly, but any future fix to this contract cannot be deployed as-is.
- **Bytecode provenance is not closed.** Reported as built from `origin/main @ b599469`, but the
  repo's own profile (`via_ir = true`, `optimizer_runs = 200`, cancun) builds **24,396 B** against
  the deployed **24,575 B** — a **+179 B** gap. Ruled out by direct measurement: shanghai 24,382,
  cancun 24,355, paris 24,711, `via_ir=false` 29,070, runs 400/600/800/1000/10,000/100,000.
  Metadata is also stripped (`bytecode_hash = none`) where the repo default emits IPFS. The
  withdrawn deployment carried the *same* +179 B offset, so this looks like a pipeline-level
  difference rather than a source one — but until the exact build settings are recorded the
  deployed binary is not reproducible from the stated commit.
- Registration is by **address**; this proposal deploys no code.

## Numbering

Drafted and filed as **016**: the Safe tx's `origin` reads
`EverStrat: schedule 016 addStrategy(UniCL UNI/WETH 0.3% v2)`. Another proposal,
[016](../016-strategy-keeper-exit-settlement-funding/) (keeper exit-settlement funding), had already
been filed at Safe nonce 25 under the same number, so this record was renumbered to **017** when
it merged. Nothing on-chain depends on the number: the JSON payloads, salt and operation id are
unchanged.

## On-chain schedule (mainnet)

| Field | Value |
|---|---|
| Proposed by | off-chain Safe delegate `0x1483E048a76A93a3A59bBfA6d60471eA4990e922`; `proposer` recorded as its delegator `0x4A2D30c7b9f7907D580f9A1902D42dd78B21F0d2`; submitted 2026-09-23 11:22:47 UTC |
| Scheduled via | DAO Safe `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9`, nonce 26 (single `schedule`) |
| safeTxHash | `0x844e87d0b91f87e94f888f21a5a33e75625c1f810fa85b246348e343aff6f51d` |
| Signatures | 3 of 4 — `0xe9BE…dc4a` 11:52:45 · `0x4A2D…F0d2` 12:02:53 · `0xF412…E149` 12:14:54 UTC (2026-09-23) |
| Execute (Safe) | 2026-09-23 12:49:47 UTC — tx `0x534f557af6b7744329b581e0f6ea0cf9f62142ce5b4a214212376d309e37c2a6`, block 26040285, executor `0x1Efb…9a46` |
| Calldata check | Safe tx `data` equals `01-schedule-raw.json` byte-for-byte |
| Ready at | 2026-09-25 12:49:47 UTC |

## On-chain execution (mainnet)

| Field | Value |
|---|---|
| Executed (UTC) | 2026-09-26 11:52:11 |
| Transaction | `0x3668b911fe773a6a8dec417a49334cbbbbcd474a5ee4dba264518ff950bd30ac` |
| Block | 26061446 |
| Gas | 266,541 — identical to the fork simulation |
| Caller | `0x046E01eE…a899D7` (permissionless — `EXECUTOR_ROLE` is `address(0)`) |
| Events | `DepositWeightUpdated`, `WithdrawalWeightUpdated`, `CallerRoleGranted`, `StrategyAdded(0x956F…AfDd)`, `CallExecuted` |

**Effect verified on-chain** (block 26149662, 2026-10-08): `isStrategyRegistered(0x956F…AfDd) == true`;
`strategies()` = `[0x5E12…ac38, 0x59C4…2252, 0x3Fb6…6501, 0x956F…AfDd]`; `depositWeight` /
`withdrawalWeight` = `10` / `10`; `Converter.isCaller(0x956F…AfDd) == true`; the strategy is
unpaused and already holds capital (`navInETH() = 116070477577341208`, ≈ 0.116 ETH). The runtime
code is 24,575 B, the size the provenance note below refers to.

## Cancelling

No longer possible — the operation is executed. To unwind: `removeStrategy` /
`forceRemoveStrategy` (`ADMIN_ROLE`, 48h); `setDepositWeight` / `setWithdrawalWeight` retune
weights without removing it.
