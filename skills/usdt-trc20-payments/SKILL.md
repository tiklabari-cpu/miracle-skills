---
name: usdt-trc20-payments
description: Self-custody USDT (TRC-20) crypto payment + credit top-up without a payment provider — TronGrid REST verification, no extra npm deps, private key never on the server. Covers multi-user order matching via a lock-free unique-amount salt reservation (no fee), plus the polished two-column payment UI (demo instant-confirm mode + real on-chain verified mode with pending/paid/expired states, black QR, cancel). Use when adding "pay with crypto" / credit loading, or solving "which order does this incoming transfer belong to", to a Node/Express + React app. Reusable pattern from Tıklatbari.
---

# USDT TRC-20 Payments (self-custody, no provider)

Verify incoming USDT (TRC-20 / Tron) payments **yourself** — no NOWPayments/Cryptomus, no commission,
no third-party dependency. Node/Express + `better-sqlite3` + React. **The Gotchas section contains
two bugs that will silently break the whole thing — read it before shipping.**

## Architecture decisions
- **TronGrid REST**, Node `fetch`, **zero extra npm packages** (no tronweb).
- **Private key is NOT on the server.** The server only knows the *receiving address* (cold wallet).
  If the server is compromised, funds are still safe.
- **Not a separate process/service** — a `services/payments.js` module + a `setTimeout`-chained poller
  started from `server.js`. (Single-instance MVP; a separate process adds no security.)
- One network only (**TRC-20**) keeps the surface small: ~$0.5 fee, ~3s blocks, ~1min finality.

## Two UI modes (same visual language)
1. **Demo mode** (`PaymentPage`): plan upgrade / credit top-up that credits *instantly* on an
   "Ödemeyi Yaptım" click. For prototypes only — trusts the client. Never ship as real.
2. **Real verified mode** (`TestOdemePage`): user gets an invoice, sends exactly that amount on-chain,
   a poller detects it via TronGrid and credits automatically. This is the production path.

## Schema (`payments`)
```
id, user_id, order_ref TEXT UNIQUE, network TEXT ('TRC20'), address TEXT,
service_amount_micro INT, fee_amount_micro INT, expected_amount_micro INT,   -- USDT*1e6
received_amount_micro INT, txid TEXT UNIQUE, confirmations INT,
status TEXT ('pending'|'paid'|'expired'), expires_at, created_at, paid_at
```
Credits balance lives on `users.credits` (REAL). `txid UNIQUE` is the idempotency backbone.
The two amount columns are **deliberately distinct**: `service_amount_micro` = the credit granted
(the round amount the user typed); `expected_amount_micro` = the amount to send / match the chain on
(= `service` + salt, or `service` + fee if you charge one). In the no-fee salt model `fee_amount_micro`
stays 0 and `expected = service + salt`.

## Matching an incoming payment to an order
TRC-20 has **no memo/tag field** — you cannot attach an order id, so the **amount is the only
discriminator**. Three strategies, cheapest first:
- **Exact amount** (simplest): `expected_amount_micro = amount*1e6`; match incoming value to the
  oldest pending invoice. Fine for single-user/test; at scale two identical-amount pending invoices
  mis-match — A pays, B gets credited (the amount can't tell them apart).
- **Unique-amount salt** (the multi-user default — detailed below): reserve a per-order micro salt so
  every *currently pending* amount is unique, so an incoming transfer binds to exactly one order.
- **Scale answer:** HD wallet, one derived address per order (real isolation; needs key management).

### Unique-amount salt reservation (no fee) — the production matcher
Split the amount into two numbers on the row (see Schema): **`service_amount_micro`** = the round
amount the user typed = **the credit they receive**; **`expected_amount_micro` = `service + salt`**
(salt ∈ `0..999` micro) = **the exact amount they must SEND and what the chain match keys on**. Salt
starts at **0**, so the first / uncontended `$10` invoice is the round `10.000000`; only a *concurrent*
second `$10` invoice becomes `10.000001`. The ≤ `$0.000999` salt is **never credited** — it's purely a
matching id (the user very slightly overpays it, by design).

`createInvoice(userId, amountUsd)` — reserve the first free salt, then INSERT:
```js
const baseMicro = Math.round(amountUsd * 1e6);              // = credit target (service_amount_micro)
// amounts already reserved by *currently pending* (non-expired) invoices in [base, base+999]:
const taken = new Set(db.prepare(
  `SELECT expected_amount_micro FROM payments
     WHERE status='pending' AND expires_at > datetime('now')
       AND expected_amount_micro BETWEEN ? AND ?`).all(baseMicro, baseMicro + 999)
  .map(r => r.expected_amount_micro));
let expectedMicro = -1;
for (let salt = 0; salt <= 999; salt++) {                   // first free salt (0 → round amount)
  if (!taken.has(baseMicro + salt)) { expectedMicro = baseMicro + salt; break; }
}
if (expectedMicro < 0) { const e = new Error('too many pending payments for this amount, retry'); e.status = 503; throw e; }
db.prepare(`INSERT INTO payments (user_id, order_ref, network, address,
    service_amount_micro, fee_amount_micro, expected_amount_micro, expires_at)
  VALUES (?,?,'TRC20',?,?,0,?,?)`).run(userId, orderRef, WALLET, baseMicro, expectedMicro, expiresAtIso);
```
Then `buildInvoicePayload` exposes **both** numbers to the UI: `amount_usdt` = `microToUsdt(expected)`
(the SEND value, salt included) **and** `credit_usdt` = `microToUsdt(service)` (the round LOAD value).
They're equal unless salt>0.

**Why it's race-safe without a lock:** `better-sqlite3` is **synchronous** — there is **no `await`
between the SELECT and the INSERT**, so in a single-process Node app two concurrent `createInvoice`
calls cannot interleave; they serialize and each picks a distinct salt. That's the whole reason no
queue/lock is needed for a single-instance MVP. **It breaks the moment you run 2+ processes** (each has
its own event loop): then add a DB-level guard — a partial UNIQUE index on `expected_amount_micro`
where `status='pending'`, or `INSERT … ON CONFLICT`, and retry on conflict. (See Gotcha #6.)

**Freed amounts recycle safely:** only `status='pending'` rows collide (the SELECT filters on it, and
`processTransfer` only matches pending), so a `paid`/`expired` invoice's amount is instantly reusable —
a settled `10.000000` returns to salt 0 for the next order. Inherent caveat: a *stray* duplicate
on-chain transfer of a reused round amount can then credit a *different* pending invoice — an
irreducible limit of amount-matching; the HD-wallet strategy is the only full fix.

**Only 1000 concurrent pending invoices can share a base amount.** The 1001st `$10` throws `503`;
surface it as a friendly "try again in a moment" (a pending frees on pay/expire). A different base
(`$11`) is unaffected.

## TronGrid read (`fetchTrc20Transfers`)
```js
const url = new URL(`https://api.trongrid.io/v1/accounts/${WALLET}/transactions/trc20`);
url.searchParams.set('only_to', 'true');
url.searchParams.set('only_confirmed', 'true');   // ← FINALITY GATE. See Gotcha #1.
url.searchParams.set('limit', '50');
url.searchParams.set('order_by', 'block_timestamp,asc');
if (minTimestampMs > 0) url.searchParams.set('min_timestamp', String(minTimestampMs));
const headers = { accept: 'application/json' };
if (API_KEY) headers['TRON-PRO-API-KEY'] = API_KEY;   // mainnet needs a key; testnets don't
const body = await (await fetch(url, { headers })).json();
return Array.isArray(body?.data) ? body.data : [];
```
Real tx fields: `token_info.address` (contract), `to`, `value` (string micro), `transaction_id`,
`block_timestamp`. (There is **no** per-tx `confirmed` field on this endpoint — Gotcha #1.)

## processTransfer security (order matters)
1. **Contract whitelist** — `tx.token_info.address === USDT_CONTRACT`
   (mainnet `TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t`). Check the **contract hash**, never the symbol/name
   (fake tokens reuse "USDT"). This one line kills the fake-token attack.
2. `tx.to === WALLET`.
3. Finality via `only_confirmed=true` (do NOT gate on `tx.confirmed !== true`; see Gotcha #1).
4. Match `Number(tx.value)` to the oldest pending invoice's `expected_amount_micro`. With the salt,
   that value is **unique among pending invoices**, so the transfer binds to exactly one order — no
   wrong-user credit. **Credit `service_amount_micro` (the round amount), not the received value** — the
   salt is not credited. (`processTransfer` is unchanged by the salt: it already matched on `expected`
   and credited `service`; the salt just makes `expected` unique.)
5. **Idempotent credit** in one transaction: `UPDATE payments SET status='paid', txid=? WHERE id=? AND
   status='pending'` (CAS) + `UPDATE users SET credits = credits + ?`. `txid UNIQUE` makes a second
   credit impossible even under retries/webhook duplicates.

## Poller + lifecycle
- `startPoller()` from `server.js`: `setTimeout`-chained tick (not `setInterval`, so errors don't
  pile up), every ~30s: `expireStale()` then `pollTron()`. Advance `min_timestamp` from the max
  `block_timestamp` seen so you don't rescan history (idempotency covers the overlap window).
- `cancelInvoice(orderRef, userId)`: user "Cancel" must flip status `pending→expired` **server-side**
  — otherwise it re-hydrates on reload and can still match a payment. The frontend cancel clears local
  state AND calls `POST /invoice/:ref/cancel`.
- `getActiveInvoice(userId)`: on page load, hydrate the latest pending invoice so the user doesn't
  re-enter the amount.

## Frontend (`TestOdemePage`)
Two-column (`md:grid-cols-5`, left `col-span-3` / right `col-span-2`). Left: method tabs
(Crypto active / Card "soon"), a single **USDT — TRC-20** badge (no coin selector — one network only),
a prominent **amber network warning**, address + **plain black QR** (`qr-code-styling`, `dotsOptions
type:'square' color:'#000'`, no logo — not the branded QR), and state blocks:
- **pending:** the amount to send as the hero (big, mono, copy button) + "waiting" box with MM:SS
  countdown, "check now", and "Cancel".
- **paid:** green "Ödeme Ulaştı", credited amount, txid, "New payment".
- **expired:** amber + "New invoice".
Auto-poll `GET /invoice/:ref` every 10s; on `pending→paid` call `refreshUser().catch(()=>{})`.
Show `walletInfo()` (`GET /billing/wallet`) so the address/QR render even before an invoice.

**Send vs credit (the salt UI contract).** The invoice carries two numbers — surface them without
confusing the user: **send** (`amount_usdt` = expected, salt included) is the **authoritative hero +
copy button** (the wallet must receive exactly this); **credit** (`credit_usdt` = service, round) is
what the paid screen ("X kredi yüklendi") and the order-summary "credit" line show. Copy the **send**
value, never credit. Render a small explanatory note **only when they differ** (`amt !== credit`, i.e.
salt>0): *"the last digits identify your order — send exactly this; you'll be credited {credit}."*
Normally they're equal so the note stays hidden and the page reads as a plain round-number payment.

## Env
`TRC20_WALLET_ADDRESS` (receive-only), `TRC20_USDT_CONTRACT`, `TRONGRID_API_KEY` (optional),
`TRC20_MIN_CONFIRMATIONS` (test 1 / mainnet 19), `PAYMENT_TTL_MINUTES`, `PAYMENT_POLL_INTERVAL_SECONDS`,
`PAYMENT_MIN_USD`, `PAYMENT_MAX_USD`. Keep `.env` out of git; there is no `.env` by default — the app
won't start the poller (and invoice creation 503s) until `TRC20_WALLET_ADDRESS` is set.

## Gotchas (each one silently breaks payments)
1. **TronGrid's `/transactions/trc20` endpoint does NOT return a `confirmed` field** (it's
   `undefined`). If you gate on `if (tx.confirmed !== true) skip`, `undefined !== true` is always
   true → **every payment is rejected, nobody is ever credited.** Filter via the **`only_confirmed=true`
   query param** instead; then trust the returned rows. Found only by hitting the live API.
2. **Mock harnesses hide #1** — the mock faked `confirmed:true`, so tests were green while the real
   integration was 100% broken. **Always verify a payment/external-API integration with a live
   read-only call** (fetch the receiving address's real transfers), not just mocks.
3. **Cancel must hit the server.** Clearing only local state leaves a live pending invoice that
   re-appears and can still capture a payment.
4. **Credit = `service_amount_micro`, not `received`.** Received = `expected` = `service + salt`
   (and + fee if you charge one), so with a salt the credited amount is *less* than what hit the chain
   by the salt (≤ $0.001). Always credit `service`; never credit `Number(tx.value)`. If `credit` and
   `send` can differ, show both in the UI (see Frontend) so the difference isn't a silent surprise.
5. **You cannot recover funds sent on the wrong network.** Make the "TRC-20 only" warning unmissable —
   this is a universal crypto truth, not something code can fix.
6. **Salt race-safety hinges on synchronous DB + a single process.** The lock-free `SELECT free salt →
   INSERT` is atomic *only* because `better-sqlite3` is synchronous and Node is single-threaded — no
   `await` between the two statements means concurrent `createInvoice` calls can't interleave. Run two
   Node processes (cluster, PM2 `-i`, k8s replicas) and each has its own event loop → two users can
   book the same `expected_amount_micro`. Before scaling out, add a **partial UNIQUE index** on
   `expected_amount_micro WHERE status='pending'` (or `INSERT … ON CONFLICT` + retry).

## Verification
- **Live E2E (the important one):** fetch a real confirmed transfer to the wallet, insert a pending
  invoice whose `expected_amount_micro` equals that transfer's `value`, run `processTransfer(realTx)`,
  assert it credits, records `txid`, is idempotent on repeat, and rejects a fake contract.
- **Concurrency / salt harness (run it, don't reason about it).** `better-sqlite3` compiles natively,
  so boot on a temp `DB_PATH` and exercise the real code — no mock. Set env before `require` (modules
  read `process.env` at load), seed users, then:
  - Fire 10 `createInvoice(u_i, 10)` via `Promise.all` → assert **10 distinct** `expected_amount_micro`
    (`10000000..10000009`), all with `service_amount_micro = 10000000` and `credit_usdt = "10.000000"`.
  - Feed 10 fake-but-correctly-shaped confirmed transfers (right contract + `to`, those 10 `value`s,
    **no `confirmed` field** — the real endpoint omits it) → each credits the **right** user exactly 10
    (no cross-contamination); replay the same `txid`s → **no** extra credit (idempotent).
  - A non-existent amount → `unmatched-amount` (doesn't leak to another user); wrong contract / wrong
    `to` / `confirmed:false` → skipped; **`confirmed` absent → still credits** (the Gotcha #1 regression).
  - 1000 same-base pending, then the 1001st `createInvoice` → **503**; a paid amount is reusable by a
    new invoice (returns to salt 0). *(Tıklatbari harness: 52/52.)*
- Boot on a temp `DB_PATH`; assert cancel → `expired` + no active invoice + no match; wrong
  contract/amount → skipped.
