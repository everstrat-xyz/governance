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

## Decisions

| # | Decision | Status | Operation id |
|---|---|---|---|
| [001](001-amm-connector-weight-0.9/) | AMM `connectorWeight` 0.5 → 0.9 | **Executed** — on-chain as of 2026-09-03 (`connectorWeight() == 9e17`) | `0x8c5d418c…f5db2` |
| [002](002-eve-token-everstrat/) | EVE token: "Everything Strategy" → "Everstrat" (`registerContract(EVE, 0x8FE6…)`) | **Executed** — on-chain as of 2026-09-03 (`getContractByKey("EVE") == 0x8FE6…61EC`) | `0xc396f407d91a7b213bf7606eb9ef50cea044810daee3fc394bb6924c7de727f3` |
| [003](003-oracle-usd-feeds/) | Oracle: register USDC + WETH USD price feeds (`updateUsdFeedInfo` ×2, batch) | **Executed** — on-chain as of 2026-09-07 (`getUsdFeedInfo(USDC\|WETH)` return the feeds; `isTokenSupported` both true) | `0xd57312a1…8b7ed5a1` (USDC), `0xa123b843…6d1d1f06` (WETH) |
| [004](004-strategy-manager-supported-usdc/) | StrategyManager: `addSupportedERC20(USDC)` | **Executed** — on-chain as of 2026-09-08 (`isSupportedERC20(USDC) == true`) | `0x602fe659…093871cb` |
| [005](005-converter-allow-univ3-adapter/) | Converter: whitelist the Uniswap V3 adapter (`setAllowedAdapter(0x0844…Cde5, true)`) | **Executed** — on-chain as of 2026-09-08 (`isAdapterAllowed(0x0844…Cde5) == true`) | `0x40ea4797…fa388f3` |
| [006](006-oracle-usdt-feed/) | Oracle: register USDT USD price feed (`updateUsdFeedInfo(USDT, USDT/USD, 86400)`) | **Scheduled** — on-chain (`getOperationState == 1 Waiting`, ready-at `1789300487`) | `0x3aa9f59b…5fe21a9a` |
| [007](007-strategy-manager-supported-usdt/) | StrategyManager: `addSupportedERC20(USDT)` (predecessor = 006) | **Draft** — not yet submitted | `0xd98323ea…6a7f9274` |

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
