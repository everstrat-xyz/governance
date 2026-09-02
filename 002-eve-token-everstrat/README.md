# 002 — EVE token swap: "Everything Strategy" → "Everstrat"

**Status:** ✅ **Scheduled on mainnet** (tx `0x65691a…`, block 25883107, 2026-09-01 14:43 UTC). Executable from **2026-09-03 14:43 UTC** — permissionless execute.
**Operation id:** `0xc396f407d91a7b213bf60eb9ef50cea044810daee3fc394bb6924c7de727f3` (recomputable via `hashOperation`, verified on mainnet).

## What it changes

`Registry.registerContract(keccak256("EVE"), 0x8FE6A43672fCa70d41e112B8426387665fD061EC)`
— re-point the protocol's EVE share-token from the current deployment
`0x3A373227AECE982F07B9D14711a05d7a879859D4` to a new static token whose
`name()`/`symbol()` are `Everstrat`/`EVE`.

**What it does NOT do:** touches no other registry key, no role, no strategy, no AMM
parameter. `MINTER_ROLE` lives on the **Registry** (`Auth.sol`), so mint/burn access
carries over automatically to the new address — no role grants needed.

## Why

The supply token was deployed under the working name "Everything Strategy"; the project
ships under the **Everstrat** brand. `EVE.sol` embeds `name()` at construction, so the
token cannot be renamed in place — the clean move while `totalSupply` is `0` is a
registry repoint to a correctly-named static token (see also team note: re-register is
safe while "supply" is present).

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` with `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` after the delay |

`01-schedule-raw.json` is the same schedule as raw calldata (selector `0x01d5062a`).

### Parameters

| Field | Value |
|---|---|
| `to` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (admin timelock) |
| `target` | `0x46AA1bd55c19be90d8767e0C22732A7DD31D993D` (Registry) |
| `value` | `0` |
| `data` | `0x645c6faeaf94fe894bf0e22494392493fc7eb18a0ab98754fe785e74fd233f476b9c37c90000000000000000000000008fe6a43672fca70d41e112b8426387665fd061ec` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0x5013804e985fe4a240876a3e3f5eae32462c68812878f6b6827435af7607b548` |
| `delay` | `172800` (48h) |

The salt must be reused byte-for-byte at execute time. It derives from
`keccak256("everstrat/eve-token/everstrat/2026-09-01")`.

## Verification performed (fork of mainnet, before drafting)

1. DAO Safe → `schedule` with this exact calldata — success (state Waiting).
2. `isOperationPending` true; `execute` before the delay reverts
   `TimelockUnexpectedOperationState(…, 4)`.
3. Warp 48h+1s, `execute` from an unrelated EOA — success (status 1).
4. Emitted `ContractRegistered(keccak256("EVE"), old → 0x8FE6A43672fCa70d41e112B8426387665fD061EC)`.
5. `Registry.getContractByKey(keccak256("EVE"))` returned `0x8FE6A43672fCa70d41e112B8426387665fD061EC`.
6. New token probed on mainnet: `name()="Everstrat"`, `symbol()="EVE"`,
   `totalSupply()=0`, static (no EIP-1967 slot).

Recompute the operation id from the fields as shown in the Safe UI and compare against
`0xc396f407d91a7b213bf60eb9ef50cea044810daee3fc394bb6924c7de727f3`.

## Cancelling

Either the DAO Safe or the Security Safe may call
`cancel(0xc396f407d91a7b213bf60eb9ef50cea044810daee3fc394bb6924c7de727f3)` on the timelock any time before execution.
