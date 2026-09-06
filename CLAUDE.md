# Working in this repo

This repo is a **human-readable record** of every privileged action on the EverStrat
mainnet protocol. Nothing here executes anything — each directory holds the exact calldata
that was (or will be) signed, the reasoning behind it, a fork simulation, and on-chain
verification of the resulting state.

Every privileged call routes through a 48h OpenZeppelin `TimelockController` (v5). The DAO
Safe can only `schedule` / `cancel`; `execute` is permissionless once the delay elapses.

## Task shapes

- **New proposal** — add `NNN-slug/` with a README + Safe Transaction Builder JSON, simulate
  it, open a PR.
- **Reconstruct** an already-signed DAO Safe action into a `NNN-slug/` record after the fact.
- **Status update** — move a proposal Draft → Scheduled → Executed as it lands on-chain, and
  update the root README `## Decisions` row.
- **Pre-wiring check** — sanity-check a freshly deployed protocol contract before a
  governance action references it (bytecode matches source, constructor args, roles).

## Conventions

- Directory `NNN-short-kebab-slug/`. Files: `README.md`, `01-schedule.json`,
  `02-execute.json`, and `01-schedule-raw.json` (raw-calldata variant, selector
  `0x01d5062a`, for when the Safe UI can't resolve the timelock ABI).
- README order: `# NNN — <title>`, `**Status:**` line, `**Operation id:**` line, then
  `## What it changes`, `## Why`, `## Transactions` (table), `### Parameters` (table),
  `## Verification performed`, `## Cancelling`. Add `## Risks`, `## Dependency on NNN`,
  `## Provenance`, `## On-chain schedule` when relevant. Copy tone from `001`/`004`.
- Root `README.md` `## Decisions` table: one row per proposal —
  `| [NNN](NNN-slug/) | <decision> | <status> | <op id / …abbrev…> |`. Keep the
  `## Mainnet addresses` table current when an address changes (see 002).
- **Salt:** `salt = keccak256("everstrat/<area>/<detail>/<yyyy-mm-dd>")`. Always a real
  per-proposal salt — a zero salt (003 did this) breaks the convention and leaves no
  on-chain fingerprint. If you inherit one, note the deviation.
- **Predecessor:** `0x00…00` unless the action has a genuine on-chain dependency on another
  scheduled op (004 → 003's USDC feed). Then set `predecessor` to that op id — `execute`
  reverts `TimelockUnexecutedPredecessor` (`0x90a9a618`) until the dependency is Done —
  and document both the coupling and the `predecessor = 0x0` alternative.
- `createdAt` in the JSON meta is nominal; a midnight-UTC ms timestamp for the proposal
  date is fine.
- One branch + PR per proposal: `docs/NNN-slug` → `main`. If a PR is stacked on another
  open PR, base it on that branch and retarget to `main` after the base merges
  (`git rebase origin/main` + `git push --force-with-lease`).
- Commit trailer: `Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>`.
  PR body trailer: `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
  For long PR bodies use `gh pr create --body-file` (avoids shell-quoting breakage).
- Don't invent rationale. Reconstructed entries get a `## Provenance` section stating they
  were rebuilt from chain state, with the *Why* marked inferred pending the proposer's notes.

## Governance addresses & roles

| Role | Holder | Delay |
|---|---|---|
| `ADMIN_ROLE` — all config, upgrades, unpause, oracle feeds, adapter/strategy mgmt | Admin timelock `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` | 48h |
| Timelock `PROPOSER` + `CANCELLER` | DAO Safe `0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9` | — |
| Timelock `EXECUTOR` | `address(0)` — anyone | — |
| `SECURITY_ROLE` — pause, emergency recovery, cancel, instant removes | Security Safe `0x7c128C1CF39822B4133F8E067F4bF14999c49846` | none |

`ADMIN_ROLE = keccak256("ADMIN_ROLE") = 0xa49807205ce4d355092ef5a8a18f56e8913cf4a201fbe287825b095693c21775`.
Access is checked via `Registry.hasRole(role, caller)` (the Registry is the auth hub);
`onlyAuthRole` reverts `RegistryClientMissingRole(bytes32)` (`0x4d616cff`).

Protocol contract addresses live in the root `README.md` `## Mainnet addresses` table.
Protocol source is the sibling repo `../contracts` (foundry). Before writing a proposal,
grep it for the target function's modifier, every precondition/`require`/custom error, the
emitted event, and any dependency.

## Timelock ABI

```
schedule(address target, uint256 value, bytes data, bytes32 predecessor, bytes32 salt, uint256 delay)  // 0x01d5062a
execute (address target, uint256 value, bytes payload, bytes32 predecessor, bytes32 salt)               // 0x134008d3, payable
hashOperation(address,uint256,bytes,bytes32,bytes32) view returns (bytes32)
getOperationState(bytes32) returns (uint8)   // 0 Unset · 1 Waiting · 2 Ready · 3 Done
isOperationPending/Ready/Done(bytes32) · getTimestamp(bytes32)   // getTimestamp == 1 ⇒ Done sentinel
cancel(bytes32)   // DAO Safe or Security Safe
```
`delay` is `172800` (the enforced minimum) unless a longer delay is explicitly wanted. A
batch of scheduled ops is one Safe tx with N `schedule` entries in `transactions` — the
Safe UI wraps them via `MultiSendCallOnly` 1.4.1 `0x9641d764fc13c8B624c04430C7356C1C7C8102e2`.

## Building the payloads

```sh
INNER=$(cast calldata "<funcSig>" <args...>)                 # the protocol call
SALT=$(cast keccak "everstrat/<area>/<detail>/<yyyy-mm-dd>")
# operation id — recompute locally; MUST be exactly 66 chars incl 0x (002's recorded id
# was 62 chars, a transcription error that cast rejected):
cast keccak $(cast abi-encode "f(address,uint256,bytes,bytes32,bytes32)" <target> 0 $INNER <predecessor> $SALT)
# raw schedule calldata for 01-schedule-raw.json:
cast calldata "schedule(address,uint256,bytes,bytes32,bytes32,uint256)" <target> 0 $INNER <predecessor> $SALT 172800
```

Copy the JSON files from the most recent `NNN-slug/` and swap `target` / `data` /
`predecessor` / `salt` / meta strings. Then validate:
`for f in NNN-*/*.json; do python3 -m json.tool "$f" >/dev/null && echo OK $f; done`
and diff `01-schedule-raw.json`'s `data` against a fresh `cast calldata`.

## Simulating on a mainnet fork (do this before signing)

```sh
anvil --fork-url https://ethereum-rpc.publicnode.com --port 8546 --silent &   # publicnode: reliable free RPC
R=http://localhost:8546
TL=0xF0911198Ef0a4b4234546fa5F50d6d1D45091774
DAO=0x1780C78eB50cD28dC349CEA8452eD1F7206D8fF9
EOA=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266     # anvil acct 0 — holds no protocol role

cast rpc anvil_impersonateAccount $DAO --rpc-url $R
cast rpc anvil_setBalance $DAO 0xde0b6b3a7640000 --rpc-url $R

# a) direct call, non-admin EOA          -> RegistryClientMissingRole 0x4d616cff
# b) direct call, impersonated timelock  -> exercises the precondition revert, if any
# c) DAO Safe -> schedule(...)           -> success; CallScheduled + CallSalt
# d) execute(...) before delay           -> TimelockUnexpectedOperationState(op,4) 0x5ead8eb5
# e) cast rpc evm_increaseTime 172801 ; cast rpc evm_mine
# f) if predecessor set: execute now     -> TimelockUnexecutedPredecessor 0x90a9a618, then execute the dep first
# g) execute(...) from $EOA              -> success; target event emitted
# h) assert the getter flipped (isSupportedERC20 / isAdapterAllowed / connectorWeight / …)
# i) re-execute & re-schedule            -> 0x5ead8eb5 ; direct re-do -> AlreadyX custom error

pkill -f "anvil --fork-url"
```
Record gas and the exact revert selectors in the README `## Verification performed`
section. Time-warping makes Chainlink feeds "stale", but set-membership getters
(`isTokenSupported`, `isAdapterAllowed`) don't read feeds, so add/remove paths still
simulate cleanly. If your op *does* read a feed (NAV, quotes), use
`evm_setNextBlockTimestamp` to land just past `scheduled + delay` instead of a big jump.

## Verifying on-chain (status updates / reconstructions)

`https://ethereum-rpc.publicnode.com` serves current state and recent blocks;
`cast logs` over old ranges needs an archive node, so use Blockscout for logs (no key):

```sh
OP=$(cast call $TL "hashOperation(address,uint256,bytes,bytes32,bytes32)(bytes32)" <target> 0 <inner> <pred> <salt> --rpc-url $RPC)
cast call $TL "getOperationState(bytes32)(uint8)" $OP --rpc-url $RPC        # 3 = Done
curl -sSL "https://eth.blockscout.com/api?module=logs&action=getLogs&fromBlock=<n>&toBlock=latest&address=$TL&topic0=<CallScheduled|CallExecuted topic>&topic1=$OP"
```
`transactionHash` + `timeStamp` (hex) come back in the result; `cast tx <hash>` gives block
and sender. Decode a DAO Safe batch from `cast tx <hash>` input, or pull it from the Safe
tx service:
`curl -sSL "https://safe-transaction-mainnet.safe.global/api/v1/safes/<safe>/multisig-transactions/?nonce=<n>"`
(`dataDecoded.parameters[0].valueDecoded` lists the inner txs). Always recompute the
operation id and compare byte-for-byte to what the repo records.

## Status lifecycle

- **Draft** — files written + simulated, not signed. `📝 Draft — not yet submitted.`
- **Scheduled** — `schedule` mined. `⏳ Scheduled on mainnet (tx …, block …, <UTC>)`;
  add ready-at = `getTimestamp` = scheduled + delay, note any pending predecessor, add an
  `## On-chain schedule` section, confirm the tx calldata == `01-schedule.json` byte-for-byte.
- **Executed** — `execute` mined / `getOperationState == 3`.
  `✅ Executed on mainnet (tx …, block …, <UTC>)` plus the observable proof
  (`connectorWeight() == 9e17`, `getContractByKey("EVE") == 0x…`, `isAdapterAllowed(...) == true`).
  Update the root README row too.

## Selectors / errors / event topics seen here

| Selector | Signature |
|---|---|
| `0x01d5062a` | `schedule(address,uint256,bytes,bytes32,bytes32,uint256)` |
| `0x134008d3` | `execute(address,uint256,bytes,bytes32,bytes32)` |
| `0x1b710e97` | `AMM.setConnectorWeight(uint256)` |
| `0x645c6fae` | `Registry.registerContract(bytes32,address)` |
| `0x8eff1c3c` | `Oracle.updateUsdFeedInfo(address,address,uint256)` |
| `0xd73acee5` / `0x2dbc46f3` | `StrategyManager.addSupportedERC20(address)` / `removeSupportedERC20(address)` |
| `0x73721fe9` | `Converter.setAllowedAdapter(address,bool)` |

| Error | Meaning |
|---|---|
| `0x5ead8eb5` | `TimelockUnexpectedOperationState(bytes32,bytes32)` — 2nd arg bitmask; `…04` = Waiting (too early) |
| `0x90a9a618` | `TimelockUnexecutedPredecessor(bytes32)` |
| `0x4d616cff` | `RegistryClientMissingRole(bytes32)` |
| `0x0eb217c5` | `StrategyManagerERC20NotPriceable(address)` — Oracle can't price the token yet |
| `0x04a77d9d` / `0x29bcb2f9` | `StrategyManagerERC20AlreadySupported(address)` / `ConverterAdapterAlreadyAllowed()` |

| Event topic0 | Event |
|---|---|
| `0x4cf4410cc57040e44862ef0f45f3dd5a5e02db8eb8add648d4b0e236f1d07dca` | `CallScheduled` |
| `0x20fda5fd27a1ea7bf5b9567f143ac5470bb059374a27e8f67cb44f946f6d0387` | `CallSalt` |
| `0xc2617efa69bab66782fa219543714338489c4e9e178271560a91b82c3f612b58` | `CallExecuted` |

## Gotchas

- Operation ids are `bytes32` — exactly 66 chars incl `0x`. Recompute and count.
- `eth.llamarpc.com` / `cloudflare-eth.com` often fail; `rpc.ankr.com` and archive
  endpoints want a key. `ethereum-rpc.publicnode.com` is the reliable free default.
- `StrategyManager.addSupportedERC20(token)` requires `Oracle.isTokenSupported(token)` — a
  USD feed must be registered first. `UniswapV3ConverterAdapter` maps `weth → address(0)`
  for Oracle lookups, so a WETH↔X route needs the native-ETH (`address(0)`) feed too, not
  just WETH.
- Feed adds / `setAllowedAdapter` / `addSupportedERC20` are `ADMIN_ROLE`, 48h. Removes are
  often `ADMIN_ROLE || SECURITY_ROLE` with SECURITY having no delay — check the source.
