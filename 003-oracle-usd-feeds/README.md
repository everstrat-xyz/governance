# 003 — Oracle USD price feeds: add USDC and WETH

**Status:** 🟢 **Ready for execution on mainnet** (scheduled tx `0xbd213e2b75329bb4a885916d604a7c4bb6fdbd19a11ae2c3996a9c9e80b469e6`, block 25907176, 2026-09-04 23:14:59 UTC). 48h delay elapsed **2026-09-06 23:14:59 UTC** — both operations verified `state=2 (Ready)` on-chain 2026-09-07 11:37 UTC. **Execute pending** — see `02-execute.json`. (batch tx `0xbd213e2b75329bb4a885916d604a7c4bb6fdbd19a11ae2c3996a9c9e80b469e6`, block 25907176, 2026-09-04 23:14:59 UTC — DAO Safe nonce 6). Both operations executable from **2026-09-06 23:14:59 UTC** — permissionless execute.
**Operation ids:**
- USDC feed: `0xd57312a1b34a92fa9799b8467c6e49733f334d0a83aefbfad9980f1a8b7ed5a1`
- WETH feed: `0xa123b8437d97fc2766453aca6597121248e88eca21925e1f9fc455b16d1d1f06`

Both recomputable via `hashOperation`, verified on mainnet.

## Provenance

This entry was reconstructed from DAO Safe transaction **nonce 6** after it was signed
and executed on-chain — it was not drafted here in advance. The calldata, operation ids
and status are read back from mainnet. The reasoning in *Why* is inferred from the
on-chain parameters; confirm it against the proposer's notes.

## What it changes

Two calls to `Oracle.updateUsdFeedInfo(address token, address feed, uint256 staleness)`
(selector `0x8eff1c3c`) on the price oracle
`0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26`, registering Chainlink USD reference feeds
for the two assets the protocol prices:

| Token | Feed | Feed pair | `staleness` |
|---|---|---|---|
| USDC `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | `0x8fFfFfd4AfB6115b954Bd326cbe7B4BA576818f6` | USDC / USD | `82800` (23h) |
| WETH `0xC02aaa39b223FE8D0A0e5C4F27eAD9083C756Cc2` | `0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419` | ETH / USD | `3600` (1h) |

`staleness` is the maximum age of a Chainlink `latestRoundData` answer the oracle will
accept before treating the price as stale. Each value tracks that feed's published
heartbeat (USDC/USD 24h, ETH/USD 1h) with a margin below it.

Before this change, `Oracle.getUsdFeedInfo(USDC)` and `getUsdFeedInfo(WETH)` both revert
(`0x868fd74e`) — no feed is configured for either asset.

**What it does NOT do:** touches no registry key, no role, no other asset. It adds two
feeds; it removes and rewrites nothing.

## Why

The oracle needs a USD reference for every asset it values. USDC and WETH are the
protocol's priced assets, and the two registered feeds are Chainlink's canonical mainnet
aggregators for `USDC / USD` and `ETH / USD`. The staleness bounds are set just under
each feed's heartbeat so a feed that stops updating fails closed rather than serving a
frozen price.

**Timing:** scheduled while the protocol is unbootstrapped — no position depends on the
oracle yet — so registering the feeds now has no effect on any user.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` ×2 (one batch), `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` ×2 after the delay |

The DAO Safe sent both `schedule` calls as a single batch via `MultiSendCallOnly`
`0x9641d764fc13c8B624c04430C7356C1C7C8102e2`. `01-schedule-raw.json` is the same pair as
raw calldata (selector `0x01d5062a`).

### Parameters

Common to both operations:

| Field | Value |
|---|---|
| `to` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (admin timelock) |
| `target` | `0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26` (Oracle) |
| `value` | `0` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `salt` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `delay` | `172800` (48h) |

Per-operation inner `data` — `updateUsdFeedInfo(address,address,uint256)`:

| Operation | `data` |
|---|---|
| USDC | `0x8eff1c3c000000000000000000000000a0b86991c6218b36c1d19d4a2e9eb0ce3606eb480000000000000000000000008fffffd4afb6115b954bd326cbe7b4ba576818f60000000000000000000000000000000000000000000000000000000000014370` |
| WETH | `0x8eff1c3c000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc20000000000000000000000005f4ec3df9cbd43714fe2740f5e3616155c5b84190000000000000000000000000000000000000000000000000000000000000e10` |

Unlike 001 and 002, both operations use a **zero salt** rather than a
`keccak256("everstrat/…")` value. Functionally fine — the two operation ids are distinct
because their calldata differs — but it departs from the repo's salt convention and
carries no proposal-specific fingerprint. The zero salt must still be reused
byte-for-byte at execute time.

## Verification performed (mainnet, after schedule)

1. DAO Safe nonce 6 → `multiSend` of two `schedule` calls — executed, status 1, block 25907176.
2. `timelock.hashOperation(Oracle, 0, <data>, 0, 0)` reproduces both operation ids above.
3. `getOperationState` → `1` (Waiting) for both; `getTimestamp` → `1788736499`
   (2026-09-06 23:14:59 UTC) for both.
4. `Oracle.getUsdFeedInfo(USDC)` and `(WETH)` currently revert `0x868fd74e` — feeds not yet set.
5. Feed addresses match Chainlink's published mainnet aggregators for USDC/USD and ETH/USD.

After execution, re-check that `Oracle.getUsdFeedInfo(USDC|WETH)` returns
`(feed, staleness)` as tabled above.

## Cancelling

Either the DAO Safe or the Security Safe may `cancel(<operation id>)` on the timelock for
either operation independently, any time before that operation is executed:

- `cancel(0xd57312a1b34a92fa9799b8467c6e49733f334d0a83aefbfad9980f1a8b7ed5a1)` — USDC feed
- `cancel(0xa123b8437d97fc2766453aca6597121248e88eca21925e1f9fc455b16d1d1f06)` — WETH feed

## Verification performed

Re-verified from live mainnet state before execution (2026-09-07):

1. `getOperationState` = 2 (Ready) for both op-ids, `isOperationReady` = true.
2. Fork simulation of `execute` from unrelated EOA — success (status Done both ops).
3. Post-execute Oracle state asserted: USDC feed + 82800, WETH feed + 3600, both supported.

Salt `0x00`, predecessor `0x00` — must match schedule; `02-execute.json` is ready for the Safe Transaction Builder.
