---
name: revolut-merchant-payments
description: "Implement payment integrations using the Revolut Merchant API. Use this skill when building, integrating, or debugging any type of payment with Revolut: card payments, Apple Pay, Google Pay, Revolut Pay, Pay by Bank, hosted checkout pages, payment links, in-person terminal payments, recurring subscriptions, or saved payment methods. Covers full order lifecycle (create, capture, refund), 3DS authentication, webhooks, and customer management."
---

# Revolut Merchant Payments

Implement all Revolut Merchant API payment types. The API is REST-based, JSON-encoded, and requires API key authentication.

## Authentication

Two keys — keep the secret key server-side only; public key is safe for client-side:

```
Authorization: Bearer <SECRET_KEY>    # all server-to-server API calls
publicKey: "<PUBLIC_KEY>"             # Checkout Widget / SDK initialization
```

Base URLs:
- **Sandbox**: `https://sandbox-merchant.revolut.com/api/1.0`
- **Production**: `https://merchant.revolut.com/api/1.0`

## Payment Methods Overview

| Payment Method | Integration Path | In Sandbox |
|---|---|---|
| Card | Widget / SDK / Self-hosted | Yes |
| Apple Pay | Widget / SDK | Yes |
| Google Pay | Widget / SDK | Yes |
| Revolut Pay | Widget / SDK | Yes |
| Pay by Bank | Widget / SDK | No (prod only) |
| Hosted Checkout / Payment Link | API or Dashboard | Yes |
| In-Person Terminal (Push) | REST API | Yes |
| Recurring / Saved Methods | API (merchant-initiated) | Yes |

## Core Order Lifecycle

All payments are anchored to an **order**. Create server-side, then pay client-side or server-side.

### 1. Create Order

```http
POST /orders
{
  "amount": 1000,
  "currency": "EUR",
  "capture_mode": "automatic",
  "customer_email": "user@example.com",
  "description": "Order #123"
}
```

Response includes two identifiers — keep them straight:
- `id` — use server-side for capture, refund, cancel, and API calls
- `public_id` — pass to the client-side SDK/Widget to attach payment to this order
For hosted checkout, redirect the customer to `checkout_url` from the response.

### 2. Payment States

```
pending
  → authentication_challenge → authentication_verified
  → authorisation_started → authorisation_passed → authorised
  → capture_started → captured → completing → completed

Failure: declining → declined | soft_declined
         failing  → failed
         cancel:  cancellation_started → cancelling → cancelled
```

### 3. Capture (manual mode only)

```http
POST /orders/{order_id}/capture
{ "amount": 1000 }
```

Partial capture supported — remainder is voided. Authorised orders auto-void after **7 days**.

### 4. Refund

```http
POST /orders/{order_id}/refund
{ "amount": 500 }
```

Full or partial. **Pay by Bank does not support API refunds** — handle outside Revolut.

### 5. Cancel

```http
POST /orders/{order_id}/cancel
```

## Integration Paths

Choose based on how much control you need over the UI:

| Path | Who hosts UI | When to use |
|---|---|---|
| **A. Web SDK** | You (embedded widget) | Custom branded checkout |
| **B. Hosted Checkout** | Revolut | Fastest integration, payment links |
| **C. Self-Hosted** | You (raw API) | Full control, custom 3DS handling |
| **D. Terminal Push** | In-person device | POS software, physical payments |

### A. Revolut Merchant Web SDK

Unified TypeScript SDK for Card, Revolut Pay, Apple Pay, Google Pay, Pay by Bank.

```html
<script src="https://merchant.revolut.com/embed.js"></script>
```

```js
const revolut = await RevolutCheckout(publicKey);
const payments = revolut.paymentRequest({
  target: document.getElementById('payment-container'),
  request: {
    label: 'My Store',
    total: { amount: 1000, currency: 'EUR' },
  },
});
```

For per-method widget config, Apple Pay domain registration, and SDK options → `references/sdk.md`

### B. Hosted Checkout Page

Revolut hosts the checkout. Redirect customer to `checkout_url` from the create order response, or generate a **payment link** from the Business dashboard (no code needed).

### C. Self-Hosted Checkout (3DS)

Build your own UI, collect card data via Widget, handle 3DS challenges manually.

When the payment response has state `authentication_challenge`, redirect to `authentication_challenge.acs_url` for 3DS completion.

→ Full flow in `references/self-hosted.md`

### D. In-Person Terminal (Push Payments)

Server-to-server REST, no SDK or physical connection needed:

```http
POST /terminal-payments
{
  "order_id": "<order_id>",
  "device_id": "<terminal_device_id>"
}
```

→ `references/in-person.md`

## Saved Payment Methods & Recurring

Save during checkout, charge server-side later (subscriptions, 1-click checkout).

```js
// SDK: set savePaymentMethodFor on card widget
revolut.card({
  target: el,
  savePaymentMethodFor: 'merchant',   // or 'customer'
});
```

- `customer` — customer-initiated future payments (1-click checkout)
- `merchant` — merchant-initiated (subscriptions, no customer present)

Charge saved method — always create a fresh order first, then pay it with the saved method:
```http
POST /orders
{ "amount": 1500, "currency": "EUR", "customer_id": "<customer_id>" }

POST /orders/{new_order_id}/payments
{
  "payment_method_id": "<saved_method_id>",
  "customer_id": "<customer_id>"
}
```

→ `references/recurring.md` for full subscription lifecycle

## Customer Management

```http
POST   /customers
GET    /customers/{id}
GET    /customers/{id}/payment-methods
DELETE /customers/{id}/payment-methods/{method_id}
```

## Webhooks

Register up to **10 webhook URLs**.

```http
POST /webhooks
{
  "url": "https://yoursite.com/webhook",
  "events": ["ORDER_COMPLETED", "ORDER_AUTHORISED"]
}
```

Key events: `ORDER_COMPLETED`, `ORDER_AUTHORISED`, `ORDER_PAYMENT_DECLINED`, `SUBSCRIPTION_INITIATED`, `SUBSCRIPTION_FINISHED`, `SUBSCRIPTION_CANCELLED`, `SUBSCRIPTION_OVERDUE`

→ `references/webhooks.md` for verification and full event catalog

## Reference Files

| File | When to read |
|---|---|
| `references/sdk.md` | Implementing any Widget or Web SDK payment method |
| `references/self-hosted.md` | Building a custom checkout with manual 3DS handling |
| `references/recurring.md` | Subscriptions and merchant-initiated charges |
| `references/in-person.md` | Terminal / push payment integration |
| `references/webhooks.md` | Setting up or debugging webhooks |
