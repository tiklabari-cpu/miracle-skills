---
name: onchain-transfer-tracking
description: Reliably track incoming on-chain crypto transfers to a receiving address by polling a block explorer's REST API (TronGrid/Etherscan-style) — a self-scheduling poller, an in-memory block-timestamp watermark with an overlap re-scan, confirmed-only finality, and txid-based idempotency so the overlap and restarts never double-process. No websockets, no extra npm deps, read-only (no private key). Use when you need to watch a wallet/contract for deposits and react to each new transfer exactly once, on Node/Express. The monitoring engine underneath the usdt-trc20-payments skill; pairs with credit-wallet-billing (what to do when a transfer lands). Reusable pattern from Tıklatbari.
---

# On-chain Transfer Tracking (poll → detect → process-once)

Watch a receiving address for incoming transfers and react to **each new confirmed transfer exactly
once**, using nothing but periodic REST reads of a block explorer. No websockets, no provider SDK, no
private key on the server. The hard part isn't reading the chain — it's **not missing a transfer and
not processing one twice** across ticks, overlaps, and restarts. Node/Express + `better-sqlite3`.

## Why polling (and why `setTimeout`-chained, not `setInterval`)
- **Polling, not webhooks:** no public endpoint to secure, no provider account, works behind NAT. A
  read-only REST call to a public explorer is enough.
- **Self-scheduling tick** — schedule the *next* tick only after the current one finishes:
  ```js
  function startPoller() {
    if (!isConfigured()) return;                       // no receiving address → don't run
    const tick = async () => {
      try { expireStale(); await pollChain(); }
      catch (e) { console.error('[poll]', e.message); }
      finally { setTimeout(tick, POLL_INTERVAL_MS); }  // reschedule AFTER completion
    };
    setTimeout(tick, 5000);                            // let the server finish booting first
  }
  ```
  `setInterval` would fire again while a slow/hung request is still in flight → overlapping ticks,
  piled-up timers, doubled work. The chained `setTimeout` guarantees exactly one tick at a time and
  that errors don't accumulate. Interval floor `~30s` (explorers rate-limit).

## The watermark (cursor) + overlap re-scan
Keep the newest block timestamp you've processed in memory and only ask for transfers *after* it —
minus a small overlap so a boundary transfer is never skipped:
```js
let lastScannedMs = 0;                                 // in-memory high-water mark
async function pollChain() {
  const from = lastScannedMs > 0
    ? lastScannedMs - 60_000                           // re-scan the last 60s (overlap safety)
    : Date.now() - TTL_MINUTES * 60_000;               // first tick: look back one TTL window
  const transfers = await fetchTransfers(from);        // ascending by block_timestamp
  let maxTs = lastScannedMs;
  for (const tx of transfers) {
    if (tx.block_timestamp > maxTs) maxTs = tx.block_timestamp;
    try { processTransfer(tx); } catch (e) { console.error(e.message); }
  }
  if (maxTs > lastScannedMs) lastScannedMs = maxTs;    // advance only forward
}
```
- **Fetch ascending** (`order_by=block_timestamp,asc`) so you process oldest-first and the watermark
  advances monotonically.
- **The overlap deliberately re-reads recent transfers.** That's fine — **idempotency (below) makes
  re-processing a no-op.** Overlap prevents the off-by-one miss when two transfers share a boundary.
- **Never rescan full history:** `from` bounds the query; the watermark moves forward each tick.

## Idempotency is the load-bearing wall
The overlap re-scan, provider retries, and restarts all re-deliver transfers you've already handled.
The **only** thing that makes that safe is a unique key on the transfer id + a compare-and-swap:
```sql
-- schema: txid TEXT UNIQUE
UPDATE payments SET status='paid', txid=?, ... WHERE id=? AND status='pending';  -- CAS
```
Check `changes === 1` before applying the side effect (crediting, fulfilling). A second delivery of the
same `txid` finds the row already `paid` (or hits the UNIQUE constraint) → `changes === 0` → nothing
happens. **Design every "on transfer" side effect to be idempotent on `txid`**; then over-delivery is
harmless and you can be generous with the overlap window.

## Reading the explorer (TronGrid shape; Etherscan/Alchemy analogous)
```js
const url = new URL(`https://api.trongrid.io/v1/accounts/${WALLET}/transactions/trc20`);
url.searchParams.set('only_to', 'true');               // only incoming (to our address)
url.searchParams.set('only_confirmed', 'true');        // FINALITY — see gotcha
url.searchParams.set('limit', '50');
url.searchParams.set('order_by', 'block_timestamp,asc');
if (minTimestampMs > 0) url.searchParams.set('min_timestamp', String(minTimestampMs));
const headers = { accept: 'application/json' };
if (API_KEY) headers['TRON-PRO-API-KEY'] = API_KEY;    // mainnet wants a key; testnets don't
const body = await (await fetch(url, { headers })).json();
return Array.isArray(body?.data) ? body.data : [];
```
Node `fetch`, **zero extra npm packages**. Fields used: `token_info.address` (contract), `to`, `value`
(string micro-units), `transaction_id` (the idempotency key), `block_timestamp` (the watermark source).

## Per-transfer processing order (security first, then match)
1. **Contract whitelist** — `token_info.address === EXPECTED_CONTRACT` (the hash, not the symbol —
   fake tokens reuse names). Kills the fake-token deposit.
2. **Recipient** — `tx.to === WALLET`.
3. **Finality** — trust `only_confirmed=true`; do not gate on a per-tx field (gotcha #1).
4. **Match** the transfer to what you're expecting (an invoice/amount), then apply idempotently (CAS).
Log skipped transfers (`unmatched-amount`, `wrong-contract`) for manual **reconciliation**.

## Restart & single-instance behavior
- The watermark is **in memory**, so a restart resets it → the next first tick re-scans the last TTL
  window. Idempotency makes that a safe no-op (no double-processing). If you want to skip the rescan,
  persist `lastScannedMs`; usually not worth it.
- **One poller per system.** This assumes a single process. Multiple instances each polling will each
  try to process every transfer — safe *only* because of `txid` idempotency, but wasteful and racy on
  the CAS; for multi-instance, elect a leader or run the poller as one dedicated worker.

## Extending to other chains
Same engine, different read: **ETH/ERC-20** drops in cleanly (Etherscan/Alchemy `tokentx` by address,
`confirmations` field for finality, block-number watermark instead of timestamp). **UTXO chains
(BTC/LTC)** need a different model — funds arrive at addresses/outputs, not a token-transfer log — so
track by derived address or scan outputs; the poller/watermark/idempotency shape still applies, the
"match" step changes.

## Gotchas
1. **TronGrid's `/transactions/trc20` returns no per-tx `confirmed` field** (it's `undefined`). Gating
   on `tx.confirmed !== true` rejects **everything**. Use the `only_confirmed=true` query param for
   finality and trust the returned rows. (Found only by hitting the live API — mocks hid it.)
2. **Advance the watermark from `block_timestamp`, not wall-clock.** Using `Date.now()` as the cursor
   can skip transfers whose block time lags the request.
3. **Overlap must be ≥ the explorer's indexing lag / clock skew** (60s is generous for Tron). Too small
   and a boundary transfer slips through between ticks.
4. **Idempotency must key on the immutable `txid`**, not on amount or order — amounts repeat, txids
   don't. Without the UNIQUE+CAS, the overlap re-scan double-processes on the very first tick.
5. **A tick must never throw out of the loop.** Wrap each `processTransfer` in try/catch so one bad
   transfer doesn't abort the batch or stop the schedule.

## Verification
- Boot on a temp `DB_PATH` with a fake `fetchTransfers` returning a fixed list. Run `pollChain()` twice
  → assert side effects apply **once** (idempotency), and `lastScannedMs` advanced to the max
  `block_timestamp`. Feed an overlapping second batch (repeats + one new) → only the new one processes.
- Assert a `wrong-contract` / `wrong-to` transfer is skipped and never advances a credit.
- **Live read-only smoke test:** actually fetch the receiving address's real confirmed transfers from
  the explorer and assert the parser returns the expected fields — external-API shape bugs (gotcha #1)
  only surface against the live endpoint, never against a mock.
