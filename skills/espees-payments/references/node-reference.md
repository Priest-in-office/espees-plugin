# Reference implementation: Node, Express, PostgreSQL

Sells fixed periods of access ("30 days"). Adapt names and the grant; keep the shape. Deps: `express`, `pg`, `axios`. Server-only config: `ESPEES_API_KEY`, `ESPEES_MERCHANT_WALLET`, `PUBLIC_URL`, optional `ESPEES_CHECKOUT_PAUSED`.

## 1. Schema

```sql
CREATE TABLE espees_orders (
  id             UUID PRIMARY KEY,
  user_id        UUID NOT NULL REFERENCES users(id),
  plan           TEXT NOT NULL,
  days           INTEGER NOT NULL CHECK (days > 0),
  price_esp      NUMERIC(12,2) NOT NULL CHECK (price_esp > 0),
  product_sku    TEXT,                 -- what was sent to Espees (the id for new orders)
  payment_ref    TEXT UNIQUE,
  status         TEXT NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending','approved','declined','expired')),
  payer_username TEXT,
  overpaid       BOOLEAN NOT NULL DEFAULT false,
  espees_status  TEXT, espees_details TEXT, checked_at TIMESTAMPTZ,   -- last answer
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  settled_at     TIMESTAMPTZ
);
CREATE INDEX ON espees_orders (created_at) WHERE status = 'pending';
CREATE INDEX ON espees_orders (created_at) WHERE status IN ('declined','expired');
CREATE INDEX ON espees_orders (user_id, created_at DESC);

CREATE TABLE espees_checks (            -- every CHANGE in Espees' answer: dispute evidence
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id UUID NOT NULL REFERENCES espees_orders(id) ON DELETE CASCADE,
  transaction_status TEXT, status_details TEXT, raw JSONB NOT NULL,
  checked_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON espees_checks (order_id, checked_at);

CREATE TABLE access_periods (           -- order_id UNIQUE: a payment is granted once
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  source TEXT NOT NULL CHECK (source IN ('espees','grant')),   -- 'grant' = by hand, with note
  order_id UUID UNIQUE REFERENCES espees_orders(id),
  starts_at TIMESTAMPTZ NOT NULL, ends_at TIMESTAMPTZ NOT NULL CHECK (ends_at > starts_at),
  note TEXT
);
CREATE INDEX ON access_periods (user_id, ends_at);
```

Access = a row with `ends_at > now()`. Remember `ENABLE ROW LEVEL SECURITY` if your host needs it.

## 2. Espees client

```js
const axios = require('axios');
const API = 'https://api.espees.org/v2/payment';
const call = (path, body) => {
  const key = process.env.ESPEES_API_KEY;
  if (!key || !process.env.ESPEES_MERCHANT_WALLET) throw new Error('Espees is not configured');
  return axios.post(`${API}${path}`, body, { headers: { 'x-api-key': key }, timeout: 20000 }).then((r) => r.data);
};
exports.createProduct = async (p) => {   // → payment_ref
  const data = await call('/product', {
    product_sku: p.sku, narration: p.narration, price: p.price,
    merchant_wallet: process.env.ESPEES_MERCHANT_WALLET,
    success_url: p.successUrl, fail_url: p.failUrl, user_data: p.userData });
  if (!data?.payment_ref) throw new Error('Espees returned no payment_ref');
  return data.payment_ref;
};
exports.confirmPayment = (ref) => call('/confirm/', { payment_ref: ref });   // keep the slash
exports.paymentUrl = (ref) => `https://payment.espees.org/pay/${encodeURIComponent(ref)}`;
```

## 3. Repository (the only SQL)

```js
const pool = require('./db');            // a pg Pool
const crypto = require('crypto');
const UNPAID = ['pending', 'declined', 'expired'];   // only 'approved' is final

async function tx(fn) {
  const c = await pool.connect();
  try { await c.query('BEGIN'); const r = await fn(c); await c.query('COMMIT'); return r; }
  catch (e) { await c.query('ROLLBACK').catch(() => {}); throw e; }
  finally { c.release(); }
}
const one = async (sql, args) => (await pool.query(sql, args)).rows[0] ?? null;

exports.createOrder = ({ userId, plan, days, priceEsp }) => {
  const id = crypto.randomUUID();      // the order id is also what we send Espees as product_sku
  return one(
    `INSERT INTO espees_orders (id, user_id, plan, days, price_esp, product_sku)
     VALUES ($1,$2,$3,$4,$5,$6) RETURNING *`, [id, userId, plan, days, priceEsp, id]);
};
exports.findOrder = (id) => one('SELECT * FROM espees_orders WHERE id = $1', [id]);
exports.setPaymentRef = (id, ref) => pool.query('UPDATE espees_orders SET payment_ref = $2 WHERE id = $1', [id, ref]);
exports.deleteOrder = (id) => pool.query("DELETE FROM espees_orders WHERE id = $1 AND status = 'pending'", [id]);
exports.declineOrder = async (id) => (await one(
  `UPDATE espees_orders SET status='declined', settled_at=now() WHERE id=$1 AND status='pending' RETURNING *`, [id]))
  ?? exports.findOrder(id);

exports.recordCheck = async (id, { status, details, raw, changed }) => {
  await pool.query('UPDATE espees_orders SET espees_status=$2, espees_details=$3, checked_at=now() WHERE id=$1', [id, status, details]);
  if (changed) await pool.query(
    'INSERT INTO espees_checks (order_id, transaction_status, status_details, raw) VALUES ($1,$2,$3,$4)',
    [id, status, details, JSON.stringify(raw ?? null)]);
};

// The account's latest still-pending order from the last few minutes (a declined one is a clear
// "not paid": no warning, they just try again).
exports.findRecentUnpaid = (userId, minutes = 15) => one(
  `SELECT * FROM espees_orders WHERE user_id=$1 AND status = 'pending' AND payment_ref IS NOT NULL
     AND created_at > now() - make_interval(mins => $2) ORDER BY created_at DESC LIMIT 1`, [userId, minutes]);

// Approve + grant, once. Period starts when the account's current paid access ends, so payments stack.
exports.approveOrder = (id, payer, { overpaid = false } = {}) => tx(async (c) => {
  const order = (await c.query('SELECT * FROM espees_orders WHERE id=$1 FOR UPDATE', [id])).rows[0];
  if (!order || !UNPAID.includes(order.status)) return order ?? null;
  const done = (await c.query(
    `UPDATE espees_orders SET status='approved', payer_username=$2, overpaid=$3, settled_at=now()
     WHERE id=$1 RETURNING *`, [id, payer ?? null, overpaid])).rows[0];
  await c.query('SELECT 1 FROM users WHERE id=$1 FOR UPDATE', [order.user_id]);   // serialise per account
  await c.query(
    `INSERT INTO access_periods (user_id, source, order_id, starts_at, ends_at)
     SELECT $1,'espees',$2,s.at, s.at + make_interval(days => $3)
       FROM (SELECT greatest(now(), (SELECT max(ends_at) FROM access_periods
                                      WHERE user_id=$1 AND source<>'grant')) AS at) s`,
    [order.user_id, id, order.days]);
  return done;
});

exports.listPending = async () => (await pool.query(
  `SELECT id FROM espees_orders WHERE status='pending' AND payment_ref IS NOT NULL
     AND created_at > now() - interval '1 day' ORDER BY created_at`)).rows;
exports.listLate = async () => (await pool.query(
  `SELECT id FROM espees_orders WHERE status IN ('declined','expired') AND payment_ref IS NOT NULL
     AND created_at > now() - interval '7 days' ORDER BY created_at`)).rows;
exports.expireStale = () => pool.query(
  `UPDATE espees_orders SET status='expired', settled_at=now()
   WHERE status='pending' AND created_at <= now() - interval '1 day'`);
```

## 4. Service

```js
const repo = require('./orders.repository');
const espees = require('./espees');
const PLANS = { monthly: { days: 30, priceEsp: 1.5 }, yearly: { days: 365, priceEsp: 15 } };
const UNPAID = new Set(['pending', 'declined', 'expired']);
class PaymentError extends Error { constructor(m, status = 400) { super(m); this.status = status; } }
const paused = () => ['1', 'true', 'yes'].includes(String(process.env.ESPEES_CHECKOUT_PAUSED ?? '').toLowerCase());

// confirmAgain: the payer saw the "earlier payment not confirmed" warning and chose to go on.
async function startCheckout(user, planId, { confirmAgain = false } = {}) {
  const plan = PLANS[planId];
  if (!plan) throw new PaymentError('Choose a plan.');
  if (paused()) throw new PaymentError('Paying with Espees is paused for now. Please try again later.', 503);

  const earlier = await repo.findRecentUnpaid(user.id);          // a link Espees calls unpaid may be paid
  if (earlier) {
    const s = await settle(earlier.id);
    if (s?.status === 'approved') return { orderId: s.id, alreadyPaid: true };
    if (s?.status === 'pending' && !confirmAgain) throw new PaymentError(
      "We haven't confirmed your earlier payment yet. If you already paid, wait a minute before paying again.", 409);
  }
  const order = await repo.createOrder({ userId: user.id, plan: planId, days: plan.days, priceEsp: plan.priceEsp });
  const back = `${process.env.PUBLIC_URL}/billing/espees/return/${order.id}`;
  let ref;
  try {
    ref = await espees.createProduct({ sku: order.product_sku, narration: `Premium, ${plan.days} days`,
      price: plan.priceEsp, successUrl: `${back}?result=success`, failUrl: `${back}?result=fail`,
      userData: { order_id: order.id } });
  } catch (err) {
    await repo.deleteOrder(order.id).catch(() => {});
    console.error('Espees product error:', err.response?.data ?? err.message);
    throw new PaymentError("Couldn't start the Espees payment. Please try again.", 502);
  }
  await repo.setPaymentRef(order.id, ref);                       // never hand out an old link again
  return { orderId: order.id, payUrl: espees.paymentUrl(ref) };
}

// The only place a payment is believed. Safe to call any number of times, from anywhere, at once.
async function settle(orderId) {
  const order = await repo.findOrder(orderId);
  if (!order || !UNPAID.has(order.status) || !order.payment_ref) return order;
  let answer;
  try { answer = await espees.confirmPayment(order.payment_ref); }
  catch (err) { console.error('Espees confirm error', order.id, err.response?.data ?? err.message); return order; }

  const status = String(answer?.transaction_status ?? '').toUpperCase();
  const details = answer?.status_details ? String(answer.status_details).slice(0, 500) : null;
  const changed = status !== (order.espees_status ?? '') || details !== order.espees_details;
  if (changed) console.log('Espees says', status, 'for order', order.id, `(${order.status})`, details ?? '');
  await repo.recordCheck(order.id, { status: status || null, details, raw: answer, changed });

  if (status === 'APPROVED') {
    const paid = Number(answer.price), due = Number(order.price_esp);   // both arrive as strings
    if (!order.product_sku || answer.product_sku !== order.product_sku || !(paid + 1e-9 >= due)) {
      console.error('Espees confirm mismatch', order.id, { sku: answer.product_sku, price: answer.price });
      return repo.findOrder(order.id);                                  // a missing SKU is a mismatch
    }
    return repo.approveOrder(order.id, answer.customer_username, { overpaid: paid > due + 1e-9 });
  }
  if (status === 'DECLINED' || status === 'DECLINE') return repo.declineOrder(order.id);
  return repo.findOrder(order.id);                                      // PENDING, NOT FOUND: not paid *yet*
}

async function reconcile({ late = true } = {}) {
  await repo.expireStale();
  for (const { id } of await repo.listPending()) await settle(id);
  if (late) for (const { id } of await repo.listLate()) await settle(id);
}
function startReconciler() {        // one process only (pm2 cluster: NODE_APP_INSTANCE '0')
  if (process.env.NODE_APP_INSTANCE && process.env.NODE_APP_INSTANCE !== '0') return () => {};
  let running = false, lateAt = 0;
  const timer = setInterval(async () => {
    if (running) return; running = true;
    try { const late = Date.now() - lateAt >= 3600e3; await reconcile({ late }); if (late) lateAt = Date.now(); }
    catch (e) { console.error('Espees reconcile error:', e.message); } finally { running = false; }
  }, 5 * 60e3);
  timer.unref();
  return () => clearInterval(timer);
}
module.exports = { PaymentError, reconcile, settle, startCheckout, startReconciler };
```

## 5. Routes and wiring

```js
api.post('/espees/checkout', requireAuth, async (req, res) => {       // mount api at /api/billing
  try { res.json(await service.startCheckout(req.user, req.body?.plan, { confirmAgain: req.body?.confirmAgain === true })); }
  catch (e) { res.status(e.status ?? 500).json({ error: e.status ? e.message : 'Something went wrong.' }); }
});
api.get('/espees/orders/:id', requireAuth, async (req, res) => {       // the client calls this on return
  const o = UUID_RE.test(req.params.id) && await repo.findOrder(req.params.id);
  if (!o || o.user_id !== req.user.id) return res.status(404).json({ error: 'Payment not found.' });
  const s = await service.settle(o.id);
  res.json({ orderId: s.id, status: s.status });
});
pages.get('/espees/return/:id', async (req, res) => {                 // mount pages at /billing; ?result= ignored
  const o = UUID_RE.test(req.params.id) ? await service.settle(req.params.id).catch(() => null) : null;
  // render by o?.status: approved "Payment received"; pending "Waiting for Espees. Don't pay this link
  // again." (auto-refresh); declined "Didn't go through"; expired "Couldn't confirm. If you paid, contact support."
});
// server.js: app.use('/api/billing', api); app.use('/billing', pages);
//            const stop = startReconciler(); process.on('SIGTERM', () => { stop(); server.close(); });
```

Client: open `payUrl` in the system browser, keep `orderId`; when the app/tab is active again call `GET /orders/:id` (poll about a minute while pending). A 409 from checkout shows its message plus "I haven't paid, pay again" (resend with `confirmAgain: true`); `alreadyPaid` means refresh access, no link.

Support (admin only): re-check any unpaid order (`settle`); grant by hand = insert an `access_periods` row with `source='grant'`, a required note and the order id if any.

## 6. Tests (fake Espees)

Replace `axios.post` before loading the app. The fake answers `/product` with `ref-N` and `/confirm/` with a per-ref answer, defaulting to `{transaction_status:'NOT FOUND'}`; "paid" is `{customer_username, product_sku, price: String(price), transaction_status:'APPROVED', value_got:false}`. Each of these was a real failure:

- pending until `APPROVED`, then access for the plan's days; settling 3 times grants **one** period;
- wrong SKU, missing SKU, or lower price grants nothing; higher price grants and sets `overpaid`;
- `DECLINED` then `APPROVED` on the same ref approves; `NOT FOUND` for a day expires, a later `APPROVED` is approved by the hourly late pass; past 7 days nobody asks;
- `?result=success` alone grants nothing; the confirm URL ends in `/confirm/` (assert it);
- Espees down at checkout leaves no order and gives 502; paused gives 503 while started orders still settle;
- Buy again within minutes: earlier order approved → `alreadyPaid`; still pending → 409 unless `confirmAgain`; declined (wrong PIN) → no warning, a new order; an old pay link is never returned;
- each change in Espees' answer adds one `espees_checks` row, the same answer twice adds none;
- the reconciler starts in process 0 only.

## 7. Other stacks

Keep: one settle; approve+grant in one transaction with a row lock (`select_for_update()`, `lockForUpdate()`) and a unique `order_id`; the reconciler as a scheduled job that runs once per interval, not per web worker (cron, Celery beat, Laravel scheduler); Espees numbers parsed before comparing (`float(...)`, `(float)`).
