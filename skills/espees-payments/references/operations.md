# Running an Espees integration: testing, going live, support

## Testing (real money; Espees has no sandbox here)

Price at 0.1 ESP. **Pay from a second Espees account**: the merchant wallet can't pay itself ("An address cannot send coin to itself"). Try each path once: a successful payment; a wrong PIN; wrong PIN then the right one on the same page; leaving without paying; paying and closing the browser before the redirect (the reconciler must catch it within ~5 min). After each, call confirm directly for the ref and compare its raw answer with the order.

## Go-live checklist

Swap in the live token and wallet and real prices, then check:

- [ ] settle is the only code that approves; the return page and the client both call it
- [ ] SKU checked against the stored `product_sku` (missing = mismatch); price compared as a number; overpayment granted and flagged
- [ ] confirm URL is `…/v2/payment/confirm/`, trailing slash included
- [ ] approve + grant is one transaction: row lock, unique order id on the grant; allowed from `pending`, `declined` and `expired`
- [ ] Espees' latest answer is on the order and every change is in the check log
- [ ] an earlier pay link is never handed out again; Buy pressed again settles the earlier order first
- [ ] the app itself starts the reconciler, in one process: pending every ~5 min for a day, declined/expired hourly for a week
- [ ] pause switch for new checkouts exists and is documented; 20 s timeouts on every Espees call
- [ ] support can re-check any unpaid order and grant by hand with a note
- [ ] key and wallet are server-side only
- [ ] the build isn't in Google Play or the App Store (or Espees isn't offered there)

## Support runbook

**"I paid but have nothing."** Re-check each of their orders (confirm on its `payment_ref`) and note the raw answer.
- `APPROVED` → settling grants it; done.
- `NOT FOUND`/`PENDING` → ask for the time, amount and their Espees username, and look in the merchant wallet's history.
- Money arrived but confirm still can't see it → grant by hand (a `grant` period, with a note naming the order and what Espees said) and send Espees the `payment_ref`s with the evidence. If Espees later confirms, the late re-check adds the paid period beside the grant; nothing is lost.
- No money arrived → it didn't go through; they can try again.

**"Charged twice."** One order with two charges on its link (confirm shows only one): Espees must refund the extra (no refund API). Grant what the first paid for. Two *orders* both approved: grant both or refund one, by policy.

**Espees is down** (pay page times out, confirm errors): turn on the pause switch, leave the reconciler running, turn it off when the pay page loads again.

## Logs

Log each *change* in Espees' answer: `Espees says NOT FOUND for order <id> (pending)`, `… DECLINED … Wrong PIN`, `… APPROVED …`, and any `confirm mismatch`. Without the `NOT FOUND` lines, a paid-but-unconfirmed order looks exactly like an abandoned checkout, which is how a double charge once went unnoticed for a day.
