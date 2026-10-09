# 018 — Controller: upgrade implementation 1.0.0 → 1.1.0 (`withdrawFromStrategy` callable by ADMIN)

**Status:** 📝 **Proposed** — queued in the DAO Safe at **nonce 28**, 1 of 3 confirmations, not
executed (`safeTxHash = 0x03dac026627cb5d5df700d23ec7426e1eda2a011f3454ee5fbe19fb5f4346abc`). The
stored transaction equals `01-schedule-raw.json` byte-for-byte. Nothing is on the timelock yet
(`getOperationState == 0`); after 3-of-5 confirmations and the Safe execution, the 48h delay starts.
**Operation id:** `0x8cc3710fc1e770419105d3974350f2de05eb8a575e8057bf602fd28830b7c98a`
(`hashOperation(Controller, 0, upgradeToAndCall(0xd4f4…d55D, 0x), 0x00…00, salt)` — recomputed
locally and reproduced by the mainnet timelock's own `hashOperation`; `getOperationState == 0`
(Unset) at block 26151707).

## What it changes

`Controller.upgradeToAndCall(0xd4f4152a887253CFdD23d0289951176Ec13cd55D, 0x)` on the Controller
UUPS proxy `0x9097868dcbda5a630729015DE1ddeCd66d74052c`. The ERC-1967 implementation slot moves
from `0x6bf777b1173b49173cce5ec82505ebd7ce5cbefc` (`version() == "1.0.0"`) to
`0xd4f4152a887253CFdD23d0289951176Ec13cd55D` (`version() == "1.1.0"`). Emits `Upgraded(0xd4f4…d55D)`.
The proxy address — the one in the root README's address table and in the Registry — is unchanged.

The only behavioural change is one access modifier
([contracts#52](https://github.com/everstrat-xyz/contracts/pull/52), `be407c9`):

```diff
 function withdrawFromStrategy(address _strategy, uint256 _amount)
     external
     override
-    onlyAuthRole(Auth.KEEPER_ROLE)
+    onlyEitherAuthRole(Auth.ADMIN_ROLE, Auth.KEEPER_ROLE)
     whenNotPaused
     nonReentrant
```

After the upgrade, `ADMIN_ROLE` — i.e. this timelock, 48h — can pull ETH out of one specific
strategy into the Controller. `KEEPER_ROLE` keeps the access it already had. Every other function
is unchanged, and there is **no storage change**: the Controller's only sequential storage is the
reserved `uint256[50] __gap` at slot 0 (same in both versions — `forge inspect … storageLayout`);
all live state sits in OpenZeppelin ERC-7201 namespaces and the Registry client's namespaced slot.
No re-initializer is needed and `data` is empty.

## Why

Requested by **Vadyusha** (2026-10-09). The rationale is the one stated in the contracts change:
governance can **drain a specific strategy** — e.g. before migrating to a replacement and calling
`removeStrategy` — without a break-glass keeper. Today only the keeper executor can call
`withdrawFromStrategy`, and it only does so for its own upkeep reasons, so a deliberate wind-down of
one strategy has no governed path.

## No predecessor

The upgrade has no on-chain dependency on any other scheduled operation, so
`predecessor = 0x00…00`.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` with `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` after the delay |

`01-schedule-raw.json` is the same transaction as raw calldata (selector `0x01d5062a`).

### Parameters

| Field | Value |
|---|---|
| Timelock | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (48h) |
| `target` | Controller proxy `0x9097868dcbda5a630729015DE1ddeCd66d74052c` |
| `value` | `0` |
| `data` | `upgradeToAndCall(address,bytes)` (`0x4f1ef286`) — `0x4f1ef286000000000000000000000000d4f4152a887253cfdd23d0289951176ec13cd55d00000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000000` |
| New implementation | `0xd4f4152a887253CFdD23d0289951176Ec13cd55D` |
| Initializer `data` | `0x` (none) |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0x7c850ef421d89b725d209b16d857da947a373067cf7bd6d56d2e2405f069143f` = `keccak256("everstrat/controller/upgrade-v1.1.0/2026-10-09")` |
| `delay` | `172800` (48h, the enforced minimum) |

## Pre-wiring check (new implementation)

| Check | Result |
|---|---|
| Deployment | tx `0x152ac78ca45eb29caf13f7be3e12006d68678cdddc74c0621e10b93de69787bf`, block 26151685, gas 3,175,851, `CREATE` from `0x046E01Ee…a899D7` — the same deployer as the live implementation. Foundry broadcast of `script/UpgradeController.s.sol` records commit `157129b` |
| Bytecode | `forge build` of `src/contracts/Controller.sol` at contracts `origin/main @ 157129b` (repo profile: solc 0.8.30, `via_ir`, 200 runs, cancun) reproduces the deployed runtime **byte-for-byte**: 14,320 B, 0 differing bytes outside the immutable, identical 51-byte CBOR metadata. The one immutable (UUPS `__self`, 2 refs) holds `0xd4f4…d55D` itself |
| `version()` | `"1.1.0"` (live implementation: `"1.0.0"`) |
| `proxiableUUID()` | `0x360894a1…382bbc` = ERC-1967 implementation slot — `upgradeToAndCall` accepts it |
| `UPGRADE_INTERFACE_VERSION` | `"5.0.0"` |
| Initializers | locked on the implementation: `Initializable._initialized == type(uint64).max`; `initialize(...)` on it reverts `InvalidInitialization` `0xf92ee8a9` |
| Upgrade auth | `_authorizeUpgrade` is `onlyAuthRole(ADMIN_ROLE)` — only the timelock can upgrade |
| Source diff vs live | Diffed against the live implementation's **Blockscout-verified** source: `Controller.sol` differs only in the modifier above and the version string; `IController.sol` only in NatSpec; `IStrategy.sol` only in NatSpec on `isHealthy()` |
| Explorer verification | verified on Blockscout as `Controller` (2026-10-09) |

**Library delta.** The live 1.0.0 binary was compiled against newer OpenZeppelin
*non-upgradeable* sources (file headers v5.5–v5.7) than the contracts repo pins
(`openzeppelin-contracts-upgradeable` v5.3.0 → nested `openzeppelin-contracts` v5.3.0). The
upgradeable sources are identical on both sides. Of the non-upgradeable files, only two are
compiled into Controller code paths, and both behave the same:

- `ERC1967Utils.upgradeToAndCall` — identical body.
- `Address.sendValue` (used for `provideExitLiquidity` / `emergencyExitToAMM`) — both forward all
  gas, bubble the callee's revert data, and otherwise revert `FailedCall`; v5.5 goes through the
  `LowLevelCall` helper, v5.2 through `call` + `_revert`.

`ReentrancyGuard` (non-upgradeable), whose storage model differs between those versions, is
**not** inherited by the Controller — it uses `ReentrancyGuardUpgradeable`, identical on both sides.
This delta is why the live binary is 14,161 B and a rebuild of the pre-#52 commit `b599469` is
14,181 B: neither the live nor the new implementation is the "old source + one line".

## Verification performed

Mainnet forks (anvil, `ethereum-rpc.publicnode.com`), 2026-10-09.

**Fork A — timelock path** (block 26151729):

| Step | Result |
|---|---|
| a) `upgradeToAndCall(new, 0x)` on the proxy from a role-less EOA | revert `RegistryClientMissingRole(ADMIN_ROLE)` `0x4d616cff` |
| a′) `upgradeToAndCall` called on the implementation directly | revert `UUPSUnauthorizedCallContext` `0xe07c8dba` |
| a″) `initialize(registry)` on the new implementation | revert `InvalidInitialization` `0xf92ee8a9` |
| c) DAO Safe → `schedule(...)` | OK, gas **57,024**; `CallScheduled` + `CallSalt`; state `1 (Waiting)` |
| d) `execute` before the delay | revert `TimelockUnexpectedOperationState` `0x5ead8eb5` |
| e) `evm_increaseTime 172801` | state `2 (Ready)` |
| g) `execute` from a role-less EOA | OK, gas **60,432**; `Upgraded(0xd4f4…d55D)` + `CallExecuted`; state `3 (Done)` |
| h) state after vs before | implementation `0x6bf7…befc` → `0xd4f4…d55D`; `version` `1.0.0` → `1.1.0`; `registry()` `0x46AA…993D`, `paused() == false`, balance `50000000000000001` wei and `_initialized == 1` all **unchanged** |
| i) re-`execute` / re-`schedule` | revert `0x5ead8eb5` |

**Fork B — the new permission, no time warp so feeds stay fresh** (block 26151731; timelock
impersonated for the upgrade itself):

| Step | Result |
|---|---|
| timelock `withdrawFromStrategy(UNI/WETH, 0.01 ETH)` **before** the upgrade | revert `RegistryClientMissingRole(KEEPER_ROLE)` `0x4d616cff` |
| timelock `upgradeToAndCall(new, 0x)` | OK, gas 43,192 |
| role-less EOA `withdrawFromStrategy` | revert `RegistryClientCallerHasNoneOfRoles(ADMIN_ROLE, KEEPER_ROLE)` `0x958c6bdf` |
| timelock `withdrawFromStrategy(…, 0)` | revert `ControllerZeroAmountRequested` `0x7ed370c6` |
| timelock `withdrawFromStrategy(UNI/WETH 0x956F…AfDd, 0.01 ETH)` | OK, gas 1,709,025; `FundsWithdrawn`, `FundsWithdrawnFromStrategy`, `DirectWithdrawalCompleted`; Controller +0.01 ETH exactly |
| timelock drains the strategy: `withdrawFromStrategy(0x956F…AfDd, navInETH())` | OK, gas 1,545,917; strategy `navInETH()` `0.106012…` → **0**; Controller received 0.105918… ETH (≈ 0.09% unwind cost); strategy stays registered |
| keeper executor `withdrawFromStrategy(WETH/USDT, 0.01 ETH)` | OK before (gas 1,666,017) and after (gas 1,657,656) the upgrade — keeper access unchanged |
| Security Safe `pause()`, then timelock `withdrawFromStrategy` | revert `EnforcedPause` `0xd93c0665` — the pause gate still applies |

One keeper call in fork B mined with status 0 at 1,732,116 gas when sent with cast's automatic gas
limit after the preceding drains; re-run on a fresh fork with an explicit limit and with cast's own
estimate (1,915,331 limit) it succeeds before and after the upgrade alike. It is the
tight-gas-limit effect recorded in [016](../016-strategy-keeper-exit-settlement-funding/), not a
change from this upgrade.

## Risks

- **Explorer verification.** The new implementation is source-verified on Blockscout as
  `Controller` (checked 2026-10-09), in addition to the local byte-for-byte reproduction above.
- **The upgrade script is not committed.** `script/UpgradeController.s.sol`, which produced the
  deployment, exists only as an uncommitted file in the contracts checkout; commit it so the
  deployment is reproducible from the repo.
- **ADMIN gains a direct strategy-withdraw lever.** It moves ETH from a strategy to the Controller
  only — funds stay inside the protocol and remain in NAV — and every use is itself a 48h
  timelock proposal. ADMIN already controls upgrades, so this widens no trust boundary.
- **Unwinding has a cost.** Draining the UNI/WETH strategy on the fork lost ≈ 0.09% to swap and
  pool fees; larger strategies with deeper positions will differ.
- **Library versions move back to the repo's pins** (see *Library delta*). Behaviour-equivalent for
  the Controller's use, but it is a change of compiled dependency code, not just the one modifier.
- Rollback is another 48h upgrade back to `0x6bf777b1173b49173cce5ec82505ebd7ce5cbefc`; there is no
  SECURITY fast path for upgrades. `pause()` (ADMIN or SECURITY, instant) stops
  `withdrawFromStrategy` for every caller in the meantime.

## Safe proposal (mainnet)

| Field | Value |
|---|---|
| Proposed by | owner `0xF412F1A5d22f08FBD406D3B2B52e80336fa8E149`, via the Safe Transaction Builder; submitted 2026-10-09 02:18:02 UTC |
| Safe | DAO Safe `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9`, nonce 28 (single `schedule` call, `operation = 0`) |
| safeTxHash | `0x03dac026627cb5d5df700d23ec7426e1eda2a011f3454ee5fbe19fb5f4346abc` — reproduced by the Safe's own `getTransactionHash(…, nonce 28)` |
| Calldata check | `to` = timelock, `value 0`, `data` equals `01-schedule-raw.json` byte-for-byte; the service decodes it as `schedule(Controller, 0, upgradeToAndCall(0xd4f4…d55D, 0x), 0x0, 0x7c850ef4…f069143f, 172800)`; `safeTxGas`/`baseGas`/`gasPrice` 0, no refund receiver |
| Signatures | 1 of 3 — `0xF412…E149` 02:18:02 UTC |
| Operation state | `0 (Unset)` (checked 2026-10-09) |

## Cancelling

Before execution, either the DAO Safe or the Security Safe may call
`cancel(0x8cc3710fc1e770419105d3974350f2de05eb8a575e8057bf602fd28830b7c98a)` on the timelock. Before
the Safe executes the `schedule`, reject it at the same Safe nonce instead. After execution, upgrade
back to `0x6bf777b1173b49173cce5ec82505ebd7ce5cbefc` through a new 48h `ADMIN_ROLE` proposal.
