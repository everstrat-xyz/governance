# 001 — AMM `connectorWeight` 0.5 → 0.9

**Status:** Scheduled on mainnet. Executable from **2026-09-03 08:35:11 UTC**.
**Operation id:** `0x8c5d418c1be823549d40cfd080e9a43e263829809264321a200e6ea2b97f5db2`

## What it changes

`AMM.setConnectorWeight(900000000000000000)` — the connector weight, stored as
fixed-point with `SCALE_FACTOR = 1e18`, so `9e17` is 0.9.

Connector weight sets the spread between the mint price and the redemption price:

```
premium (mint) = NAV / (supply · cw)
base    (burn) = NAV / supply
```

| `cw` | Entrant pays |
|---|---|
| 0.5 (before) | **2.000×** the base price |
| 0.9 (after) | **1.111×** the base price |

The premium is retained by the protocol and accrues to existing holders, so this is a
material shift in favour of new depositors and away from incumbents.

## Why

`0.5` was never chosen. It is `DEFAULT_CONNECTOR_WEIGHT = 5e17`, a hardcoded constant in
`script/ProtocolDeployBase.sol`, passed through by `DeployAll`. Unlike every other launch
parameter — which the deploy scripts deliberately refuse to default, so a missing value
reverts the deployment — connector weight was never surfaced as a decision. It arrived
by default and went live with the deployment.

A 2× entry premium is aggressive; comparable bonding-curve protocols sit in the
0.8–0.95 range. 0.9 was chosen as a deliberate, moderate premium.

**Timing:** proposed while the AMM is unbootstrapped, `totalSupply` is zero, and no
address is whitelisted. No user is affected, because no user can yet enter. This is the
only window in which this parameter can change without altering someone's position.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` with `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` after the delay |

`01-schedule-raw.json` is the same transaction as raw calldata, for when the Safe UI has
not yet cached the target's ABI (it returns *"Contract ABI doesn't have any public
methods"* against a contract verified only minutes earlier).

### Parameters

| Field | Value |
|---|---|
| `to` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (admin timelock) |
| `target` | `0x42c618D9457BE7cb3b836F7FD3332E2800C48a8b` (AMM) |
| `value` | `0` |
| `data` | `0x1b710e970000000000000000000000000000000000000000000000000c7d713b49da0000` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0xd4893469daf02f45827715d83f62408024a12ef71566a1b35d94d4fde921db7b` |
| `delay` | `172800` (48h — the enforced minimum) |

The salt must be reused **byte-for-byte** at execute time. It derives from
`keccak256("everstrat/connector-weight/0.9/2026-09-01")`.

## Verification performed

Simulated end to end against forked mainnet state before signing:

1. DAO Safe → `schedule` with this exact calldata — **success**, 56,027 gas.
2. `isOperationPending` `true`, `isOperationReady` `false`, ready-at exactly `scheduled + 172800`.
3. `execute` before the delay — **reverts** `TimelockUnexpectedOperationState` (`0x5ead8eb5`, state 4 = Waiting).
4. Warp 48h + 1s, `execute` **from an unrelated EOA** — success, 52,375 gas, confirming execution is permissionless.
5. `AMM.connectorWeight()` → `900000000000000000`. Correct.

Recomputing `hashOperation` from the fields as rendered in the Safe UI reproduces the
operation id above, which covers target, value, data, predecessor and salt in one hash.

## Bounds

`setConnectorWeight` rejects `0` and anything above `1e18`:

```solidity
if (_cw == 0) revert AMMInvalidRange();
if (_cw > Math.SCALE_FACTOR) revert AMMInvalidRange();
```

A zero weight would divide by zero in the premium formula; the contract refuses it.

## Cancelling

Either the DAO Safe or the Security Safe may call
`cancel(0x8c5d418c1be823549d40cfd080e9a43e263829809264321a200e6ea2b97f5db2)` on the
timelock at any point before execution.
