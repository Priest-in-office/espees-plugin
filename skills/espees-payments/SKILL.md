---
name: espees-payments
description: Use this skill for anything involving accepting or verifying payments in Espees (ESP), including in Loveworld/KingsChat-connected apps, api.espees.org, payment.espees.org, merchant wallets, payment_ref or "pay with Espees". Covers writing checkout and confirm code in any stack (Laravel, Django, Node, mobile backends), working out how to learn a payment succeeded (Espees has no webhook), and debugging orders stuck pending or expired, "NOT FOUND" confirm responses, customers who paid but got nothing, and double charges or double grants. Also use it for questions about whether Espees can be used in iOS or Android app-store builds. Trigger even when the user only describes the symptom and says "espees" once, such as an order status column, a payment-confirmation controller or a ticket or course purchase.
---

# Espees merchant payments

Espees' API is small (create a product, send the payer to a link, ask if it was paid) but its docs differ from its behaviour, and the gaps have cost real money. This is the design that survived production, with the reason for each part.

Read `references/api.md` before writing HTTP calls (real response shapes). For Node/Express + Postgres, adapt `references/node-reference.md`. For testing, go-live and support, see `references/operations.md`.

## Flow

1. `POST https://api.espees.org/v2/payment/product` (header `x-api-key`): `product_sku`, `narration`, `price` (ESP), `merchant_wallet`, `success_url`, `fail_url`, `user_data` → `payment_ref`.
2. Send the payer to `https://payment.espees.org/pay/<payment_ref>` (they enter their PIN there).
3. Espees redirects the browser to `success_url` or `fail_url`.
4. `POST https://api.espees.org/v2/payment/confirm/` (trailing slash) with `{payment_ref}`: `transaction_status` is the only proof of payment.

**There is no webhook.** Your server must ask. Everything below follows from that.

## Design

One **order** row per checkout, created before calling Espees: `id`, `user_id`, what was bought, `price` (yours), `product_sku` (what you sent; the order id for new orders), `payment_ref` (unique), `status` (`pending` → `approved` | `declined` | `expired`), Espees' last answer (`espees_status`, `espees_details`, `checked_at`), `payer_username`, `overpaid`, timestamps. Plus an append-only **check log** of every *change* in Espees' answer for an order: in a dispute weeks later, "NOT FOUND at 21:58, DECLINED at 21:51, NOT FOUND at 22:03" is the evidence you and Espees need, and the latest answer alone overwrites it.

One function, **settle(order)**, is the only place a payment is believed:

1. Skip if `approved` (final) or no `payment_ref`.
2. Call confirm; on a network error change nothing.
3. Save the raw answer on the order; append to the log if it changed.
4. `APPROVED` → require `product_sku === order.product_sku` (a missing SKU is a mismatch) and `Number(price) >= order.price`. Then approve and grant in one transaction. Otherwise log loudly, grant nothing, leave it for a person. A price *above* the order's: grant, set `overpaid` so support can refund the difference.
5. `DECLINED`/`DECLINE` → mark a *pending* order `declined`.
6. Anything else (`PENDING`, `NOT FOUND`) → leave the status.

Call settle from three places, since none is reliable alone:
- **Return page** (`success_url` and `fail_url` share a handler): settle, then show the result. `?result=` is a hint anyone can type; ignore it.
- **Client**, when the payer returns (app foregrounds, tab refocuses): `GET /orders/:id` settles and returns the status; poll briefly while pending.
- **Reconciler** for payers who never return: every ~5 min settle pending orders from the last day; hourly settle declined and expired ones from the last week; expire pending orders over a day old. One process only. Start it from the app's entry point or a scheduler as part of the integration (one the developer must remember to start isn't running).

## Rules, and why

1. **Only confirm grants.** Redirects, `result` queries, client claims and echoed `user_data` can be forged or replayed; confirm, called by your server with your key, can't.
2. **Match SKU and price numerically.** Espees sends `price` as a string (`"15.0"`) and Postgres returns NUMERIC as one: `Number(a) + 1e-9 >= Number(b)`, never `===`.
3. **`NOT FOUND` is not proof nobody paid.** In production Espees took a payment, twice, while confirm kept answering `NOT FOUND`. Never treat it or `expired` as final; keep asking about declined and expired orders for a week and let a late `APPROVED` approve them; give support a "check again" on any unpaid order and a grant-by-hand (with a required note) for payers who show proof.
4. **Only `approved` is final.** A declined link (wrong PIN) can be paid on a retry, so approving works from `pending`, `declined` and `expired`.
5. **Grant exactly once.** Confirm says `APPROVED` every time, from three callers at once. In one transaction: `SELECT … FOR UPDATE` the order, return if already approved, set approved, insert the entitlement with a **unique** `order_id`. If it extends time, also lock the account row so payments stack rather than overlap.
6. **The same link can be charged twice unseen** (confirm shows one transaction per ref). Never hand out an existing pay link again. When Buy is pressed with an unpaid order from the last few minutes, settle that one first; if still unconfirmed, say "We haven't confirmed your earlier payment yet. If you already paid, wait a minute before paying again", and only then create a *new* order. Show "Payment received" the moment it's approved, and reconcile the merchant wallet's history against approved orders regularly. Refunds go through Espees (no refund API).
7. **Docs are wrong in places** (details in `references/api.md`): `DECLINED` not `DECLINE` (accept both), price is a string, `value_got` is false even when paid (log, never rely), and keep the confirm URL's trailing slash (don't "tidy" it).
8. **A wallet can't pay itself**: test from a second Espees account at 0.1 ESP.
9. **Outages happen.** Add an env switch that stops new checkouts with a clear message while started orders keep being checked; 20 s timeouts on every call.
10. **One SKU per checkout.** Use a backend-generated order id as `product_sku`, store it on the order, never reuse it; each retry is a new order. Delete the order if the product call fails.
11. **Key and wallet stay server-side.** The client gets an order id and a pay URL only.
12. **App stores:** Google Play and Apple require their own billing for in-app digital goods, so don't offer Espees in store builds; it fits web, sideloaded builds, physical goods and donations. Say so.

## Fixing an integration that already has orders

Old orders may carry another SKU than their id (a shared `CREDITS-20`, a random string). Backfill `product_sku` with what each was *actually* sent and compare against the column, not the id, or a late `APPROVED` for the very customers you're rescuing gets rejected. If the old SKU was shared, `payment_ref` is the identity and the SKU a sanity check: credit those by hand after checking the wallet history. Set old `failed` orders to `expired` so the late re-check asks again.

## By request

- **Add checkout:** order table, check log, settle, the three callers, the reconciler, support grant/re-check, pause switch, and tests with a fake Espees. The reconciler, late approval, idempotent grant and SKU/price check are the integration, not extras.
- **Stuck pending / paid but nothing:** ask confirm for that `payment_ref`, show the raw answer, compare with the wallet; if money arrived but confirm says `NOT FOUND`, grant by hand and report the ref to Espees (runbook in `references/operations.md`).
- **Charged twice:** one order with two charges on one link: refund via Espees, check rule 6.
- **Review:** check rules 1–6 first; report each gap with the failure it causes.
