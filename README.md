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

The DAO Safe is **3-of-5**. It was 3-of-4 until 2026-10-01, when
`addOwnerWithThreshold(0xB183142d33a2641233b608f3648e81e6A2Ef257C, 3)` executed at Safe nonce 27
(tx `0x5460a2dfac6728a8f595e2e624d72c595b9ae1b23c87d855c6c976a76e55959f`, block 26100280, signed by
`0x4A2D…F0d2`, `0xF412…E149` and `0x1Efb…9a46`). That is a change to the Safe's own owner set, so it
does not go through the timelock and has no proposal directory. Owners:
`0xB183142d33a2641233b608f3648e81e6A2Ef257C`, `0xe9BEf44a516A70cA83857927A1021D976F52dc4a`,
`0x1Efbc65eF286BE72155e26a19551eC1f01129a46`, `0xF412F1A5d22f08FBD406D3B2B52e80336fa8E149`,
`0x4A2D30c7b9f7907D580f9A1902D42dd78B21F0d2`.

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

_As of 2026-10-09 12:54 UTC (block 26154982). Read live from `Timelock.getOperationState(opId)`
(`0` Unset · `1` Waiting · `2` Ready · `3` Done) and the Safe Transaction Service._

| Proposal | Where it stands |
|---|---|
| 001–014 | **Executed** — every operation `Done`, effects re-verified on-chain |
| [015](015-strategy-manager-add-unicl-uni-weth-strategy/) | **Withdrawn** — never scheduled; superseded by 017 |
| 016–017 | **Executed** — every operation `Done`, effects re-verified on-chain |
| [018](018-controller-upgrade-v1.1.0/) | **Scheduled, Waiting** — Controller upgrade to v1.1.0; ready 2026-10-11 12:07:59 UTC |
| [019](019-strategy-manager-register-unicl-v2-strategies/) | **Draft** — register the four UniCL v2 strategies |
| [020](020-strategy-manager-drain-remove-unicl-v1-strategies/) | **Draft** — drain and remove the four UniCL v1 strategies (after 018 and 019) |

**018 is scheduled.** The Controller upgrade to v1.1.0 (`withdrawFromStrategy` callable by
`ADMIN_ROLE`) was scheduled 2026-10-09 12:07:59 UTC at Safe nonce 28 and can execute from
2026-10-11 12:07:59 UTC. The DAO Safe holds no pending transactions (next nonce 29).

**019 and 020 are drafted: a v1 → v2 strategy migration.** 019 registers the UniCL v2 build of each
of the four live strategies, same pools, caps and weights, as one atomic batch. 020 then drains each
v1 strategy and removes it, one atomic batch per strategy, gated on 019 by `predecessor` and on 018
by the Controller permission. A v1 strategy paused while its 020 operation is pending must have that
operation cancelled, or the batch would force-remove it with its funds. On forks the whole migration, including re-depositing
into v2, cost −0.02% to −0.05% of NAV.

**Since the last sweep (2026-09-22):**

- **014 turned the performance fee on.** `performanceFeeBps() == 1500` (15%), paid to the DAO
  Safe as `daoTreasury()`. Executed 2026-09-26 11:39:47 UTC.
- **016 funded keeper exit settlement.** On the StrategyKeeperExecutor, `minWithdrawETH` went
  0.01 → 0.0001 ETH and `controllerReserveETH` went 0 → 0.05 ETH. Executed 2026-09-26 11:40:59 and
  11:42:11 UTC.
- **017 registered the re-deployed UniCL UNI/WETH 0.3% strategy** `0x956F…AfDd` with weights
  10/10 and no predecessor, since 013 had already made UNI priceable. It is the fourth registered
  strategy and already holds capital. Executed 2026-09-26 11:52:11 UTC. It supersedes the
  withdrawn 015. It was filed as "016" and renumbered here because the keeper-funding proposal
  had already taken that number. Its bytecode provenance question (+179 B over the repo's own
  build) is still open; see the README.
- **The DAO Safe went from 3-of-4 to 3-of-5.** See [Governance model](#governance-model).

All four timelock executions came from `0x046E01eE…a899D7`, the same permissionless executor that
ran 008–013.

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
| [014](014-strategy-manager-performance-fee-bps/) | StrategyManager: `setPerformanceFeeBps(1500)` (0 → 15%) | **Executed** — 2026-09-26 (`performanceFeeBps() == 1500`) | `0xc4302e52…7f74ea46` |
| [015](015-strategy-manager-add-unicl-uni-weth-strategy/) | StrategyManager: register the UniCL UNI/WETH 0.3% strategy (`addStrategy(0x2c3A…715d, 10, 10)`, predecessor = 013 op2) | **Withdrawn** — 2026-09-21, never scheduled (Safe nonce 24 rejected); superseded by 017 | `0x489a556a…2201cd07` (never scheduled) |
| [016](016-strategy-keeper-exit-settlement-funding/) | StrategyKeeperExecutor: `setMinWithdrawETH(1e14)` (0.01 → 0.0001 ETH) + `setControllerReserveETH(5e16)` (0 → 0.05 ETH) (batch) | **Executed** — 2026-09-26 (`minWithdrawETH() == 1e14`, `controllerReserveETH() == 5e16`) | `0xc9c3bc84…5e5f5725` (min withdraw), `0x826b1bdb…566a02d5` (reserve) |
| [017](017-strategy-manager-add-unicl-uni-weth-strategy/) | StrategyManager: register the re-deployed UniCL UNI/WETH 0.3% strategy (`addStrategy(0x956F…AfDd, 10, 10)`) | **Executed** — 2026-09-26 (`isStrategyRegistered(0x956F…AfDd) == true`) | `0x941ba602…7e035210` |
| [018](018-controller-upgrade-v1.1.0/) | Controller: UUPS upgrade to implementation `0xd4f4…d55D` (v1.0.0 → v1.1.0; `withdrawFromStrategy` callable by `ADMIN_ROLE`) | **Scheduled** — 2026-10-09; ready 2026-10-11 12:07:59 UTC | `0x8cc3710f…30b7c98a` |
| [019](019-strategy-manager-register-unicl-v2-strategies/) | StrategyManager: register the four UniCL v2 strategies (`addStrategy` ×4, one `scheduleBatch`, weights 40/15/45/10) | **Draft** — 2026-10-09, not yet submitted | `0xf558cda6…fc9ed6a6` |
| [020](020-strategy-manager-drain-remove-unicl-v1-strategies/) | Controller + StrategyManager: drain and force-remove the four UniCL v1 strategies (4 × `scheduleBatch` `[withdrawFromStrategy(max) ×2, forceRemoveStrategy]`, predecessor = 019) | **Draft** — 2026-10-09, not yet submitted | `0xfa9e716c…ed7493d5`, `0xe9438cb7…acef75c8`, `0xe680e969…b9265bd9`, `0x93ee86d5…b81f4cd2` |

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
