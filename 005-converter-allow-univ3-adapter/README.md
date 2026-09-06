# 005 — Converter: whitelist the Uniswap V3 adapter

**Status:** ⏳ **Scheduled on mainnet** (tx `0x85192e709b06d5e165e4ff3944c2ea8166af2417225fcbc6c60d1604f5bff23a`, block 25920019, 2026-09-06 18:11:47 UTC). Timelock state Waiting. Executable from **2026-09-08 18:11:47 UTC** — permissionless execute.
**Operation id:** `0x40ea4797d59df5b32a799cb1f8586c5511bdc7255862818f7cf691accfa388f3`
(`hashOperation(Converter, 0, setAllowedAdapter(adapter, true), predecessor, salt)` — recomputed and verified on mainnet).

## What it changes

`Converter.setAllowedAdapter(0x0844580a121124CAEc6Cf4A933aac401813cCde5, true)` — adds the
deployed `UniswapV3ConverterAdapter` to the Converter's adapter allowlist
(`_allowedAdapters` set; checked by `isAdapterAllowed`, listed by `getAllowedAdapters`).

The Converter dispatches every swap by `DELEGATECALL` into a caller-supplied adapter, but
only after checking that adapter is on this allowlist. Until the adapter is whitelisted,
`swapExactAmountIn` / `swapExactAmountOut` / `quoteSwap*` all revert `ConverterAdapterNotAllowed`
for it.

**What it does NOT do:** grants no role, registers no Oracle feed, wires no strategy, moves
no funds. It does not make any swap reachable on its own — a strategy still has to supply a
route, and the route's tokens still have to be Oracle-priceable.

## Why

`DeployUniCLStrat` validates its configured routes against the Converter allowlist in its
constructor, so the adapter must be whitelisted **before** the Uniswap CL strategy can be
deployed. This is the same 48h-timelock gate that strategy registration and Oracle feed
changes go through — deliberately not a deploy-time action (see
`DeployUniswapV3ConverterAdapter.s.sol`, which refuses to call `setAllowedAdapter` itself).

The adapter itself is static and immutable — no owner, no upgrade path, every integration
address fixed in code at construction (verified on-chain: `router()`, `factory()`,
`oracle()`, `weth()`, `twapInterval()` all match the intended values, and the runtime
bytecode is byte-identical to the compiled source). Whitelisting is the only action that
brings it into use; `setAllowedAdapter(adapter, false)` (ADMIN, 48h) is the only way to
take it back out.

**Timing:** proposed while no strategy routes through the Converter yet, so whitelisting
has no effect on live flow until a Uniswap strategy is deployed and funded.

## No predecessor

`setAllowedAdapter` checks only: caller holds `ADMIN_ROLE`, `_adapter != address(0)`, and
`_adapter.code.length != 0`. It does **not** touch the Oracle or any other timelock
operation, so this proposal has no timelock `predecessor` (`0x00…00`). It is independent of
003 and 004.

That said, the adapter is only *useful* once the Oracle can price the routes it will be
asked to quote: the adapter maps `weth → address(0)` for Oracle lookups, so a WETH↔USDC
route needs both the native-ETH (`address(0)`) USD feed **and** the USDC USD feed (003)
registered, or `quoteExactAmountIn/Out` reverts. Whitelisting does not depend on that;
routing through it does.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` with `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` after the delay |

`01-schedule-raw.json` is the same transaction as raw calldata (selector `0x01d5062a`), for
when the Safe UI cannot resolve the timelock ABI.

### Parameters

| Field | Value |
|---|---|
| `to` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (admin timelock) |
| `target` | `0xc8700441A8ca74aE5390F137bf3Dd659CaD570de` (Converter) |
| `value` | `0` |
| `data` | `0x73721fe90000000000000000000000000844580a121124caec6cf4a933aac401813ccde50000000000000000000000000000000000000000000000000000000000000001` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0x7122c3f7da42822abd51144eda21d5acb3f05d180f3e870a920c0f17b27715d3` |
| `delay` | `172800` (48h — the enforced minimum) |

`data` decodes to `setAllowedAdapter(address,bool)` (selector `0x73721fe9`) with
`_adapter = 0x0844580a121124CAEc6Cf4A933aac401813cCde5`, `_allowed = true`.

The salt derives from `keccak256("everstrat/converter/allow-adapter/univ3/2026-09-06")`.
Predecessor and salt must be reused **byte-for-byte** at execute time.

## Verification performed (forked mainnet, before signing)

Fork at block 25919677. `Converter.isAdapterAllowed(adapter) == false`,
`getAllowedAdapters() == []`, `Converter.paused() == false`,
`Registry.hasRole(ADMIN_ROLE, timelock) == true`, adapter `code.length == 6381`.

1. Direct `setAllowedAdapter(adapter, true)` from an unrelated EOA — **reverts**
   `RegistryClientMissingRole(ADMIN_ROLE)` (`0x4d616cff`).
2. DAO Safe → `schedule` with the exact calldata above — **success**, 56,622 gas.
   `CallScheduled` + `CallSalt` emitted for op `0x40ea4797…fa388f3`.
3. `isOperationPending` `true`, `isOperationReady` `false`, ready-at exactly `scheduled + 172800`.
4. `execute` before the delay — **reverts** `TimelockUnexpectedOperationState` (`0x5ead8eb5`, state 4 = Waiting).
5. Warp 48h + 1s. `execute` **from an unrelated EOA** — **success**, 122,848 gas.
   `AdapterUpdated(adapter, true)` emitted (`0x3a7f7534…9cd5`), confirming execution is permissionless.
6. Post-state: `isAdapterAllowed(adapter) == true`, `getAllowedAdapters() == [adapter]`.
7. Re-`execute` and re-`schedule` the same op — both revert `TimelockUnexpectedOperationState`;
   a direct re-add reverts `ConverterAdapterAlreadyAllowed` (`0x29bcb2f9`).

Recomputing `hashOperation` from the fields as rendered in the Safe UI reproduces the
operation id above.

## On-chain schedule (mainnet)

Scheduled 2026-09-06 18:11:47 UTC in tx
`0x85192e709b06d5e165e4ff3944c2ea8166af2417225fcbc6c60d1604f5bff23a` (block 25920019),
DAO Safe → timelock `schedule` (Safe nonce 8). Confirmed against mainnet:

- The `schedule` calldata in the tx matches `01-schedule.json` byte-for-byte (target
  Converter, `data` `0x73721fe9…0001`, predecessor `0x00…00`, salt
  `0x7122c3f7…715d3`, delay `172800`).
- `getOperationState(0x40ea4797…fa388f3)` → `1` (Waiting); `isOperationPending` → `true`.
- `getTimestamp` → `1788891107` = **2026-09-08 18:11:47 UTC** (ready-at).
- `Converter.isAdapterAllowed(adapter)` still `false` — flips on execute.

## Cancelling

Either the DAO Safe or the Security Safe may call
`cancel(0x40ea4797d59df5b32a799cb1f8586c5511bdc7255862818f7cf691accfa388f3)` on the timelock
at any point before execution. After execution, `setAllowedAdapter(adapter, false)` (ADMIN,
48h) removes the adapter again.
