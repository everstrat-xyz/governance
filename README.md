# EverStrat Governance

Transaction payloads for every privileged action on the EverStrat mainnet protocol —
scheduled, executed, and cancelled — with the reasoning behind each one.

Nothing in this repository executes anything. It is a record: the exact calldata that
was (or will be) signed, what it changes, and how to verify it independently.

## Why this exists

Every privileged action on EverStrat routes through a 48-hour `TimelockController`.
The DAO Safe can only *propose*; execution is permissionless once the delay elapses.
That makes each change a two-transaction, two-day public process — and this repo is
the human-readable half of it, so anyone can check what was proposed before it lands.

## Governance model

| Role | Holder | Delay | Can do |
|---|---|---|---|
| `ADMIN_ROLE` | Admin timelock `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` | 48h | All configuration, upgrades, `unpause` |
| Timelock `PROPOSER` + `CANCELLER` | DAO Safe `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9` | — | Schedule and cancel operations only |
| Timelock `EXECUTOR` | `address(0)` — anyone | — | Execute after the delay |
| `SECURITY_ROLE` | Security Safe `0x7c128C1CF39822B4133F8E067F4bF14999c49846` | none | `pause`, emergency recovery, cancel |

The DAO holds **no** protocol role directly. It cannot call the protocol; it can only
queue a call for the timelock to make.

## Mainnet addresses

| Contract | Address |
|---|---|
| Registry | `0x46AA1bd55c19be90d8767e0C22732A7DD31D993D` |
| Admin timelock (48h) | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` |
| AMM | `0x42c618D9457BE7cb3b836F7FD3332E2800C48a8b` |
| EVE | `0x8FE6A43672fCa70d41e112B8426387665fD061EC` (was `0x3A373227AECE982F07B9D14711a05d7a879859D4` until [002](002-eve-token-everstrat/)) |
| Controller | `0x9097868dcbda5a630729015DE1ddeCd66d74052c` |
| StrategyManager | `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` |
| Oracle | `0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26` |
| ExitQueue | `0x70E284502e99e150Fde1C07B4E8171ffc365423c` |
| Converter | `0xc8700441A8ca74aE5390F137bf3Dd659CaD570de` |
| Whitelist | `0x225B517a68A869020D99F51fFf0Ab43D47Ce3B48` |
| QueueKeeperExecutor | `0xb7D76E4334e9E23B6edFF77e0C05B07E938a090B` |
| StrategyKeeperExecutor | `0xE94F714fbBF85c421b6FB80D3Ce0DFA20110EA29` |

All verified on Etherscan.

## Status at a glance

_As of 2026-09-22 23:22 UTC (block 26036274). Read live from `Timelock.getOperationState(opId)`
(`0` Unset · `1` Waiting · `2` Ready · `3` Done) and the Safe Transaction Service._

| Proposal | Where it stands |
|---|---|
| 001–013 | **Executed** — every operation `Done`, effects re-verified on-chain |
| [014](014-strategy-manager-performance-fee-bps/) | **Scheduled, Waiting** — ready 2026-09-24 13:32:59 UTC |
| [015](015-strategy-manager-add-unicl-uni-weth-strategy/) | **Withdrawn** — never scheduled; its Safe nonce was consumed by a rejection |
| [016](016-strategy-keeper-exit-settlement-funding/) | **Proposed** — DAO Safe nonce 25, 1 of 3 confirmations |

**013 opened UNI.** Both operations executed 2026-09-22 at 23:15:47 and 23:16:47 UTC, about 40h
after they became `Ready`. The callers were permissionless (`0x046E01eE…a899D7`, the same address
that ran 008–012). The Oracle now prices UNI (`0x5533…220e`, staleness 4200), and
`StrategyManager.supportedERC20()` is `[USDC, USDT, UNI]`. The 48h delay is a floor, not a
schedule: nothing executes until someone sends the transaction.

**014 turns the performance fee on** (`setPerformanceFeeBps(1500)`, 0 → 15%), paid to the DAO
Safe as `daoTreasury()`. It was scheduled 2026-09-22 13:32:59 UTC and can execute from
2026-09-24 13:32:59 UTC.

**015 was withdrawn before anyone signed it.** It would have registered the UniCL UNI/WETH
0.3% strategy. The proposer pulled it because the strategy wasn't final and UNI had no Oracle
feed yet. 013 has since fixed the second point. A same-nonce Safe rejection executed 2026-09-22 13:39:11 UTC, so the
filed transaction can never run. The record keeps the hazard analysis and an open bytecode
provenance question for a future re-proposal.

**016 is queued in the DAO Safe for keeper exit funding** (nonce 25, submitted 2026-09-23). It lowers the StrategyKeeperExecutor's `minWithdrawETH`
from 0.01 to 0.0001 ETH, so small priced-exit shortfalls get pulled from the strategies instead of
running out the 3-day window. It also sets `controllerReserveETH` to 0.05 ETH, which stays on the
Controller to settle exits without touching the LP positions. `exitLiquidityTargetETH` stays 0.

The DAO Safe holds **one pending transaction**: 016 at nonce 25 (`safeTxHash 0x3fe47bc2…fdd8f637`, 1 of 3
confirmations; stored bytes match `01-schedule-raw.json`).

## Decisions

| # | Decision | Status | Operation id |
|---|---|---|---|
| [001](001-amm-connector-weight-0.9/) | AMM `connectorWeight` 0.5 → 0.9 | **Executed** — 2026-09-03 (`connectorWeight() == 9e17`) | `0x8c5d418c…b97f5db2` |
| [002](002-eve-token-everstrat/) | EVE token: "Everything Strategy" → "Everstrat" (`registerContract(EVE, 0x8FE6…)`) | **Executed** — 2026-09-03 (`getContractByKey("EVE") == 0x8FE6…61EC`) | `0xc396f407…7de727f3` |
| [003](003-oracle-usd-feeds/) | Oracle: register USDC + WETH USD price feeds (`updateUsdFeedInfo` ×2, batch) | **Executed** — 2026-09-07 (`isTokenSupported` true for both) | `0xd57312a1…8b7ed5a1` (USDC), `0xa123b843…6d1d1f06` (WETH) |
| [004](004-strategy-manager-supported-usdc/) | StrategyManager: `addSupportedERC20(USDC)` | **Executed** — 2026-09-08 (`isSupportedERC20(USDC) == true`) | `0x602fe659…093871cb` |
| [005](005-converter-allow-univ3-adapter/) | Converter: whitelist the Uniswap V3 adapter (`setAllowedAdapter(0x0844…Cde5, true)`) | **Executed** — 2026-09-08 (`isAdapterAllowed(0x0844…Cde5) == true`) | `0x40ea4797…cfa388f3` |
| [006](006-oracle-usdt-feed/) | Oracle: register USDT USD price feed (`updateUsdFeedInfo(USDT, USDT/USD, 86400)`) | **Executed** — 2026-09-13 (`isTokenSupported(USDT) == true`) | `0x3aa9f59b…5fe21a9a` |
| [007](007-strategy-manager-supported-usdt/) | StrategyManager: `addSupportedERC20(USDT)` | **Executed** — 2026-09-13 (`isSupportedERC20(USDT) == true`) | `0xd98323ea…6a7f9274` |
| [008](008-oracle-staleness-margin/) | Oracle: widen WETH/`address(0)` staleness 3600 → 4200 and USDC 82800 → 84600 (jitter margin, ×3) | **Executed** — 2026-09-15 (`getUsdFeedInfo` staleness 4200 / 4200 / 84600) | `0xc6360fe9…8abaada9` (WETH), `0x209d4394…ad272166` (`address(0)`), `0x74253eb4…840d93fe` (USDC) |
| [009](009-strategy-manager-add-unicl-strategies/) | StrategyManager: register the two UniCL USDC/WETH strategies (`addStrategy` ×2, batch) | **Executed** — 2026-09-15 (`isStrategyRegistered` true for both) | `0x345dfba9…a5e7eca8` (0.3% pool), `0xf340bad1…8ea50990` (0.01% pool) |
| [010](010-oracle-usdt-staleness-margin/) | Oracle: widen USDT staleness 86400 → 88200 (jitter margin) | **Executed** — 2026-09-16 (`getUsdFeedInfo(USDT)` staleness 88200) | `0xa2e19cb9…17ab4c6e` |
| [011](011-strategy-manager-add-unicl-weth-usdt-strategy/) | StrategyManager: register the UniCL WETH/USDT 0.3% strategy (`addStrategy`) | **Executed** — 2026-09-16 (`isStrategyRegistered(0x3Fb6…6501) == true`) | `0x4b6a7a40…6fcd9fd6` |
| [012](012-keeper-executors-allow-mimic-caller/) | Keeper executors: `allowExecutorCaller(Mimic 0x4115…8256)` on QueueKeeperExecutor + StrategyKeeperExecutor (batch, predecessor = 011) | **Executed** — 2026-09-17 (`isExecutorCaller(0x4115…8256) == true`, `executorCallerCount() == 1` on both) | `0xa4c9f4d3…505f6e40` (queue), `0xb716df94…06b880b5` (strategy) |
| [013](013-add-uni-supported-token/) | Oracle + StrategyManager: register the UNI/USD feed (`updateUsdFeedInfo(UNI, 0x5533…220e, 4200)`) and `addSupportedERC20(UNI)` (batch, predecessor = feed op) | **Executed** — 2026-09-22 (`isSupportedERC20(UNI) == true`) | `0xeca68714…06d49bbd` (feed), `0xcd443da6…a5f35ae9` (supported ERC-20) |
| [014](014-strategy-manager-performance-fee-bps/) | StrategyManager: `setPerformanceFeeBps(1500)` (0 → 15%) | **Scheduled** — 2026-09-22; ready 2026-09-24 13:32:59 UTC | `0xc4302e52…7f74ea46` |
| [015](015-strategy-manager-add-unicl-uni-weth-strategy/) | StrategyManager: register the UniCL UNI/WETH 0.3% strategy (`addStrategy(0x2c3A…715d, 10, 10)`, predecessor = 013 op2) | **Withdrawn** — 2026-09-21, never scheduled (Safe nonce 24 rejected) | `0x489a556a…2201cd07` (never scheduled) |
| [016](016-strategy-keeper-exit-settlement-funding/) | StrategyKeeperExecutor: `setMinWithdrawETH(1e14)` (0.01 → 0.0001 ETH) + `setControllerReserveETH(5e16)` (0 → 0.05 ETH) (batch) | **Proposed** — 2026-09-23, DAO Safe nonce 25, 1 of 3 confirmations | `0xc9c3bc84…5e5f5725` (min withdraw), `0x826b1bdb…566a02d5` (reserve) |

## Layout

Each decision is a numbered directory containing a `README.md` explaining what and why,
plus Safe Transaction Builder JSON files numbered in execution order:

```
NNN-short-name/
  README.md          what it changes, why, and how to verify
  01-schedule.json   DAO Safe -> timelock.schedule
  02-execute.json    anyone -> timelock.execute, after the delay
```

Some directories also carry `01-schedule-raw.json`, a raw-calldata variant for when the
Safe UI cannot resolve the target ABI.

## Verifying a payload yourself

Recompute the operation id from the fields shown in the Safe UI and compare it against
the one recorded in the decision's README. A single hash covers target, value, data,
predecessor and salt at once:

```sh
cast call <timelock> \
  "hashOperation(address,uint256,bytes,bytes32,bytes32)(bytes32)" \
  <target> <value> <data> <predecessor> <salt> \
  --rpc-url <mainnet-rpc>
```

Then decode the inner call to confirm what it actually does:

```sh
cast calldata-decode "setConnectorWeight(uint256)" <data>
```
