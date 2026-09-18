# 013 — Add UNI as a supported ERC-20 (UNI/USD Oracle feed + `StrategyManager`)

**Status:** 📝 **Draft — not scheduled.** Both operation ids are `Unset` (`0`) on mainnet;
the DAO Safe's next free nonce is **22** and it holds no pending transactions.
Nothing in this directory has been signed.

**Operation ids (verify with `Timelock.getOperationState`):**

| # | Operation | id |
|---|---|---|
| op1 | `Oracle.updateUsdFeedInfo(UNI, 0x5533…220e, 4200)` | `0xeca687148460f11ee30984d2685c68a72a434233ff9bf2b984760bb606d49bbd` |
| op2 | `StrategyManager.addSupportedERC20(UNI)` | `0xcd443da64d612cc5bc96157197e5ab62c5207d1d45d402be1d2e5170a5f35ae9` |

Requested by **Arseny** (2026-09-18), in the same thread that picked the UNI/WETH 0.3% pool
as a strategy candidate. Complements [011](../011-strategy-manager-add-unicl-weth-usdt-strategy/)
by making the second token leg of a UniCL strategy priceable.

---

## What it changes

Two privileged operations, scheduled in **one Safe batch**, executing in order after 48h:

1. **`Oracle.updateUsdFeedInfo(UNI, 0x553303d460EE0afB37EdFf9bE42922D8FF63220e, 4200)`**
   Registers the Chainlink **UNI / USD** feed. `Oracle.isTokenSupported(UNI)` → `true`,
   `getUsdPrice(UNI)` returns a fresh price, `convert(UNI ⇄ ETH)` becomes available.
2. **`StrategyManager.addSupportedERC20(UNI)`**
   Adds UNI to `supportedERC20()` so a strategy that holds UNI is counted correctly in
   `totalNAVInETH()` instead of silently falling out of NAV.

`_registry`-wide effects: none. No user funds move. No contract is upgraded. Both calls are
config setters on already-deployed, already-verified contracts.

## Why

- **UNI is the next strategy token.** The UniCL strategies registered in 009/011 hold their
  paired token (USDC / USDT) as working capital and pay withdrawals in it; a UniCL **UNI/WETH**
  instance needs UNI to be a *priceable, supported* token before it can be registered.
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
- **Ordering is forced, not stylistic.** `addSupportedERC20` reverts with
  `StrategyManagerERC20NotPriceable` until the Oracle can price the token. Verified on a fork
  (below), so the feed op must land first — hence the predecessor link.

## Dependency / ordering

```
DAO Safe ── schedule ──▶ op1 (Oracle feed, predecessor = 0x00)
DAO Safe ── schedule ──▶ op2 (addSupportedERC20, predecessor = op1)   [same batch, same nonce]
T+48h    ── execute  ──▶ op1 … then op2 (permissionless, EXECUTOR_ROLE = address(0))
```

Three reverts enforce this, all reproduced on a mainnet fork:

| Attempt | Revert | Selector |
|---|---|---|
| `execute` op1 before the delay | `TimelockUnexpectedOperationState(op1, 1)` | `0x5ead8eb5` |
| `execute` op2 before op1 is done | `TimelockUnexecutedPredecessor(op1)` | `0x90a9a618` |
| `addSupportedERC20(UNI)` with no feed | `StrategyManagerERC20NotPriceable(UNI)` | `0x0eb217c5` |
| re-`addSupportedERC20(UNI)` | already-supported | `0x04a77d9d` |
| re-`execute` / re-`schedule` | `TimelockUnexpectedOperationState` | `0x5ead8eb5` |

## Transactions

| # | Target | Function | Calldata |
|---|---|---|---|
| 1 | Oracle `0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26` | `updateUsdFeedInfo(address,address,uint256)` | feed `0x553303d460EE0afB37EdFf9bE42922D8FF63220e`, staleness **4200** |
| 2 | StrategyManager `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` | `addSupportedERC20(address)` | UNI `0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984` |

- `01-schedule.json` — Safe Transaction Builder, 2 transactions (op1 then op2), delay `172800`.
- `01-schedule-raw.json` — same two calls as raw `Timelock.schedule` calldata.
- `02-execute.json` — raw `Timelock.execute` calldata, op1 first.

Salts are fixed so the operation ids above are reproducible:
`salt1 = 0x97e0462f…9e503428`, `salt2 = 0xd17cce2b…6441d2ac5b`.

## Why staleness = 4200 s

Measured empirically, not copied: 600 consecutive Chainlink rounds on the UNI/USD
aggregator `0xdEf8C51d7c1040637A198efFc39613865B32EA51` (13.00 days of history, read via
batched JSON-RPC on `ethereum-rpc.publicnode.com`):

| gap min | median | mean | **max** | gaps > 3600s | gaps > 3900s | gaps > 4200s |
|---|---|---|---|---|---|---|
| 12 s | 1596 s | 1874.5 s | **3660 s** | 132 / 599 | **0** | **0** |

132 of 599 gaps sit in the 3600–3899 s bucket — the feed is heartbeat-bound at ~1h, with a
worst observed gap of **3660 s**. 4200 s therefore leaves a **540 s (14.7%) cushion** over the
worst case seen in two weeks, and matches the convention from
[008](../008-oracle-staleness-margin/), which widened WETH (also a 3600 s heartbeat) 3600 → 4200
for exactly this jitter. Feed metadata verified on mainnet: `description() = "UNI / USD"`,
`decimals() = 8`, proxy `0x5533…220e` → aggregator `0xdEf8C51d…`, owner
`0x21f73d42eb58ba49ddb685dc29d3bf5c0f0373ca`.

## Verification performed (mainnet fork, 2026-09-18)

Two `anvil` forks of Ethereum mainnet against `https://ethereum-rpc.publicnode.com`; scripts
`/tmp/sim013.sh` (timelock semantics, with the 48h warp) and `/tmp/sim013b.sh` (pricing and
preconditions, no warp — the 48h warp necessarily staleness-breaks a 1h-heartbeat feed, so the
two questions were probed on two forks, with the Timelock impersonated in the second).

**State before** (block 26,006,157): `Oracle.isTokenSupported(UNI) = false`,
`SM.isSupportedERC20(UNI) = false`, `SM` UNI balance `0`,
`SM.totalNAVInETH() = 436737449545944677`, `Registry.hasRole(ADMIN_ROLE, timelock) = true`.

**Timelock mechanics** — both ops scheduled in one batch (gas 57,264 / 56,591), state `1 (Waiting)`,
`readyAt = 1789928670`; pre-delay execute reverts `0x5ead8eb5`; after `warp 48h + 1s` both go
`2 (Ready)`; executed from an **unrelated EOA** (op1 gas 157,422 @ block 26,006,161, op2 gas
124,677 @ block 26,006,162) — confirming permissionless execution; both `3 (Done)`.

**Effects after** — `Oracle.isTokenSupported(UNI) = true`;
`getUsdFeedInfo(UNI) = (0x553303d460EE0afB37EdFf9bE42922D8FF63220e, 4200)`;
`Oracle.getSupportedTokens() = [address(0), USDC, WETH, USDT, UNI]`;
`SM.supportedERC20() = [USDC, USDT, UNI]`; `SM.paused() = false`; **`totalNAVInETH()` unchanged**
(`436737449545944677`) because the StrategyManager holds no UNI.

**Pricing** (fresh feed, no warp) — `getUsdPrice(UNI) = 8.910119e18` (feed answered `891011916` at
8 dp), `getUsdPrice(ETH) = 2598.67918e18`;
`convert(1 UNI → ETH) = 3428710718030402` wei, `convert(1 ETH → UNI) = 291654817871145058850` wei
(consistent: 8.910119 / 2598.67918 = 0.0034287);
`getUsdPriceWithStalenessCheck(UNI, 4200)` returns the price while `…(UNI, 100)` reverts
`0xa9f73445` — the margin argument is actually enforced, not advisory.

**Duplicate / invalid paths** — re-adding UNI reverts `0x04a77d9d`; adding a token with code but no
feed (DAI) reverts `0x0eb217c5`; a direct `updateUsdFeedInfo` from a non-admin reverts `0x4d616cff`
(Registry permission, admin role hash as the argument).

## Risks

- **The feed can freeze NAV.** This is the fail-closed design of `_supportedERC20sNAVInETH`
  (zero balances skip the Oracle entirely; a non-zero balance with a stale feed **reverts**
  `totalNAVInETH()` rather than mispricing). After the 48h warp both `totalNAVInETH()` and
  `getUsdPrice(UNI)` reverted `0xa9f73445` while `SM.paused()` stayed `false` — NAV would be frozen,
  not wrong. 4200 s is a tight-ish bound for a 1h-heartbeat feed; the 540 s cushion is measured, not
  assumed, and should be monitored after UNI actually enters NAV.
- **Fee APR on the motivating pool is bursty, not stable.** 0.3% `apyBase` is 108%/1d and 152%/7d,
  while the 1% pool printed 621% over 7d and 83% over 1d in the same snapshot — realised yield
  depends heavily on which fee tier, and on how much of the flow persists. 1d volume ($16.3M) is
  ~99% of pool TVL, i.e. ~1× daily turnover.
- **IL / exposure.** DeFiLlama flags the pool `ilRisk = yes`, `exposure = multi`, and its predictor
  reads `Down` at 99% confidence. A UNI/WETH position is a volatile-pair LP, not a stable leg.
- **Removal is instant.** `removeSupportedERC20(UNI)` via `SECURITY_ROLE` (Security Safe
  `0x7c128C1C…9846`) flips `isSupportedERC20(UNI)` to `false` with **no timelock** — verified, gas
  45,213. That is the escape hatch if the feed or the strategy misbehaves.
- **Adding support alone does not allocate capital.** No strategy holding UNI is registered by this
  proposal, and no UniCL strategy can use UNI until one is registered separately.

## Cancelling

The DAO Safe is `CANCELLER`. `Timelock.cancel(opId)` any time before execution; after execution it
reverts. Cancelling op1 makes op2 permanently unexecutable (its predecessor would never be done),
so a cancel of the feed op also kills the `addSupportedERC20` — both must be re-proposed together.

## How to verify independently

```bash
R=https://ethereum-rpc.publicnode.com
TL=0xF0911198Ef0a4b4234546fa5F50d6d1D45091774
ORACLE=0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26
SM=0x94916ab93C669E7c734f844dB019Ce9449a3b5C9
UNI=0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984

# state before scheduling (expect 0 = Unset)
cast call $TL "getOperationState(bytes32)(uint8)" 0xeca687148460f11ee30984d2685c68a72a434233ff9bf2b984760bb606d49bbd --rpc-url $R
cast call $TL "getOperationState(bytes32)(uint8)" 0xcd443da64d612cc5bc96157197e5ab62c5207d1d45d402be1d2e5170a5f35ae9 --rpc-url $R

# current state (expect false / false)
cast call $ORACLE "isTokenSupported(address)(bool)" $UNI --rpc-url $R
cast call $SM "isSupportedERC20(address)(bool)" $UNI --rpc-url $R

# recompute the op ids from the salts in 01-schedule.json
cast call $TL "hashOperation(address,uint256,bytes,bytes32,bytes32)(bytes32)" \
  $ORACLE 0 0x8eff1c3c0000000000000000000000001f9840a85d5af5bf1d1762f925bdaddc4201f984000000000000000000000000553303d460ee0afb37edff9be42922d8ff63220e0000000000000000000000000000000000000000000000000000000000001068 \
  0x0000000000000000000000000000000000000000000000000000000000000000 \
  0x97e0462f7a8786613c32e7059180c27fb633374a737c54c353f5a8729e503428 --rpc-url $R

# feed sanity
cast call 0x553303d460EE0afB37EdFf9bE42922D8FF63220e "description()(string)" --rpc-url $R
cast call 0x553303d460EE0afB37EdFf9bE42922D8FF63220e "decimals()(uint8)" --rpc-url $R
```
