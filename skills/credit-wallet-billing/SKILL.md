---
name: credit-wallet-billing
description: In-app credit balance (a wallet) as the billing currency — users top up credits (real crypto on-chain, or a demo instant path) then spend them to buy subscription plans, with server-enforced balance checks and an atomic plan+credit update on SQLite. Also covers the honest real-vs-demo dual-page naming ("Kredi Yükle" vs "DEMOÖDEME") and the demo-endpoint production gate. Use when building a credit/wallet economy, plan purchases paid from an in-app balance, or "top up then upgrade" billing in a Node/Express + React app. Pairs with the usdt-trc20-payments skill (the real on-chain top-up) and plan-gating in config/plans.js. Reusable pattern from Tıklatbari.
---

# Credit Wallet + Plan Purchase (SQLite, server-enforced)

A **credit balance is the billing currency**. Two things earn credits (one real, one demo); one
server-enforced action spends them to buy a subscription plan. Decoupling "get money in" from "buy a
plan" means the plan-purchase path never touches a payment provider — it only moves an integer balance,
so it's simple, synchronous, and race-free. Node/Express + `better-sqlite3` + React.

```
        ┌─ REAL: on-chain USDT top-up ─┐
top up ─┤  (usdt-trc20-payments skill)  ├─→  users.credits ──spend──→ POST /plan/credits ──→ users.plan
        └─ DEMO: instant confirm ──────┘        (balance)     (server-enforced, atomic)      (upgraded)
```

## The currency
- **`users.credits REAL NOT NULL DEFAULT 0`** — the wallet balance. **1 credit = 1 USD** (1:1), a fixed
  convention so prices and credits are directly comparable.
- **Prices live in ONE place** — `config/plans.js` (`getPlan(id).price`), the same source that drives
  plan-gating and the pricing table. Never hardcode a price in a route. `normalizePlan()` maps legacy
  ids (e.g. `premium → pro`) so old rows keep working.

## Earning credits — one real path, one demo path
- **REAL (production):** the on-chain top-up (see **usdt-trc20-payments**). A poller verifies an
  incoming USDT transfer and does `UPDATE users SET credits = credits + <service_amount>`; nothing else
  touches the plan. This is the honest "Kredi Yükle" page.
- **DEMO (prototype only):** `POST /billing/credit/confirm { amount }` → `credits += amount` on a
  client-entered number, **no payment**. Bounded (`10 ≤ amount ≤ 100000`, 2-dp) but trusts the client.
  Ship-blocker (see Gotchas).

## Spending credits — the ONE enforced path
`POST /billing/plan/credits { plan }` — buy a plan with the balance. Everything is server-side; the
client is never trusted for price or balance:
```js
const target = normalizePlan(req.body?.plan);
if (target === 'free') return res.status(400).json({ error: 'Ücretli bir plan seçiniz.' });
const price = getPlan(target).price;                          // price from the single source
const row   = db.prepare('SELECT credits, plan FROM users WHERE id = ?').get(userId);
const have  = Number(row?.credits || 0);
if (have < price)                                             // ← balance check, server-side
  return res.status(402).json({ error: 'Yetersiz kredi.', need: price, have });
db.transaction(() => {                                        // ← atomic: debit + upgrade together
  db.prepare('UPDATE users SET credits = credits - ?, plan = ? WHERE id = ?').run(price, target, userId);
})();
logActivity({ userId, action: 'plan.purchase_credits', detail: { plan: target, price, prevPlan: row.plan } });
```
**Why it's race-free:** `better-sqlite3` is synchronous and there is **no `await`** between the
`SELECT` (read balance) and the `UPDATE` (debit) — in a single-process Node app the read→check→write
runs atomically, so no TOCTOU, no double-spend. (Multi-process breaks this — see Gotchas.)

**Demo direct upgrade:** `POST /billing/crypto/confirm { plan }` just does `UPDATE users SET plan = ?`
with **no payment and no balance check** — demo only. **Downgrade:** `POST /billing/downgrade` →
`plan='free'` (no charge). Log every mutation (`plan.purchase_credits` / `plan.purchase_direct` /
`plan.downgrade` / `credit.demo_load` / `payment.credited`) to an audit table — invisible to users.

## Frontend
- **Balance source of truth:** `user.credits`, hydrated from `GET /auth/me` (which selects `credits`).
  After any credit/plan change call `refreshUser().catch(()=>{})` so the balance re-renders everywhere
  (settings, pricing modal, payment pages).
- **Purchase choice modal** (on the pricing page): show the balance, then two buttons —
  **"Kredilerimle Öde"** (`POST /plan/credits`; **disabled when `credits < price`**, labeled
  "— yetersiz kredi") and **"Direkt Ödeme (Kripto)"** (navigates to the real top-up page so the user
  loads credits, then buys). On success, `refreshUser()` + reload the plan.

## Honest dual-page naming (real vs demo)
When both a real and a demo money flow exist, **name them so nobody is misled** — users *or* future
devs:
- **"Kredi Yükle"** (`/kredi-yukle`) = the REAL on-chain top-up. Loads `users.credits`, never touches
  the plan.
- **"DEMOÖDEME"** (`/demo-odeme`) = the DEMO page; its UI literally says *"Demo modu — gerçek zincir
  kontrolü yapılmaz"* and it flips plan/credits instantly on a click.
Keep the two as separate sidebar entries with the real one first. Renaming a page = update **all** of:
route path (App.jsx), sidebar label, header-title map, nginx reserved-route regex, and every inbound
`navigate()/<Link>` — grep the old slug to zero afterward.

## Gotchas
1. **Demo endpoints are a free-upgrade backdoor.** `POST /billing/crypto/confirm {plan:'pro'}` and
   `credit/confirm {amount}` are behind auth but do **no** payment/balance check — any logged-in user
   can self-grant Pro or unlimited credits. **Gate or delete them before production** (`NODE_ENV`
   check, or remove). Their existence next to the real flow is the single biggest liability.
2. **Balance safety is single-process only.** The lock-free read→check→debit is atomic *only* because
   `better-sqlite3` is synchronous. Under 2+ processes, two concurrent purchases can both pass the
   check and drive `credits` negative — the `UPDATE` has no `WHERE credits >= price` guard. Before
   scaling out, add that guard (`… WHERE id = ? AND credits >= ?`) and check `changes === 1`.
3. **No "target vs current" check.** `/plan/credits` will happily let a Pro user re-buy Pro (charged
   again) or "buy" a lower tier (paid downgrade). Add `if (target === current) 400` and decide
   whether credit-buying a lower plan is allowed.
4. **Price only from `config/plans.js`.** If a route hardcodes a price it drifts from the pricing table
   and the gating caps. One source, always.
5. **Credit = intended value.** With the salt top-up, credit the round `service` amount, not the
   received on-chain value (see usdt-trc20-payments Gotcha #4).

## Verification
- On a temp `DB_PATH`, seed a user with `credits = 20`, plan `free`. `POST /plan/credits {pro}`
  (price 29.99) → **402** `{need:29.99, have:20}`, plan unchanged. Top up to 30, retry → plan `pro`,
  `credits = 0.01`, one `plan.purchase_credits` log. Repeat with 0 credits → 402 again.
- Assert the demo `credit/confirm` adds credits with no payment (proving it must be gated), and that
  `crypto/confirm` upgrades plan with no balance check.
- Assert `GET /auth/me` returns the updated `credits` so the frontend refresh shows the new balance.
- Assert prices come from `getPlan()` (change a price in `config/plans.js` → the enforced charge moves).
