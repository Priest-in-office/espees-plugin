# Espees Merchant Web API: documented vs observed

Base `https://api.espees.org/v2/payment`. Every call: `Content-Type: application/json`, `x-api-key: <developer token>` (Token menu of the Espees developer console), and a ~20 s timeout (the API and pay page have both gone down). Observations are from production (Sep to Nov 2026); code for what's observed and accept the documented form too.

## 1. Create a product: `POST /product`

```json
{ "product_sku": "<order uuid>", "narration": "Premium, 30 days", "price": 1.5,
  "merchant_wallet": "0x…",
  "success_url": "https://api.example.com/billing/espees/return/<id>?result=success",
  "fail_url": "https://api.example.com/billing/espees/return/<id>?result=fail",
  "user_data": { "order_id": "<id>" } }
```

`product_sku`: backend-generated, one per checkout (your order id). `price`: float, ESP. Both URLs go to one handler that confirms either way. `user_data` is echoed by confirm; informational only. Response: `{ "statusCode": 200, "payment_ref": "61a28c…", "message": "Successfully Done" }`; no `payment_ref` = failure (delete the order).

## 2. Pay: `GET https://payment.espees.org/pay/<payment_ref>`

Open in the (system) browser. The payer logs in and enters their PIN; Espees then redirects to `success_url` or `fail_url`. The same link stays payable: a declined attempt can be retried on it, and one link was **charged twice**.

## 3. Confirm: `POST /confirm/` (keep the trailing slash), body `{ "payment_ref": "…" }`

Approved:
```json
{ "customer_username": "payer", "product_sku": "<id>", "description": "Premium, 30 days",
  "price": "1.5", "transaction_status": "APPROVED", "value_got": false,
  "status_details": "Successfully Done", "transaction_date": "2026-10-30 16:03:20", "user_data": {…} }
```
Declined: same shape with `"transaction_status": "DECLINED"`, `"status_details": "Wrong PIN"`. Paying from the merchant wallet: `"…[espees.client] An address cannot send coin to itself"`. Not found: `{ "transaction_status": "NOT FOUND" }` and nothing else: returned when no attempt was made **and, once, for a link that had been paid (twice)**. Never final.

| Docs | Observed | Do |
|---|---|---|
| `DECLINE` | `DECLINED` | accept both |
| `price` float | string (`"15.0"`) | compare with `Number()` |
| `narration` | `description` | depend on neither |
| `transaction_date` `18-01-2023 5:50:23` | `2026-10-30 16:03:20` | use your own `settled_at` |
| none | `value_got`, always `false`, even when paid | log, never rely |
| `NOT FOUND` = no attempt | also seen for a paid link | keep re-checking; grant by hand with proof |
| `PENDING` = processing | not seen yet | leave pending |

No webhook, no refund API, no per-merchant transaction list: the merchant wallet's own history in Espees is the only other record.
