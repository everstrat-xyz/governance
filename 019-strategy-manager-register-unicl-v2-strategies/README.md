# 019 — StrategyManager: register the four UniCL v2 strategies

**Status:** 📝 Draft — not yet submitted.
**Operation id:** `0xf558cda6b3831a4fb393278e368ff3b319c04696e37ebb034d31c463fc9ed6a6`
(one `scheduleBatch` operation — `hashOperationBatch(targets, values, payloads, 0x00…00, salt)`,
recomputed locally and reproduced by the mainnet timelock's own `hashOperationBatch`;
`getOperationState == 0` (Unset) at block 26154876).

## What it changes

One atomic timelock batch of four `StrategyManager.addStrategy` calls, registering the UniCL **v2**
build of each strategy currently live, with the same weights as the v1 strategy it replaces:

| # | New strategy (v2) | Pool | Weights (deposit / withdrawal) | Replaces (v1, see [020](../020-strategy-manager-drain-remove-unicl-v1-strategies/)) |
|---|---|---|---|---|
| 1 | `0xfbcC73fA67F50D38FB318991C6CAbD5A6f64014A` | USDC/WETH 0.3% `0x8ad5…e6D8` | 40 / 40 | `0x5E12FFD5e0F69d37B581D06b68ad6c53609cac38` |
| 2 | `0x63B68CC99991429CA8a808125e446990Ae7fe669` | USDC/WETH 0.01% `0xE055…939F` | 15 / 15 | `0x59C476c1817b23791d96F35b9f468E6d9A242252` |
| 3 | `0xE80bfd65A55855ead8Cb5f7040031426CB718Cc5` | WETH/USDT 0.3% `0x4e68…Fa36` | 45 / 45 | `0x3Fb6B9174427CA4FF38728398F4F4CB526F66501` |
| 4 | `0xFC888a499c4172e393A258FbC6cAd1475a7e4129` | UNI/WETH 0.3% `0x1d42…D801` | 10 / 10 | `0x956fE55DB50527E75c1fcA679adE509AD89cAFDd` ([017](../017-strategy-manager-add-unicl-uni-weth-strategy/)) |

Each `addStrategy` also grants the strategy `CONVERTER_CALLER_ROLE` on the Converter. After execution
`strategyCount()` goes 4 → **8** until [020](../020-strategy-manager-drain-remove-unicl-v1-strategies/)
removes the v1 set. This proposal moves no funds: the new strategies start empty and receive capital
through the keeper's normal `DepositExcess` path, by weight.

## Why

Requested by **Vadyusha** (2026-10-09) as the first half of a v1 → v2 strategy migration; 020 is
the second half. The v2 strategy code is contracts
[#53](https://github.com/everstrat-xyz/contracts/pull/53) (`157129b`): ratio-aware inventory swaps,
alt position from leftover, incremental liquidity (`996ab2a`); swaps and withdraw path sized on the
route rather than the strategy pool (`d714231`); near-full withdrawals unwind fully (`4e782d7`);
logic moved into the linked `UniCLStratLib` to fit EIP-170 (`8dac70a`). The withdrawal fixes are
visible on mainnet state: depending on prices at the time, a full withdrawal from the three original
v1 strategies left anywhere from 0 to ≈ 0.09% of their NAV behind (measured — see 020), while the v2
path unwinds completely.

**Why one batch.** A single `scheduleBatch` registers all four or none, and gives 020 one operation id
to name as its `predecessor`, so the v1 set can never be drained before the v2 set exists.

## No predecessor

Registering strategies has no on-chain dependency on another scheduled operation: `predecessor =
0x00…00`. (It does not depend on [018](../018-controller-upgrade-v1.1.0/); 020 does.)

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.scheduleBatch(...)` — 4 calls, `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.executeBatch(...)` after the delay |

`01-schedule-raw.json` is the same transaction as raw calldata (`scheduleBatch` selector
`0x8f2a0bb0`). Both encode to identical bytes. **Prefer the raw file** for a batch: the Transaction
Builder's parsing of array inputs (`address[]`, `bytes[]`) is less predictable than raw calldata.

### Parameters

| Field | Value |
|---|---|
| Timelock function | `scheduleBatch(address[],uint256[],bytes[],bytes32,bytes32,uint256)` (`0x8f2a0bb0`); execute with `executeBatch(address[],uint256[],bytes[],bytes32,bytes32)` (`0xe38335e5`) |
| `targets` | StrategyManager `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` × 4 |
| `values` | `0` × 4 |
| `payloads[0]` | `addStrategy(0xfbcC…014A, 40, 40)` = `0xdca0c48f000000000000000000000000fbcc73fa67f50d38fb318991c6cabd5a6f64014a00000000000000000000000000000000000000000000000000000000000000280000000000000000000000000000000000000000000000000000000000000028` |
| `payloads[1]` | `addStrategy(0x63B6…e669, 15, 15)` = `0xdca0c48f00000000000000000000000063b68cc99991429ca8a808125e446990ae7fe669000000000000000000000000000000000000000000000000000000000000000f000000000000000000000000000000000000000000000000000000000000000f` |
| `payloads[2]` | `addStrategy(0xE80b…8Cc5, 45, 45)` = `0xdca0c48f000000000000000000000000e80bfd65a55855ead8cb5f7040031426cb718cc5000000000000000000000000000000000000000000000000000000000000002d000000000000000000000000000000000000000000000000000000000000002d` |
| `payloads[3]` | `addStrategy(0xFC88…4129, 10, 10)` = `0xdca0c48f000000000000000000000000fc888a499c4172e393a258fbc6cad1475a7e4129000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000a` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0xf204b152986e83e94ffbbbc3a4d8f4fec04a75b44347272f1912b2b3499237fe` = `keccak256("everstrat/strategy-manager/add-strategies/unicl-v2-x4/2026-10-09")` |
| `delay` | `172800` (48h, the enforced minimum) |

## Pre-wiring check (new strategies)

**Deployment.** All from `0x046E01Ee…a899D7` on 2026-10-09: the linked library `UniCLStratLib` at
`0x302E23EEAaafB3322A9093BCf528Dd61a64D5614` via the deterministic CREATE2 factory
`0x4e59b448…b4956C` (salt `0x00…00`, tx `0xa8fc5ecef6c4de565719fe402036d572607d79927cc00fe88f8cb56dc82c58c3`,
block 26151909), then the four strategies by plain `CREATE`:

| Strategy | Deployment tx | Block | Blockscout |
|---|---|---|---|
| `0xfbcC…014A` | `0x532037b603fa54b0afa22370c4f575e1162d9e5e261c6ef62980e5475c2ff951` | 26151909 | verified (`UniCLStrat`) |
| `0x63B6…e669` | `0xe5d51c54b60e924015488767733e41ea0e3bd1333ce01a88d603884e73765d1f` | 26151931 | verified (`UniCLStrat`) |
| `0xE80b…8Cc5` | `0xad3376f30e98a463ebbe6e159b694f98c56880edb636ddabd816bf012c1f90a5` | 26151939 | **not verified** |
| `0xFC88…4129` | `0xa4708c6e6a815915d439ccfa668745371c455abbce9c7d15c63af523b020b513` | 26151945 | verified (`UniCLStrat`) |
| `UniCLStratLib` `0x302E…5614` | (above) | 26151909 | **not verified** |

**Bytecode.** `forge build` of `UniCLStrat` at contracts `origin/main @ 157129b` (repo profile, library
linked to `0x302E…5614`) gives **identical executable code** for all four strategies: 24,155 B each,
the only differing bytes after masking immutables being the 32-byte IPFS hash inside the CBOR metadata.
The library likewise matches except for its 20-byte own-address guard (expected for a deployed
library) and the same 32-byte metadata hash. So behaviour is reproduced from source, but not byte-for-byte: the metadata (sources or
settings as recorded at compile time) differed from this rebuild. Each strategy's code embeds the
library address `0x302E…5614`.

**Configuration** (read from chain) — each v2 strategy is a like-for-like copy of the v1 one it replaces:

| | USDC/WETH 0.3% | USDC/WETH 0.01% | WETH/USDT 0.3% | UNI/WETH 0.3% |
|---|---|---|---|---|
| `pool` / fee | `0x8ad5…e6D8` / 3000 | `0xE055…939F` / 100 | `0x4e68…Fa36` / 3000 | `0x1d42…D801` / 3000 |
| `pairedToken` | USDC | USDC | USDT | UNI |
| `maxTotalNAV` | 1000 ETH | 140 ETH | 4500 ETH | 250 ETH |
| `positionWidth` / `rebalanceTickThreshold` / `maxTickDeviation` | 30 / 900 / 100 | 1823 / 911 / 100 | 30 / 900 / 100 | 18 / 480 / 60 |
| `twapInterval` / `shortTwapInterval` | 1800 / 60 | 1800 / 60 | 1800 / 60 | 1800 / 60 |
| `swapSlippageBps` | 100 | 100 | 100 | 100 |
| `pairedTokenToWethPath` | USDC →500→ WETH | USDC →500→ WETH | USDT →500→ WETH | UNI →3000→ WETH |

All four: `registry() == 0x46AA…993D`, `swapAdapter == 0x0844…Cde5` (allowlisted, [005](../005-converter-allow-univ3-adapter/)),
`paused() == false`, `isHealthy() == true`, `navInETH() == 0`, `MIN_INVENTORY_SWAP_BPS == 10` (v2-only
constant), not registered. Runtime size 24,155 B — 421 B under EIP-170.

## Verification performed

Mainnet forks (anvil, `ethereum-rpc.publicnode.com`), 2026-10-09.

**Mechanics, exact `01-schedule-raw.json` bytes from the DAO Safe** (block 26154963):

| Step | Result |
|---|---|
| `addStrategy` directly from a role-less EOA | revert `RegistryClientMissingRole(ADMIN_ROLE)` `0x4d616cff` |
| DAO Safe → `scheduleBatch` | OK, gas **81,222**; 4 × `CallScheduled` + 1 × `CallSalt`; state `1 (Waiting)` |
| `executeBatch` before the delay | revert `TimelockUnexpectedOperationState` `0x5ead8eb5` |
| re-`scheduleBatch` | revert `0x5ead8eb5` |
| `evm_increaseTime 172801` | state `2 (Ready)` |
| `executeBatch` from a role-less EOA | OK, gas **823,913**; 4 × `CallExecuted`; state `3 (Done)`; `strategyCount() == 8` |
| re-`executeBatch` | revert `0x5ead8eb5` |

**Effects, the whole migration in order** (block 26154937; fork-only `updateDelay(0)` so no time
warp stales the Chainlink feeds — the delay is not part of the operation id, so the same ids are
exercised): after 019, each v2 strategy has `isStrategyRegistered == true`, weights as above and
`Converter.isCaller == true`. After 018 and 020 the keeper's `depositToStrategies(3.3566 ETH)` (gas
3,668,221) split the funds into the v2 set ≈ 40/15/45/10 (1.2185 / 0.4569 / 1.3710 / 0.3050 ETH),
all four `isHealthy() == true`; a follow-up keeper `withdrawFromStrategies(0.05 ETH)` (gas 4,249,825)
and `checkAndRebalanceStrategies()` (gas 516,846) succeeded. Full numbers in 020.

## Risks

- **Two contracts are unverified on Blockscout** — the WETH/USDT strategy `0xE80b…8Cc5` and the
  library `0x302E…5614`. Executable code is reproduced from `157129b` above; verify both on the
  explorer before collecting signatures so signers can read them.
- **Audit coverage of contracts #53 is not established by this record.** The v2 changes touch
  swap sizing, inventory and withdrawal logic. Start capped exactly as v1 (same `maxTotalNAV`).
- **Eight strategies are registered between 019 and 020.** Keeper loops over all of them (deposit /
  withdraw / rebalance), so their gas roughly doubles, and Mimic's gas limits leave 1.30–1.44×
  headroom ([016](../016-strategy-keeper-exit-settlement-funding/)). Deposits keep flowing into the
  v1 set too (its weights are unchanged) and are swept by 020. Execute 020 right after 019 to keep
  the window short; scheduling both in the same signing round makes them Ready together.
- Removal of a v2 strategy is `removeStrategy` / `forceRemoveStrategy` (`ADMIN_ROLE`, 48h); `pause()`
  on a strategy (ADMIN or SECURITY, instant) stops deposits and withdrawals for it.

## Cancelling

Before execution, either the DAO Safe or the Security Safe may call
`cancel(0xf558cda6b3831a4fb393278e368ff3b319c04696e37ebb034d31c463fc9ed6a6)` on the timelock.
Cancelling 019 makes all four 020 operations permanently unexecutable (their predecessor would never
be `Done`) — re-propose both together. After execution, remove a strategy with `removeStrategy`
(empty) or `forceRemoveStrategy` through a new 48h `ADMIN_ROLE` proposal.
