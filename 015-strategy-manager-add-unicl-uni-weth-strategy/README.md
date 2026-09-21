# 015 — Register the UniCL UNI/WETH 0.3% strategy

> 🔴 **WITHDRAWN 2026-09-21 16:40 UTC — do not sign, do not execute.** No owner ever signed
> it, it never executed, and it never reached the timelock. Kept as a record of the hazard
> analysis; see [Status](#status) for the cancellation and the rejection tx.

Wires the newly deployed UNI/WETH 0.3% concentrated-liquidity strategy into
`StrategyManager` so it can receive deposits and be valued in protocol NAV.

| | |
|---|---|
| Target | `StrategyManager` `0x94916ab93C669E7c734f844dB019Ce9449a3b5C9` |
| Call | `addStrategy(address,uint8,uint8)` — `0xdca0c48f` |
| Arguments | `0x2c3AEFaCb41065810f43F23d0eBa8dF20350715d`, `10`, `10` |
| Delay | `172800` (48h) |
| Predecessor | `0xcd443da64d612cc5bc96157197e5ab62c5207d1d45d402be1d2e5170a5f35ae9` — **013 op2** |
| Salt | `0x04c997170365fd7c8ae41d6545e71cd8636caacda545cb6c146a8e7b81dc40b2` |
| Salt preimage | `keccak256("everstrat/strategy-manager/add-strategy/unicl-uni-weth-0.3pct/2026-09-21")` |
| Operation id | `0x489a556a585fbae6efeb04d3f1627d9168124c63a2b089e29b2e5ba42201cd07` |
| Inner calldata | `0xdca0c48f0000000000000000000000002c3aefacb41065810f43f23d0eba8df20350715d000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000a` |

The operation id above was recomputed from these exact fields with the on-chain
5-field `hashOperation(target, value, data, predecessor, salt)` — `delay` is **not**
part of it — and matches the live contract call.

## The strategy

Deployed by Arseny on 2026-09-21: `0x2c3AEFaCb41065810f43F23d0eBa8dF20350715d`
(tx `0x9481e0e2…5113d68`, 5.88M gas, status success). Read back from the contract:

| Param | Value |
|---|---|
| `pool` | `0x1d42064Fc4Beb5F8aAF85F4617AE8b3b5B8Bd801` — UNI/WETH, fee 3000, tickSpacing 60 |
| `pairedToken` | UNI `0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984` |
| `token0` / `token1` | UNI / WETH |
| `swapAdapter` | `0x0844580a121124CAEc6Cf4A933aac401813cCde5` (005) — `isAdapterAllowed` true |
| `registry` | `0x46AA1bd55c19be90d8767e0C22732A7DD31D993D` |
| `positionWidth` | 18 (±1080 ticks ≈ ±10.8%) |
| `rebalanceTickThreshold` | 480 |
| `maxTickDeviation` | 60 |
| `twapInterval` / `shortTwapInterval` | 1800 / 60 |
| `maxTotalNAV` | **250 ETH** (smallest of the four strategies) |
| `paused` | false |
| `totalDeposited` / `navInETH` | 0 / 0 — empty |
| `isStrategyRegistered` | **false** |

The pool's `observationCardinality` is 274, comfortably above the 150 minimum the
1800s TWAP needs. Nothing about the wiring is wrong: correct registry, correct
adapter, allowlisted route, unpaused, empty, cap set.

## Why this is gated behind 013 (the predecessor)

`addStrategy` itself succeeds with UNI unsupported — it only checks role, pause
state, code presence and duplicate registration. So the precondition is not
enforced by that call. But a strategy holding **UNI** is unpriceable until 013
lands, and that is a protocol-wide hazard, not a local one.

Measured on a fork pinned to block 26027017, registering the strategy first and
then giving it 1 UNI:

| Step | Result |
|---|---|
| `addStrategy` with 013 unexecuted | succeeds (249,565 gas) — registration alone is fine |
| `StrategyManager.totalNAVInETH()`, strategy empty | works (strategy is worth 0) |
| `strategy.navInETH()` with 1 UNI held | **reverts `0x868fd74e`** `OracleTokenNotSupported()` |
| `StrategyManager.totalNAVInETH()` with 1 UNI held | **reverts `0x868fd74e`** — NAV frozen protocol-wide |
| …then execute 013 op1 + op2 | both calls work again |
| `strategy.navInETH()` after 013 | `0xb92d5f861ec02` = 3,257,672,436,673,538 wei ≈ 0.0032577 ETH for 1 UNI |
| `totalNAVInETH()` after 013 | baseline + exactly that value |

So without 013 the strategy is not merely useless, it is a loaded gun: the first
UNI it holds makes `totalNAVInETH()` revert, and every deposit, withdrawal and
keeper flow that reads NAV reverts with it. Encoding 013 op2 as the predecessor
makes the ordering impossible to get wrong — the timelock refuses to execute
this operation until UNI is priceable, whatever order the queue ends up in.

The cost of that choice: if 013 is ever cancelled, this operation becomes
unexecutable and must be re-scheduled with a zero predecessor. That is the
fail-closed direction, which is the one to err in.

## Weights: 10 / 10

Weights are normalised across eligible strategies, so this dilutes the existing
three rather than adding a fourth share: 40/15/45/10 → **36.4 / 13.6 / 40.9 / 9.1 %**.

10 is deliberately the most conservative weight in the set, on three grounds:
this is the smallest cap of the four (250 ETH vs 1000 / 140 / 4500); UNI is the
first paired token that is neither ETH nor a USD stable, so the position carries
idiosyncratic single-asset risk on top of the β≈0.5 the other pools already have;
and the strategy is unproven in production. It is a one-line change to re-cut if
the DAO wants it to matter more on day one.

## Simulations

Fork (anvil, mainnet pinned to 26027017), full real path through the timelock:

| Test | Result |
|---|---|
| direct `addStrategy` from a non-ADMIN account | reverts `0x4d616cff` `RegistryClientMissingRole` |
| `schedule` from the DAO Safe | succeeds, 57,420 gas |
| `execute` before the 48h delay | reverts `0x5ead8eb5` `TimelockUnexpectedOperationState` |
| `execute` after 48h with 013 op2 **unexecuted** | reverts `0x90a9a618` `TimelockUnexecutedPredecessor` |
| `execute` after 013 op2 is Done | succeeds, **269,246 gas**; registered, `strategyCount` 3 → 4, weights 10/10, `CONVERTER_CALLER_ROLE` granted, op state Done |
| re-`execute` the same op | reverts `0x5ead8eb5` (single-shot) |

## Bytecode provenance — OPEN ITEM

This deployment is **not reproducible from any committed revision I can build**,
and it was compiled with different settings than the three strategies already
registered. Measured:

| Binary | Runtime size | CBOR metadata |
|---|---|---|
| `0x2c3AEF…0715d` (this one) | **24,570 B** | 10 B — solc 0.8.30, **IPFS hash stripped** |
| WETH/USDT strategy (registered) | 24,391 B | 51 B — IPFS hash present |
| `origin/main` (49c0e63) build | 24,391 B | 51 B — IPFS hash present |

The `origin/main` build reproduces the **registered** binaries: identical length,
differences confined to 1,057 bytes in 56 runs, i.e. the immutable slots alone.
So `origin/main` is the baseline the existing three came from, and this new
contract is **+179 bytes** of extra code, differing from that baseline in 20,592
bytes across 568 runs — a different build, not a different set of constructor
arguments.

What is reassuring: the ABI surface is **identical** — 61 of 61 `PUSH4 … EQ`
dispatcher entries match the registered strategy exactly, so no new or changed
external function, no new privileged entry point. What is not: a 179-byte code
delta plus a different metadata setting means I cannot tell what source produced
this binary, and therefore cannot claim the audit's line coverage applies to it.

It is not a build-profile difference either. Rebuilding the same source under
other profiles does not reach 24,570 B:

| Build profile | Runtime size |
|---|---|
| default | 24,391 B |
| `via_ir = true` | 24,391 B |
| `bytecode_hash = none` | 24,350 B (metadata stripped, as on-chain) |
| `optimizer_runs = 1000000` | 32,912 B |

`bytecode_hash = none` explains the stripped metadata but not the size. The
+179 B is therefore a **source** difference, not a compiler-setting difference.

**Before this executes, the deployer should supply the commit and the build
profile used.** If it turns out the 179 bytes are a change to strategy logic,
that change has not been audited, and it is sitting behind a `maxTotalNAV` of
250 ETH — which is the whole reason to settle it now rather than after deposits.

## Verifying this independently

```bash
RPC=https://ethereum-rpc.publicnode.com
TL=0xF0911198Ef0a4b4234546fa5F50d6d1D45091774
SM=0x94916ab93C669E7c734f844dB019Ce9449a3b5C9
NEW=0x2c3AEFaCb41065810f43F23d0eBa8dF20350715d

# 1. the operation id
cast call $TL "hashOperation(address,uint256,bytes,bytes32,bytes32)(bytes32)" \
  $SM 0 0xdca0c48f0000000000000000000000002c3aefacb41065810f43f23d0eba8df20350715d000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000a \
  0xcd443da64d612cc5bc96157197e5ab62c5207d1d45d402be1d2e5170a5f35ae9 \
  0x04c997170365fd7c8ae41d6545e71cd8636caacda545cb6c146a8e7b81dc40b2 --rpc-url $RPC

# 2. is it live yet, and did it run
cast call $TL "getOperationState(bytes32)(uint8)" 0x489a556a585fbae6efeb04d3f1627d9168124c63a2b089e29b2e5ba42201cd07 --rpc-url $RPC
cast call $SM "isStrategyRegistered(address)(bool)" $NEW --rpc-url $RPC

# 3. the dependency this waits on
cast call $TL "getOperationState(bytes32)(uint8)" 0xcd443da64d612cc5bc96157197e5ab62c5207d1d45d402be1d2e5170a5f35ae9 --rpc-url $RPC
```

## Status

🔴 **WITHDRAWN — 2026-09-21 16:40 UTC. Never executed; nothing ever reached the timelock.**

Withdrawn by **Arseny**: *"it seems that the strategy is not completely ready and we didn't
enable Uni oracle."*

### What had been filed

- `safeTxHash` = `0x49019656754da2fb6e5146e0b5163bd2770ba603b882abb9cddcbd4c78f7c1f9`
- submitted `2026-09-21T16:29:08Z`, nonce 24, origin `everstrat/015-strategy-manager-add-unicl-uni-weth-strategy`
- proposedByDelegate `0x1483E048…0E922`, proposer `0x4A2D30c7…1F0d2`
- stored `data` byte-verified against [`01-schedule-raw.json`](01-schedule-raw.json) (714 chars, both sides)
- **`confirmations: []`** — no owner ever signed it, so it could never have executed on its own

### Why there was nothing to cancel on-chain

The Safe tx never executed, so `schedule(...)` never ran and the operation was never
created:

```sh
cast call $TL "getOperationState(bytes32)(uint8)" 0x489a556a585fbae6efeb04d3f1627d9168124c63a2b089e29b2e5ba42201cd07 --rpc-url $RPC
# 0  (Unset)
```

No scheduled operation, no ETA, no `isOperationPending` — and therefore no
`TimelockController.cancel(bytes32)` to call. The DAO Safe's `CANCELLER_ROLE` is
irrelevant to this case; it only matters once an operation actually exists.

### How it was cancelled

`DELETE /safes/{safe}/multisig-transactions/{hash}/` is **not exposed** on
`api.safe.global` — it answers an HTML `404` (and the OpenAPI schema is not published at
any of the usual paths), so the queue entry could not simply be deleted. The enforceable
cancel is the standard Safe **rejection transaction**: a tx at the *same nonce* from the
Safe to itself.

| field | value |
|---|---|
| `to` | `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9` (the Safe itself) |
| `value` | `0` |
| `data` | `0x` |
| `operation` | `0` (call) |
| `nonce` | `24` |
| `safeTxHash` | `0xe51719fe3911bd8d37f5b56939f17a415d9fbf73437a28b8caa3d8dacabff734` |
| submitted | `2026-09-21T16:40:52Z` |

Both transactions now sit at nonce 24. The first one to collect **3-of-4** confirmations
and execute consumes the nonce, and a Safe nonce can only ever be spent once — so the 015
tx becomes permanently unexecutable. Until that rejection is signed, the 015 tx remains
harmless but *visible* in the queue, so owners should sign the rejection rather than
merely ignore it.

### Re-proposing later

Unchanged by the withdrawal: the inner calldata is sound and `01-schedule-raw.json` can be
re-filed as-is once the strategy is final. Two things should be settled first:

1. **013 must be executed** (op1 `updateUsdFeedInfo(UNI…)`, then op2
   `addSupportedERC20(UNI)`) — that is exactly the "we didn't enable Uni oracle" gap, and
   it is why the op carried 013's op id as its predecessor.
2. **The deployed binary should be attributable** — see the provenance finding below
   (24,570 B vs the 24,391 B reproduced from `origin/main`).

Sequence lives in [`../README.md`](../README.md).
