# 008 — Oracle: add jitter margin to WETH/ETH and USDC staleness bounds

**Status:** ⏳ **Scheduled on mainnet** (tx
`0x4be3e8c7bec7da0297352928cd660f6d71f064ece5603e95123264fa2afd4180`, block 25968101,
2026-09-13 11:05:47 UTC). Timelock state Waiting for all three ops. Executable from
**2026-09-15 11:05:47 UTC**.
**Operation ids:**
- WETH staleness: `0xc6360fe9cd159fb18d1436bb65e431afeb8909c3c0a03b431f2b8cd08abaada9`
- Native-ETH (`address(0)`) staleness: `0x209d4394819de0a3c5994870976244f2945004b217efe7a1beaf5492ad272166`
- USDC staleness: `0x74253eb43d0d22fbe52315a2b134c38b6de73ba499c0177bb339b558840d93fe`

All three recomputable via `hashOperation`, verified on mainnet.

## What it changes

Three calls to `Oracle.updateUsdFeedInfo(address token, address feed, uint256 staleness)`
on `0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26`, **re-passing the existing feed address**
for each token and changing only `stalenessInterval`:

| Token | Feed (unchanged) | Staleness: before → after | Chainlink heartbeat |
|---|---|---|---|
| WETH `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` | `0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419` (ETH/USD) | `3600` → **`4200`** | 3600 (1h) |
| `address(0)` (native ETH) | `0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419` (ETH/USD) | `3600` → **`4200`** | 3600 (1h) |
| USDC `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | `0x8fFfFfd4AfB6115b954Bd326cbe7B4BA576818f6` (USDC/USD) | `82800` → **`84600`** | 82800 (23h) |

Because the feed address is unchanged for all three, `_upsertFeed` takes the update path
(not the first-add path): each call emits `UsdStalenessIntervalUpdated(token, old, new)`
only — no re-check of feed decimals, no membership change (`isTokenSupported` stays
`true` for all three throughout).

**What it does NOT do:** touches no other registry key, no role, no WETH/USDC/native-ETH
feed *address* (only the staleness number), no other token (USDT from
[006](../006-oracle-usdt-feed/) is untouched — its `86400` bound is a separate proposal if
it needs the same treatment once it's live).

## Why

All three feeds were registered ([003](../003-oracle-usd-feeds/)) with `stalenessInterval`
set to **exactly** the feed's own heartbeat — WETH and `address(0)` at `3600` (1h), USDC at
`82800` (23h) — with zero jitter margin in every case. A Chainlink round that lands even a
second past its nominal heartbeat (ordinary block-time variance, not a stalled feed) makes
`getUsdPrice`/`getUsdPriceWithStalenessCheck` revert with `OracleStalePrice` even though the
feed is healthy. This proposal adds margin above each heartbeat: `+600` (10 min) for the
two 1h-heartbeat feeds, `+1800` (30 min) for USDC's 23h-heartbeat feed — enough to absorb
normal jitter without materially loosening the check (a feed that actually stops updating
still fails closed well within the added margin).

**Timing:** proposed independent of any strategy activity — this only changes how tolerant
three already-live feeds are to normal timing variance, not what they price or whether
they're supported.

## No predecessor (×3)

Each `updateUsdFeedInfo` call only checks `ADMIN_ROLE`, `_priceFeed != address(0)`,
`_stalenessInterval != 0`, and (only on a feed-address change, which none of these are)
`feed.decimals() <= 18`. None of the three depends on another timelock operation or on
each other — all three `predecessor` fields are `0x00…00`.

## Transactions

| # | File | From | Calls |
|---|---|---|---|
| 1 | `01-schedule.json` | DAO Safe | `timelock.schedule(...)` ×3 (one batch), `delay = 172800` |
| 2 | `02-execute.json` | anyone | `timelock.execute(...)` ×3 after the delay |

The three `schedule` calls are meant to go out as a single Safe batch (Transaction
Builder / `MultiSendCallOnly`), same shape as 003's nonce-6 batch. `01-schedule-raw.json`
is the same three transactions as raw calldata (selector `0x01d5062a`).

### Parameters

Common to all three:

| Field | Value |
|---|---|
| `to` | `0xF0911198Ef0a4b4234546fa5F50d6d1D45091774` (admin timelock) |
| `target` | `0xF5B0C0ab00F92f6B8DC5F6314507FACC9c610c26` (Oracle) |
| `value` | `0` |
| `predecessor` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| `delay` | `172800` (48h) |

Per-operation `data` (`updateUsdFeedInfo(address,address,uint256)`, selector `0x8eff1c3c`)
and salt:

| Operation | `data` | `salt` |
|---|---|---|
| WETH | `0x8eff1c3c000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc20000000000000000000000005f4ec3df9cbd43714fe2740f5e3616155c5b84190000000000000000000000000000000000000000000000000000000000001068` | `0x88df0cbed0092f90974fa0fe93cab84e685c9cf84f41e78cfd848335a906615a` |
| `address(0)` | `0x8eff1c3c00000000000000000000000000000000000000000000000000000000000000000000000000000000000000005f4ec3df9cbd43714fe2740f5e3616155c5b84190000000000000000000000000000000000000000000000000000000000001068` | `0xf3be49b7996241345a11e359ac0db97d73ec34ae48cf6a833ea59cb0e946d57f` |
| USDC | `0x8eff1c3c000000000000000000000000a0b86991c6218b36c1d19d4a2e9eb0ce3606eb480000000000000000000000008fffffd4afb6115b954bd326cbe7b4ba576818f60000000000000000000000000000000000000000000000000000000000014a78` | `0x825cf1a3ac0260f5a4598503b9fd946d9fd68332aeba257cf3b7d48945aa54e2` |

Salts derive from `keccak256("everstrat/oracle/usd-feed/<weth|native-eth|usdc>-staleness/2026-09-12")`
— real, distinct per-operation salts (003's original batch used a zero salt for both its
operations; this proposal does not repeat that). Predecessor and salt must be reused
**byte-for-byte** at execute time, per operation.

## Verification performed (forked mainnet, before signing)

Fork at block 25962915. Starting state: `getUsdFeedInfo(WETH) == (0x5f4e…8419, 3600)`,
`getUsdFeedInfo(address(0)) == (0x5f4e…8419, 3600)`, `getUsdFeedInfo(USDC) == (0x8fFf…18F6, 82800)`.

1. `hashOperation` for each of the three (target Oracle, value 0, respective `data`, zero
   predecessor, respective salt) reproduces the three operation ids above.
2. Direct `updateUsdFeedInfo(WETH, …)` from an unrelated EOA — **reverts**
   `RegistryClientMissingRole(ADMIN_ROLE)` (`0x4d616cff`).
3. DAO Safe → `schedule` ×3 with the exact calldata above — **all succeed**; `CallScheduled`
   + `CallSalt` emitted for each op id.
4. `execute` (WETH op) before the delay — **reverts** `TimelockUnexpectedOperationState`
   (`0x5ead8eb5`).
5. Warp 48h + 1s. `execute` **from an unrelated EOA** for all three — **all succeed**
   (61,856 / 61,616 / 61,868 gas respectively — cheap: same feed address, staleness-only
   update, no `_validateFeedDecimals` re-check).
6. Post-state: `getUsdFeedInfo(WETH) == (0x5f4e…8419, 4200)`,
   `getUsdFeedInfo(address(0)) == (0x5f4e…8419, 4200)`,
   `getUsdFeedInfo(USDC) == (0x8fFf…18F6, 84600)`. `isTokenSupported` unchanged (`true`) for
   all three throughout.
7. Re-`execute` (WETH op) — reverts `TimelockUnexpectedOperationState` (`0x5ead8eb5`); a
   direct identical re-update (same feed, same staleness) from the timelock reverts
   `OracleNothingToUpdate` (`0xf00155ca`).

`getUsdPrice(WETH)` / `getUsdPrice(USDC)` revert `OracleStalePrice` (`0xa9f73445`) on the
warped fork immediately after execute — the 48h time jump outruns even the widened bounds.
Fork artifact only: on mainnet the underlying Chainlink rounds are fresh (both feeds
verified `latestRoundData` within their heartbeat at proposal time).

## On-chain schedule (mainnet)

Scheduled 2026-09-13 11:05:47 UTC in tx
`0x4be3e8c7bec7da0297352928cd660f6d71f064ece5603e95123264fa2afd4180` (block 25968101),
DAO Safe (Transaction Builder, `multiSend` via `MultiSendCallOnly`, nonce 17) → timelock
`schedule` ×3. Confirmed against mainnet (Safe tx service `dataDecoded`):

- The three `schedule` calls in the tx match `01-schedule.json` byte-for-byte: same
  target (Oracle), `data` (`0x8eff1c3c…`), zero predecessor, and salts
  `0x88df0cbe…906615a` (WETH), `0xf3be49b7…6d57f` (`address(0)`), `0x825cf1a3…a54e2`
  (USDC), all with `delay = 172800`.
- `getOperationState` → `1` (Waiting) for all three op ids; `isOperationPending` → `true`.
- `getTimestamp` → `1789470347` = **2026-09-15 11:05:47 UTC** (ready-at) for all three.

## Cancelling

Either the DAO Safe or the Security Safe may `cancel(<operation id>)` on the timelock for
any of the three operations independently, any time before that operation executes:

- `cancel(0xc6360fe9cd159fb18d1436bb65e431afeb8909c3c0a03b431f2b8cd08abaada9)` — WETH
- `cancel(0x209d4394819de0a3c5994870976244f2945004b217efe7a1beaf5492ad272166)` — `address(0)`
- `cancel(0x74253eb43d0d22fbe52315a2b134c38b6de73ba499c0177bb339b558840d93fe)` — USDC

After execution, `updateUsdFeedInfo` again to change any bound further.
