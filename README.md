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

_As of 2026-09-21 16:30 UTC. Read live from `Timelock.getOperationState(opId)`
(`0` Unset · `1` Waiting · `2` Ready · `3` Done) and the Safe Transaction Service._

| Proposal | Where it stands |
|---|---|
| 001–012 | **Done** — every privileged action queued to date has executed on mainnet, state and effects re-verified on-chain |
| [013](013-add-uni-supported-token/) | ⏳ **Proposed** — pending in the Safe queue at **nonce 22** since 2026-09-18 19:05:31 UTC, awaiting **3-of-4** owner confirmations. Both op ids `Unset` (`0`) on mainnet until it executes |

**The whole queue has landed.** 008 (3 ops) and 009 (2 ops) executed 2026-09-15 between
13:24:11 and 13:43:47 UTC; **010 and 011 on 2026-09-16 at 11:44:47 and 11:46:59 UTC**; **012
closed it out on 2026-09-17 at 12:02:59 and 12:04:23 UTC**. All nine calls came from
`0x046E01eE…a899D7`: `EXECUTOR_ROLE` is `address(0)`, so `execute` is **permissionless** —
any address with gas can run a `Ready` operation, no owner action required. The 48h delay is a
**floor, not a schedule**: every operation of the last three days ran 20 minutes to 1 hour 40
after it became `Ready`. Per-proposal transactions, gas, and post-execution getter reads live in
each proposal's `## On-chain execution` section.

**012 wired up keeper automation.** The Mimic smart account `0x4115…8256` is now the only
allowed executor caller on both keeper executors (`executorCallerCount() == 1` on each); a live
`perform` from the Mimic clears the caller gate while every other address is rejected with
`KeeperExecutorUnauthorizedCaller`. This is the first governance action of the batch whose
visible effect is operational rather than a parameter change.

**013 has executed and opened UNI; two proposals now sit in the Safe behind it.**
The queued 013 Safe tx ran (on-chain Safe nonce is 23 ⇒ nonce 22 executed), which
scheduled both of its timelock operations; they are `Ready` (`getOperationState == 2`)
and `execute` is **permissionless** — the feed registration and `addSupportedERC20(UNI)`
take effect whenever someone with gas calls them, op1 (`updateUsdFeedInfo(UNI, 0x5533…220e, 4200)`)
first, then op2 (`addSupportedERC20(UNI)`, which reverts `StrategyManagerERC20NotPriceable`
without a feed). Both reverts, the NAV semantics, the pricing and the UNI/WETH 0.3% target
pool were reproduced on a mainnet fork — see [013](013-add-uni-supported-token/). Staleness
4200 s is measured against 600 Chainlink rounds / 13 days (max gap 3660 s, zero gaps above 3900 s).

Two new proposals are queued behind that:

- **[014](014-strategy-manager-performance-fee-bps/) at nonce 23** — turn the performance fee
  on: `setPerformanceFeeBps(1500)` (0 → 15%). 48h admin timelock, no predecessor.
- **[015](015-strategy-manager-add-unicl-uni-weth-strategy/) at nonce 24 — 🔴 WITHDRAWN
  2026-09-21.** Was to register the UNI/WETH 0.3% strategy Arseny deployed on 2026-09-21,
  gated on 013's op2 as predecessor. Withdrawn before a single signature: the strategy isn't
  final and UNI has no oracle feed yet. A **rejection tx** now occupies nonce 24
  (`0xe51719fe…abff734`) to consume it and kill the original.

The queue head is **nonce 23** (014). Nonce 24 holds the 015 withdrawal rejection, which
needs 3-of-4 to become final — until then the withdrawn 015 tx is merely unsigned, not
unexecutable.

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
| [012](012-keeper-executors-allow-mimic-caller/) | Keeper executors: `allowExecutorCaller(Mimic 0x4115…8256)` on both QueueKeeperExecutor and StrategyKeeperExecutor (batch, predecessor = 011) | **Executed** — 2026-09-17 12:02:59 / 12:04:23 UTC (tx `0x0441b1f8…cd788049` / `0x08760b79…775e0df3f`); `isExecutorCaller(0x4115…8256) == true` on both executors with `executorCallerCount() == 1`, and a live `perform` from the Mimic passes the caller gate | `0xa4c9f4d3…5f6e40` (queue), `0xb716df94…06b880b5` (strategy) |
| [013](013-add-uni-supported-token/) | Oracle + StrategyManager: register the Chainlink UNI/USD feed (`updateUsdFeedInfo(UNI, 0x5533…220e, 4200)`) **and** `addSupportedERC20(UNI)` (batch, predecessor = feed op) | ✅ **Scheduled** — the queued Safe tx executed; both operation ids are now `Ready` (`getOperationState == 2`) as of 2026-09-21 (on-chain Safe nonce is 23 ⇒ nonce 22 executed). The feed + token registration take effect when someone calls `execute` (permissionless) | `0xeca68714…6d49bbd` (feed), `0xcd443da6…5f35ae9` (supported ERC-20) |
| [014](014-strategy-manager-performance-fee-bps/) | StrategyManager: turn the performance fee on — `setPerformanceFeeBps(1500)` (0 → **15%**, rate set by Arseny 2026-09-21) | 🟡 **Proposed** — queued in the DAO Safe at **nonce 23**, `confirmations: []`, not executed; `safeTxHash = 0x4fba6636…dd0004`, submitted 2026-09-21T16:02:21Z by delegate `0x1483E048…0e922`. Stored bytes audited byte-for-byte against `01-schedule-raw.json`, and the service decodes it as `schedule` (target `0x94916a…b5C9`, `setPerformanceFeeBps(1500)`, predecessor `0x0`, salt `0x2115f78d…2598c0`, delay `172800`). Needs 3-of-4 owner confirmations. *(Filing was first blocked by `429 Monthly quota exceeded`; a `SAFE_API_KEY` raised the limit 5000 → 50,000.)* | `0xc4302e52…f74ea46` |
| [015](015-strategy-manager-add-unicl-uni-weth-strategy/) | StrategyManager: register the newly deployed UniCL UNI/WETH 0.3% strategy — `addStrategy(0x2c3AEFaC…0715d, 10, 10)`, **predecessor = 013's `addSupportedERC20(UNI)`** so it can only land once UNI is priceable | 🔴 **WITHDRAWN 2026-09-21 16:40 UTC** — filed at nonce 24 (`0x49019656…4c78f7c1f9`, never executed, `confirmations: []` — no owner ever signed), then withdrawn by Arseny: *the strategy is not completely ready and the UNI oracle was not enabled*. Nothing reached the timelock (`getOperationState == 0` Unset), so there was no scheduled op and no `cancel(bytes32)` to call. Cancellation was filed as a **Safe rejection at nonce 24** (Safe→Safe, `value: 0`, `0x`, `safeTxHash = 0xe51719fe…abff734`, submitted 16:40:52Z): 3-of-4 signatures consume the nonce and make the 015 tx permanently unexecutable. The hazard analysis below still stands as the argument against registering an unfundable strategy | `0x489a556a…2201cd07` (never scheduled) |

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
