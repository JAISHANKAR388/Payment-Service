# Payment Service

A reusable payment service built with a service-oriented design: **one core engine, many applications**.
Each consuming app (an e-commerce shop, a bank, ...) plugs in its own **profile** (payment methods, rules, fee policy, refund policy and event hooks) without touching the core.

Zero dependencies. Requires Node.js 18+.

## Why

Payments look similar across products, but the policies differ. A shop takes cards, charges the merchant a fee and allows refunds. A bank moves money between accounts, enforces limits and OTP checks, charges the payer and offers no refunds. This project shows how to keep one tested engine and express those differences as configuration.

## Architecture

```
            +-------------------------- PaymentService (core) --------------------------+
 App  ----> | validate -> rules -> fee -> gateway -> events      idempotency | refunds   |
 (API key)  +------------------------------------------------------------------------+
                  ^            ^             ^                 ^
              apps.js       rules.js     gateways/*          hooks
            (app profile)  (policies)   (how money moves)  (per-app events)
```

```
src/core/PaymentService.js   payment lifecycle, idempotency, fees, refunds (app-agnostic)
src/gateways/card.js         simulated card processor (swap for Stripe/Razorpay)
src/gateways/ledger.js       account ledger, used as "wallet" by the shop and "transfer" by the bank
src/rules.js                 reusable rules: currencies, maxAmount, dailyLimit, requireOtp
src/apps.js                  per-app profiles: shopmart, securebank
src/index.js                 registry: forApp(appId), appIdFromKey(key)
src/server.js                REST API (x-api-key identifies the calling app)
src/demo.js                  runnable walkthrough of both apps
```

## Applications included

| | shopmart (e-commerce) | securebank (banking) |
|---|---|---|
| Methods | card, wallet | transfer |
| Fee | 2.9% + 30c on cards, paid by merchant | Rs 5 over Rs 10,000, paid by payer |
| Rules | USD/INR, max $5,000 per payment | INR only, Rs 50,000/day, OTP above Rs 25,000 |
| Refunds | yes (partial supported) | no |
| Hooks | order confirmation | audit log |

All amounts are integers in minor units (cents / paise).

## Getting started

```bash
git clone <https://github.com/JAISHANKAR388/Payment-Service>
cd payment-service
npm run demo     # walkthrough of both apps
npm start        # REST API on http://localhost:3000
```

Set `PORT` to change the port. API keys and secrets can be overridden with `SHOP_KEY` and `BANK_KEY`.

## API

| Method | Path | Notes |
|---|---|---|
| POST | `/v1/payments` | headers: `x-api-key`, optional `idempotency-key` |
| GET | `/v1/payments/:id` | only returns payments of the calling app |
| POST | `/v1/payments/:id/refund` | body: `{ "amount": 1500 }` (omit for full refund) |

Example:

```bash
curl -X POST http://localhost:3000/v1/payments \
  -H "x-api-key: shop-demo-key" -H "content-type: application/json" \
  -d '{"amount":4999,"currency":"USD","method":"card","payer":"cust_42","payee":"SHOPMART","source":{"token":"tok_ok"}}'
```

Demo keys: `shop-demo-key`, `bank-demo-key`. Demo OTP for the bank: `482913`. Card token `tok_declined` simulates a declined card.

Errors return `{ "error": { "code": "...", "message": "..." } }` with codes such as `CURRENCY_NOT_ALLOWED`, `OTP_REQUIRED`, `DAILY_LIMIT_EXCEEDED`, `REFUNDS_DISABLED`, `NOT_FOUND`.

## Add a new application

1. Add an entry in `src/apps.js` choosing gateways, rules, fee function, fee bearer, refund policy and hooks.
2. Need a new payment method? Add `src/gateways/yourGateway.js` exporting `() => ({ name, charge(ctx), refund(ctx) })` and use it in the profile.
3. Need a new policy? Add a rule factory to `src/rules.js` returning `(req, ctx) => void` that throws `PaymentError` to reject.

The core engine does not change.

## Production notes

This is a reference implementation. Before real use: replace the in-memory store with a database, replace simulated gateways with real provider SDKs, move API keys and secrets to a secrets manager, verify OTPs through a real service, and add persistent audit logging, rate limiting and HTTPS.

## License

MIT
