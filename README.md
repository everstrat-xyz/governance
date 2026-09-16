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

_As of 2026-09-16 12:25 UTC. Read live from `Timelock.getOperationState(opId)`
(`0` Unset · `1` Waiting · `2` Ready · `3` Done) and the Safe Transaction Service._

| Proposal | Where it stands |
|---|---|
| 001–011 | **Done** — executed on mainnet, state and effects re-verified on-chain |
| **012** | **Scheduled** — DAO Safe nonce 21 executed 2026-09-15 11:42:47 UTC with 3/4 signatures; both ops `Waiting`, executable from **2026-09-17 11:42:47 UTC**. Its `predecessor` (011) is now `Done`, so that gate is cleared. |

**Everything queued so far has executed.** 008 (3 ops) and 009 (2 ops) landed on
2026-09-15 between 13:24:11 and 13:43:47 UTC; **010 and 011 landed on 2026-09-16 at
11:44:47 and 11:46:59 UTC**. All seven calls came from `0x046E01eE…a899D7`: `EXECUTOR_ROLE`
is `address(0)`, so `execute` is **permissionless** — any address with gas can run a `Ready`
operation, no owner action required. Per-proposal transactions, gas, and post-execution
getter reads live in each proposal's `## On-chain execution` section.

The DAO Safe holds **no pending transactions** (next nonce 22). The only operation still in
flight is **012**, whose first execution window opens 2026-09-17 11:42:47 UTC and needs
nothing but gas.

## Decisions

| # | Decision | Status | Operation id |
|---|---|---|---|
| [001](001-amm-connector-weight-0.9/) | AMM `connectorWeight` 0.5 → 0.9 | **Executed** — on-chain as of 2026-09-03 (`connectorWeight() == 9e17`) | `0x8c5d418c…f5db2` |
| [002](002-eve-token-everstrat/) | EVE token: "Everything Strategy" → "Everstrat" (`registerContract(EVE, 0x8FE6…)`) | **Executed** — on-chain as of 2026-09-03 (`getContractByKey("EVE") == 0x8FE6…61EC`) | `0xc396f407d91a7b213bf7606eb9ef50cea044810daee3fc394bb6924c7de727f3` |
| [003](003-oracle-usd-feeds/) | Oracle: register USDC + WETH USD price feeds (`updateUsdFeedInfo` ×2, batch) | **Executed** — on-chain as of 2026-09-07 (`getUsdFeedInfo(USDC\|WETH)` return the feeds; `isTokenSupported` both true) | `0xd57312a1…8b7ed5a1` (USDC), `0xa123b843…6d1d1f06` (WETH) |
| [004](004-strategy-manager-supported-usdc/) | StrategyManager: `addSupportedERC20(USDC)` | **Executed** — on-chain as of 2026-09-08 (`isSupportedERC20(USDC) == true`) | `0x602fe659…093871cb` |
| [005](005-converter-allow-univ3-adapter/) | Converter: whitelist the Uniswap V3 adapter (`setAllowedAdapter(0x0844…Cde5, true)`) | **Executed** — on-chain as of 2026-09-08 (`isAdapterAllowed(0x0844…Cde5) == true`) | `0x40ea4797…fa388f3` |
| [006](006-oracle-usdt-feed/) | Oracle: register USDT USD price feed (`updateUsdFeedInfo(USDT, USDT/USD, 86400)`) | **Executed** — on-chain as of 2026-09-13 (`isTokenSupported(USDT) == true`) | `0x3aa9f59b…5fe21a9a` |
| [007](007-strategy-manager-supported-usdt/) | StrategyManager: `addSupportedERC20(USDT)` | **Executed** — on-chain as of 2026-09-13 (`isSupportedERC20(USDT) == true`) | `0xd98323ea…6a7f9274` |
| [008](008-oracle-staleness-margin/) | Oracle: widen WETH/`address(0)` staleness 3600→4200 and USDC 82800→84600 (jitter margin, ×3) | **Executed** — 2026-09-15, three ops `Done` (13:24:11 / 13:35:59 / 13:37:11 UTC); on-chain `getUsdFeedInfo` now returns staleness **4200** (WETH & native ETH) and **84600** (USDC) | `0xc6360fe9…8abaada9` (WETH), `0x209d4394…5492ad272166` (`address(0)`), `0x74253eb4…8840d93fe` (USDC) |
| [009](009-strategy-manager-add-unicl-strategies/) | StrategyManager: register the two UniCL USDC/WETH strategies (`addStrategy` ×2, batch) | **Executed** — 2026-09-15, both ops `Done` (13:42:35 / 13:43:47 UTC); `isStrategyRegistered` is `true` for both strategies | `0x345dfba9…5e7eca8` (0.3% pool), `0xf340bad1…828ea50990` (0.01% pool) |
| [010](010-oracle-usdt-staleness-margin/) | Oracle: widen USDT staleness 86400→88200 (jitter margin) | **Executed** — 2026-09-16 11:44:47 UTC (tx `0xfa6d1d56…0e6023e3a`); `getUsdFeedInfo(USDT)` now returns staleness **88200** with the feed unchanged (`0x3E7d1eAB…e32D`) | `0xa2e19cb9…17ab4c6e` |
| [011](011-strategy-manager-add-unicl-weth-usdt-strategy/) | StrategyManager: register the UniCL WETH/USDT 0.3% strategy (`addStrategy`) | **Executed** — 2026-09-16 11:46:59 UTC (tx `0x1807f0d1…45c084b72e`); `isStrategyRegistered(0x3Fb6B917…6501)` is `true` | `0x4b6a7a40…96fcd9fd6` |
| [012](012-keeper-executors-allow-mimic-caller/) | Keeper executors: `allowExecutorCaller(Mimic 0x4115…8256)` on both QueueKeeperExecutor and StrategyKeeperExecutor (batch, predecessor = 011) | **Scheduled** — DAO Safe nonce 21 executed 2026-09-15 11:42:47 UTC, 3/4 signatures (tx `0x41e5a640…cc515211`); both ops `Waiting`, executable from **2026-09-17 11:42:47 UTC**; `predecessor` (011) is `Done` as of 2026-09-16 11:46:59 UTC, so that gate is cleared | `0xa4c9f4d3…5f6e40` (queue), `0xb716df94…06b880b5` (strategy) |

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
