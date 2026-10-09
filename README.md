# Agentic Commerce with UCP + Stripe Shared Payment Tokens

An AI shopping agent that finds products in chat, then checks out at a separate merchant service using Google's **Universal Commerce Protocol (UCP)** and pays with a Stripe **Shared Payment Token (SPT)**, so the merchant never handles the card.

I built this during Stripe's *Agentic Commerce* workshop. The workshop provided a starter kit with the chat UI, the hosted AI model, and stubbed services. My job was to implement the commerce and payment layer so the end-to-end purchase works. See [What I built](#what-i-built) for the parts I wrote and [Credits](#credits) for the parts I didn't.

## The problem

When an AI agent buys something for a user, three parties need to trust each other: the user, the agent, and the merchant. The merchant needs a standard way to expose checkout to any agent, and the user's card details must not be passed to the agent or the merchant. UCP standardizes the checkout API. SPTs give the merchant a single-use, amount-limited payment credential instead of a card.

## What I built

### 1. UCP merchant checkout API (`merchant-service/routes/checkout-sessions.js`)
I implemented the full checkout session lifecycle against the [UCP checkout REST spec](https://ucp.dev/2026-04-08/specification/checkout-rest/):

| Endpoint | What it does |
|---|---|
| `POST /checkout-sessions` | Validates items (exists, positive quantity, in stock) against the selected catalog, computes line items, creates the session |
| `GET /checkout-sessions/:id` | Returns the session with computed status and totals |
| `PUT /checkout-sessions/:id` | Merges buyer and fulfillment address, validates the shipping option, refuses edits to completed or canceled sessions |
| `POST /checkout-sessions/:id/complete` | Validates the `spt_` token, totals the order including shipping, and charges it with a Stripe PaymentIntent (`payment_method_data.shared_payment_granted_token`) |
| `POST /checkout-sessions/:id/cancel` | Cancels with an optional reason, with guards for sessions that are already completed or canceled |

All errors use the UCP shape `{ type, code, message }` with the right HTTP status (400, 404, 500). A declined payment returns **402** with the session, a `payment_error` block (`code`, `message`, `decline_code`), and an error added to the session's `messages`, so the agent can explain the failure to the user.

### 2. Agent-side UCP client (`agent-service/routes/checkout.js`)
I wrote `createCheckout`, `getCheckout`, `updateCheckout`, `completeCheckout`, and `cancelCheckout`. They call the merchant's UCP endpoints, convert the agent's camelCase fields to the protocol's snake_case, and turn failed responses into errors. Every call goes through the logged fetch wrapper, so it appears live in the UCP Inspector panel.

### 3. Shared Payment Token issuing (`agent-service/routes/payment.js`)
I implemented `getCustomerPaymentMethods` and `createSPT`. Before checkout completes, the agent turns the user's saved card into a scoped SPT (amount, currency, expiry) and sends only that token to the merchant.

### 4. Card collection with Stripe Elements (`frontend/components/PaymentSetup.tsx`)
I connected `loadStripe`, a SetupIntent per session customer, the `<Elements>` provider, and `stripe.confirmSetup` with `redirect: 'if_required'`. The page then saves the resulting payment method and shows its last 4 digits on the profile.

### 5. Product catalogs (`merchant-service/lib/`)
I filled in the ski catalog and wrote an 11-item **yacht** catalog (`yachts.json`) to test the agent with high-value products.

## How a purchase flows

```
User ──chat──► Frontend (Next.js :3000)
                  │
                  ▼
           Agent Service (:3001) ──► AI model (workshop Lambda): understands intent, picks products
                  │
                  ├─ POST/PUT /checkout-sessions ──► Merchant Service (:4000)   [UCP]
                  │
                  ├─ create SPT from saved card ──► Stripe (agent account)
                  │
                  └─ POST /checkout-sessions/:id/complete {spt_…} ──► Merchant
                                                                         │
                                         PaymentIntent(shared_payment_granted_token) ──► Stripe (merchant account)
```

1. The user saves a card once through Stripe Elements. It's attached to the agent's Stripe customer.
2. The agent creates a UCP checkout session at the merchant and updates it with the buyer, the address, and the shipping option.
3. When the session is ready, the agent issues an SPT limited to that amount.
4. The merchant charges the SPT. The order is confirmed and returned to the chat.

The merchant also publishes `GET /.well-known/ucp` (UCP discovery), which advertises Stripe as the payment handler and states that it accepts SPTs.

## Tech stack

- **Frontend:** Next.js 14, React 18, TypeScript, Tailwind, `@stripe/react-stripe-js`
- **Agent service:** Node.js and Express, Stripe SDK, an AI model hosted on AWS Lambda
- **Merchant service:** Node.js and Express, Stripe SDK (PaymentIntents with SPT)
- **Protocols:** UCP `2026-04-08` checkout REST, Stripe Shared Payment Tokens

## Setup

You need Node 18 or later, a Stripe test account, and the workshop's Lambda and proxy endpoints. The agent service's AI and SPT proxy are hosted by the workshop, so the chat flow won't run without those values.

```bash
cp agent-service/.env.example agent-service/.env        # LAMBDA_ENDPOINT, STRIPE_PROXY_URL, WORKSHOP_SECRET (+ ALLOWED_MERCHANT_URLS for extra merchants)
cp merchant-service/.env.example merchant-service/.env  # STRIPE_SECRET_KEY (sk_test_…)
./dev.sh            # installs dependencies and starts all three services (dev.ps1 on Windows)
./dev.sh --setup    # prompts for the .env values again
```

| Service | URL |
|---|---|
| Frontend | http://localhost:3000 |
| Agent service | http://localhost:3001 |
| Merchant service | http://localhost:4000 (`/.well-known/ucp`, `/checkout-sessions`, `/api/{catalog}`) |

To try the merchant API without the agent, use the Postman collection in `merchant-service/ACP_Checkouts.postman_collection.json`.

## Repository structure

```
frontend/            Next.js chat UI, UCP Inspector, Stripe Elements payment setup
agent-service/       Agent: chat orchestration, UCP client, SPT issuing
  ai-service/        AWS SAM Lambda for the AI model (provided)
  stripe-service/    AWS SAM Lambda for the Stripe proxy (provided)
merchant-service/    UCP merchant: checkout sessions, catalogs, webhooks, discovery
UCP-MIGRATION-LOG.md Upstream notes on the ACP to UCP migration
dev.sh / dev.ps1     Start all services
```

## Known limitations

- Checkout sessions are stored in memory and are lost when the merchant restarts.
- The agent only calls merchants on its allowlist (`MERCHANT_API_URL` plus `ALLOWED_MERCHANT_URLS`), so payment tokens can't be sent to an arbitrary URL.
- There are no automated tests yet. I verified the flow by hand through the chat UI and the UCP Inspector.
- `tsc --noEmit` reports a few type errors in the provided starter UI (`OrderConfirmation.tsx`, and `PaymentElement` options in `PaymentSetup.tsx`). The Next.js dev server runs anyway.

## Credits

The starter kit (chat UI, UCP Inspector, AI Lambda, Stripe proxy, catalog/admin routes, and dev scripts) comes from Stripe's Agentic Commerce workshop by Ben Smith: [benjasl-stripe/agentic-commerce-workshop-starter-kit](https://github.com/benjasl-stripe/agentic-commerce-workshop-starter-kit). The upstream commit history is kept in this repo, and my work is in the commits after it.

Protocol references: [UCP specification](https://ucp.dev/2026-04-08/specification/checkout-rest/) · [Google UCP docs](https://developers.google.com/merchant/ucp) · [Stripe Shared Payment Tokens](https://docs.stripe.com/payments/shared-payment-tokens)
