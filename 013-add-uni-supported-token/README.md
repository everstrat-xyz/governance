# 013 — Add UNI as a supported ERC-20 (UNI/USD Oracle feed + StrategyManager)

**Status:** ✅ **Executed on mainnet** — 2026-09-22 23:15:47 / 23:16:47 UTC (tx
`0x2c158399c99f1820b2bebe4101f0fdde529a6a3cc9e27a8ae239be66d7304f72` /
`0x25d9e669ffdb784be09560ce0b6e94e5ab14d6f24f415c3206bcea193736c39a`, blocks 26036243 / 26036248).
`Oracle.isTokenSupported(UNI) == true`, `getUsdFeedInfo(UNI) == (0x5533…220e, 4200)`,
`StrategyManager.isSupportedERC20(UNI) == true`. Scheduled 2026-09-19 07:18:11 UTC via DAO Safe
nonce 22 (tx `0x4b36e80469d8fe3f7d6cf71b34d82d3ab8ece17961365ec2e03ecb000d02afc7`, block
26010017); `Ready` from 2026-09-21 07:18:11 UTC, executed ~40h later.
**Operation ids:**
- op1 — UNI/USD feed: `0xeca687148460f11ee30984d2685c68a72a434233ff9bf2b984760bb606d49bbd`
- op2 — `addSupportedERC20(UNI)`: `0xcd443da64d612cc5bc96157197e5ab62c5207d1d45d402be1d2e5170a5f35ae9`

## What it changes

Two privileged operations, scheduled in **one DAO Safe batch**, executed in order after 48h:

1. **`Oracle.updateUsdFeedInfo(UNI, 0x553303d460EE0afB37EdFf9bE42922D8FF63220e, 4200)`** —
   registers the Chainlink **UNI / USD** feed. `Oracle.isTokenSupported(UNI)` → `true`,
   `getUsdPrice(UNI)` returns a fresh price, `convert(UNI ⇄ ETH)` becomes available.
2. **`StrategyManager.addSupportedERC20(UNI)`** — adds UNI to `supportedERC20()` so a strategy
   that holds UNI is counted in `totalNAVInETH()` instead of silently falling out of NAV.

No user funds move, no contract is upgraded. Both calls are config setters on
already-deployed, already-verified contracts.

## Why

Requested by **Arseny** (2026-09-18), in the same thread that picked the UNI/WETH 0.3% pool as a
strategy candidate. Complements [011](../011-strategy-manager-add-unicl-weth-usdt-strategy/) by
making the second token leg of a UniCL strategy priceable.

- **UNI is the next strategy token.** The UniCL strategies registered in 009/011 hold their
  paired token (USDC / USDT) as working capital and pay withdrawals in it; a UniCL **UNI/WETH**
  instance needs UNI to be a *priceable, supported* token before it can be registered
  (see [015](../015-strategy-manager-add-unicl-uni-weth-strategy/), withdrawn partly for this
  reason).
- **The pool that motivates it is deep.** Live DeFiLlama read, 2026-09-18:

  | UNI-WETH pool | TVL | `apyBase` (1d) | `apyBase7d` | vol 1d | vol 7d |
  |---|---|---|---|---|---|
  | **0.3%** (`0x1d42064F…8d60`) | **$16,530,502** | **108.25%** | **151.81%** | **$16,341,383** | $54,848,334 |
  | 1% | $5,384,048 | 83.20% | 621.12% | $1,227,297 | $1,230,377 |
  | 0.05% | $552,691 | 33.30% | 130.50% | $1,008,514 | $10,109,626 |

  The 0.3% pool is the one a `UniCLStrat` would target: `factory.getPool(UNI, WETH, 3000)` =
  `0x1d42064Fc4Beb5F8aAF85F4617AE8b3b5B8Bd801`, `token0 = UNI`, `token1 = WETH`,
  `tickSpacing = 60`, `observationCardinality` 102 (TWAP-ready), and the
  `UniswapV3ConverterAdapter` already allowlists it — **no new adapter work is required.**

### Why staleness = 4200 s

Measured, not copied: 600 consecutive Chainlink rounds on the UNI/USD aggregator
`0xdEf8C51d7c1040637A198efFc39613865B32EA51` (13.00 days of history):

| gap min | median | mean | **max** | gaps > 3600 s | gaps > 3900 s | gaps > 4200 s |
|---|---|---|---|---|---|---|
| 12 s | 1596 s | 1874.5 s | **3660 s** | 132 / 599 | **0** | **0** |

The feed is heartbeat-bound at ~1h with a worst observed gap of **3660 s**. 4200 s leaves a
**540 s (14.7%) cushion** over the worst case seen in two weeks, and matches
[008](../008-oracle-staleness-margin/), which widened WETH (also a 3600 s heartbeat) 3600 → 4200
for the same jitter. Feed metadata verified on mainnet: `description() = "UNI / USD"`,
`decimals() = 8`, proxy `0x5533…220e` → aggregator `0xdEf8C51d…`.

## Dependency on op1 (same batch)

`addSupportedERC20` reverts `StrategyManagerERC20NotPriceable` (`0x0eb217c5`) until the Oracle
can price the token, so op2 carries **op1's id as `predecessor`**: `execute(op2)` reverts
`TimelockUnexecutedPredecessor` (`0x90a9a618`) until op1 is `Done`.

```
DAO Safe ── schedule ──▶ op1 (Oracle feed,         predecessor = 0x00…00)
DAO Safe ── schedule ──▶ op2 (addSupportedERC20,   predecessor = op1)      [same batch, nonce 22]
T+48h    ── execute  ──▶ op1, then op2 (permissionless, EXECUTOR_ROLE = address(0))
```

Alternative: `predecessor = 0x00…00` on op2 and sequence the two executes by hand — op2's
operation id changes if you do. The coupling was kept because it makes the wrong order
impossible rather than merely reverting.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` ×2 (op1, op2), `delay = 172800` — wrapped by `MultiSendCallOnly` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` ×2 — **op1 first**, then op2 |

`01-schedule-raw.json` is the same batch as raw `schedule` calldata (selector `0x01d5062a`).

### Parameters

| Field | op1 | op2 |
|---|---|---|
| `target` | Oracle `0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26` | StrategyManager `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` |
| `value` | `0` | `0` |
| `data` | `updateUsdFeedInfo(UNI, 0x5533…220e, 4200)` = `0x8eff1c3c…00001068` | `addSupportedERC20(UNI)` = `0xd73acee5…4201f984` |
| `predecessor` | `0x00…00` | `0xeca68714…06d49bbd` (op1) |
| `salt` | `0x97e0462f…9e503428` = `keccak256("everstrat/oracle/usd-feed/uni/2026-09-18")` | `0xd17cce2b…6441d2ac5b` = `keccak256("everstrat/strategy-manager/supported-erc20/uni/2026-09-18")` |
| `delay` | `172800` | `172800` |

UNI = `0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984`. Feed = Chainlink UNI/USD
`0x553303d460EE0afB37EdFf9bE42922D8FF63220e`.

## Verification performed

Mainnet forks (anvil, `ethereum-rpc.publicnode.com`), 2026-09-18. The 48h warp necessarily
staleness-breaks a 1h-heartbeat feed, so timelock mechanics and pricing were probed on separate
forks (the Timelock impersonated in the second).

**State before** (block 26,006,157): `Oracle.isTokenSupported(UNI) = false`,
`SM.isSupportedERC20(UNI) = false`, SM UNI balance `0`,
`SM.totalNAVInETH() = 436737449545944677`, `Registry.hasRole(ADMIN_ROLE, timelock) = true`.

| Step | Result |
|---|---|
| direct `updateUsdFeedInfo` from a non-admin | revert `RegistryClientMissingRole` `0x4d616cff` |
| `addSupportedERC20(UNI)` with no feed (timelock impersonated) | revert `StrategyManagerERC20NotPriceable` `0x0eb217c5` |
| DAO Safe `schedule` ×2 | OK, gas 57,264 / 56,591; both `1 (Waiting)` |
| `execute` op1 before the delay | revert `TimelockUnexpectedOperationState` `0x5ead8eb5` |
| `execute` op2 before op1 is done | revert `TimelockUnexecutedPredecessor` `0x90a9a618` |
| `execute` op1, op2 after 48h + 1s, unrelated EOA | OK, gas 157,422 / 124,677; both `3 (Done)` |
| re-`execute` / re-`schedule` | revert `0x5ead8eb5` |
| re-`addSupportedERC20(UNI)` | revert `StrategyManagerERC20AlreadySupported` `0x04a77d9d` |
| `addSupportedERC20(DAI)` (code, no feed) | revert `0x0eb217c5` |
| `removeSupportedERC20(UNI)` from the Security Safe | OK, gas 45,213, no timelock |

**Effects after** — `Oracle.isTokenSupported(UNI) = true`;
`getUsdFeedInfo(UNI) = (0x553303d460EE0afB37EdFf9bE42922D8FF63220e, 4200)`;
`Oracle.getSupportedTokens() = [address(0), USDC, WETH, USDT, UNI]`;
`SM.supportedERC20() = [USDC, USDT, UNI]`; `SM.paused() = false`; **`totalNAVInETH()` unchanged**
(`436737449545944677`) because the StrategyManager holds no UNI.

**Pricing** (fresh feed, no warp) — `getUsdPrice(UNI) = 8.910119e18`,
`getUsdPrice(ETH) = 2598.67918e18`; `convert(1 UNI → ETH) = 3428710718030402` wei,
`convert(1 ETH → UNI) = 291654817871145058850` wei (8.910119 / 2598.67918 = 0.0034287);
`getUsdPriceWithStalenessCheck(UNI, 4200)` returns the price while `…(UNI, 100)` reverts
`0xa9f73445` — the bound is enforced, not advisory.

**Safe payload end to end** (third fork, block 26,006,240): the packed `MultiSendCallOnly`
payload decoded back to 2 entries (both `to = Timelock`, `operation = 0`, `value = 0`) equal to
`01-schedule-raw.json` byte-for-byte; with the Safe's owner set patched in fork storage only to
1-of-1, `execTransaction(MultiSendCallOnly, …, operation = 1)` succeeded (gas 126,126) against
safeTxHash `0xe396b3f0…191795` — the same hash later signed on mainnet.

## Risks

- **The feed can freeze NAV.** `_supportedERC20sNAVInETH` is fail-closed: zero balances skip the
  Oracle, but a non-zero UNI balance with a stale feed **reverts** `totalNAVInETH()` rather than
  mispricing. After the 48h warp both `totalNAVInETH()` and `getUsdPrice(UNI)` reverted
  `0xa9f73445` while `SM.paused()` stayed `false` — NAV frozen, not wrong. 4200 s is tight for a
  1h-heartbeat feed; the 540 s cushion is measured and should be monitored once UNI enters NAV.
- **Fee APR on the motivating pool is bursty.** 0.3% `apyBase` is 108%/1d vs 152%/7d, while the
  1% pool printed 621% over 7d and 83% over 1d in the same snapshot. 1d volume is ~99% of pool TVL.
- **IL / exposure.** DeFiLlama flags the pool `ilRisk = yes`, `exposure = multi`. A UNI/WETH
  position is a volatile-pair LP, not a stable leg.
- **Removal is instant.** `removeSupportedERC20(UNI)` via `SECURITY_ROLE` (Security Safe
  `0x7c128C1C…9846`) flips `isSupportedERC20(UNI)` to `false` with **no timelock**.
- **Adding support alone does not allocate capital.** No strategy holding UNI is registered by
  this proposal.

## On-chain schedule (mainnet)

| Field | Value |
|---|---|
| Proposed by | off-chain Safe delegate `0x1483E048a76A93a3A59bBfA6d60471eA4990e922`; `proposer` recorded as its delegator `0x4A2D30c7b9f7907D580f9A1902D42dd78B21F0d2`; submitted 2026-09-18 19:05:31 UTC |
| Scheduled via | DAO Safe `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9`, nonce 22 (batch of 2 `schedule` ops via `MultiSendCallOnly`) |
| safeTxHash | `0xe396b3f0217ee7d8496397ba9c8e91f0e070d6a495d82ca426af264140191795` |
| Signatures | 3 of 4 — `0x1Efb…9a46` 2026-09-18 21:00:03 · `0xF412…E149` 23:22:02 · `0xe9BE…dc4a` 2026-09-19 07:18:11 UTC (executor) |
| Execute (Safe) | 2026-09-19 07:18:11 UTC — tx `0x4b36e80469d8fe3f7d6cf71b34d82d3ab8ece17961365ec2e03ecb000d02afc7`, block 26010017, gas 136,454 |
| Events | `CallScheduled` + `CallSalt` for op1 and op2; `ExecutionSuccess(0xe396b3f0…191795)` |
| Calldata check | the two `MultiSend` entries equal `01-schedule-raw.json` byte-for-byte; `hashOperation` reproduces both op ids |
| Ready at | `getTimestamp` = `1789975091` = **2026-09-21 07:18:11 UTC** (both ops) |
| Execute (permissionless) | `02-execute.json` — op1, then op2 |

## On-chain execution (mainnet)

Executed on 2026-09-22, one call per operation, op1 first, both from `0x046E01eE…a899D7`
(permissionless — `EXECUTOR_ROLE` is `address(0)`). The calldata of both transactions equals
`02-execute.json` byte-for-byte.

| Operation | Transaction | Block | Executed (UTC) | Gas | Events |
|---|---|---|---|---|---|
| op1 feed `0xeca68714…06d49bbd` | `0x2c158399c99f1820b2bebe4101f0fdde529a6a3cc9e27a8ae239be66d7304f72` | 26036243 | 2026-09-22 23:15:47 | 157,422 | `UsdFeedAdded`, `CallExecuted` |
| op2 supported ERC-20 `0xcd443da6…a5f35ae9` | `0x25d9e669ffdb784be09560ce0b6e94e5ab14d6f24f415c3206bcea193736c39a` | 26036248 | 2026-09-22 23:16:47 | 124,677 | `SupportedERC20Added(UNI)`, `CallExecuted` |

Gas matches the fork simulation (157,422 / 124,677) exactly. Both ops `getOperationState == 3`
(Done).

**Getters after execution** (block 26036274):

- `Oracle.isTokenSupported(UNI)` → `true`; `getUsdFeedInfo(UNI)` →
  `(0x553303d460EE0afB37EdFf9bE42922D8FF63220e, 4200)`
- `Oracle.getSupportedTokens()` → `[address(0), USDC, WETH, USDT, UNI]`
- `StrategyManager.isSupportedERC20(UNI)` → `true`; `supportedERC20()` → `[USDC, USDT, UNI]`
- `StrategyManager.totalNAVInETH()` → `1304420368675317954` (reads cleanly with UNI in the set)

This clears the UNI-oracle blocker cited when
[015](../015-strategy-manager-add-unicl-uni-weth-strategy/) was withdrawn.

## Cancelling

No longer possible — both operations are executed. Before execution, either the DAO Safe or the
Security Safe could `cancel(opId)`; cancelling op1 would have made op2 permanently unexecutable.
To reverse the effect now: `removeSupportedERC20(UNI)` (Security Safe, no delay) and
`updateUsdFeedInfo(UNI, …)` via the 48h `ADMIN_ROLE` path.
