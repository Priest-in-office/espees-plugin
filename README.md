# espees-payments

A Claude Code skill for building, reviewing and debugging **Espees (ESP) merchant payment integrations**, so a payment is confirmed correctly, granted exactly once, and never lost when Espees answers late or wrongly.

Espees' merchant API is small (create a product, send the payer to a pay link, ask whether it was paid), but it has **no webhook** and its documentation differs from what the API actually returns. This skill captures the design and the pitfalls from a production integration.

## What it does

When you work on Espees payments, Claude loads this skill and follows it to:

- **Build** a checkout: order table, a single `settle()` function that is the only place a payment is believed, the three places that must call it (return page, client, background reconciler), exactly-once granting, a support grant-by-hand, and a pause switch.
- **Review** an existing integration against the rules, and report each gap with the failure it causes.
- **Debug** "customer paid but got nothing", orders stuck pending or expired, `NOT FOUND` answers, and double charges.

Reference code is in **Node.js / Express / PostgreSQL**. The rules are written so they apply to any stack (Laravel, Django and others).

## Install

In Claude Code:

```
/plugin marketplace add Priest-in-office/espees-plugin
/plugin install espees-payments@espees-tools
```

The first command adds this repository as a marketplace; the second installs the skill from it. To update later, run `/plugin marketplace update espees-tools`.

## Contents

| File | What it holds |
|---|---|
| `skills/espees-payments/SKILL.md` | The flow, the design, 12 rules with their reasons, and how to handle each kind of request |
| `skills/espees-payments/references/api.md` | The API as documented, next to what it actually returns |
| `skills/espees-payments/references/node-reference.md` | A complete Node/Express/Postgres implementation and the tests worth writing |
| `skills/espees-payments/references/operations.md` | Testing with real money, a go-live checklist, a support runbook |

## Important: what "observed" means

The Espees behaviour described here was **observed in one production integration between September and November 2026** (last checked 6 November 2026). It is not taken from Espees' documentation and is **not guaranteed by Espees**. Examples:

- the confirm call answering `NOT FOUND` for a payment that had in fact been made;
- `DECLINED` in place of the documented `DECLINE`, and `price` returned as a string;
- the same pay link being charged twice without the server seeing the second charge.

Espees may change or fix any of this. Before relying on it in production, test with a small real payment from a second Espees account (Espees has no sandbox here, and a merchant wallet cannot pay itself).

The reference code was run against a real PostgreSQL database with a simulated Espees API. It has **not** been run against the live Espees API in this form.

This is an independent project. It is not affiliated with or endorsed by Espees, Loveworld or KingsChat.

## Corrections

Found behaviour that differs from what is written here, or that Espees has since changed? Please open an issue on this repository.

## Licence

MIT. See [LICENSE](LICENSE).
