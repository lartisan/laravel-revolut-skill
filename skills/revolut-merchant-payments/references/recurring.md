# Recurring Payments & Subscriptions

## Overview

Two use cases for saved payment methods:

| Use Case | Save target | Who initiates payment |
|---|---|---|
| 1-click checkout | `customer` | Customer (on checkout page) |
| Subscriptions / recurring | `merchant` | Merchant (no customer present) |

## Setup: Save a Payment Method

### During Widget Checkout

```js
revolut.card({
  target: el,
  publicToken: orderPublicId,
  savePaymentMethodFor: 'merchant',   // saves for merchant-initiated use
  onSuccess() {
    // method is now saved — retrieve method_id from order
  },
});
```

### During Self-Hosted Checkout

Add to the payment request body:

```http
POST /orders/{order_id}/payments
{
  "payment_method": { ... },
  "save_payment_method_for": "merchant"
}
```

## Retrieve Saved Payment Methods

```http
GET /customers/{customer_id}/payment-methods?usage_type=MERCHANT
```

Response includes array of payment methods with `id`, `type`, `last4`, `expiry_month`, `expiry_year`. Use `usage_type=MERCHANT` to filter for methods that can be charged server-side without the customer present. Use `usage_type=CUSTOMER` for 1-click checkout methods.

## Charge a Saved Method (No Customer Present)

1. Create a new order for the charge amount.
2. Pay the order using the saved method ID:

```http
POST /orders
{
  "amount": 1500,
  "currency": "EUR",
  "description": "Monthly subscription - April 2026",
  "customer_id": "<customer_id>"
}
```

```http
POST /orders/{order_id}/payments
{
  "payment_method_id": "<saved_method_id>",
  "customer_id": "<customer_id>"
}
```

No 3DS required for merchant-initiated transactions (MITs) — liability shifts to merchant.

## Subscriptions API

The Subscriptions API manages recurring billing plans automatically.

### Create a Subscription Plan

```http
POST /subscriptions
{
  "customer_id": "<customer_id>",
  "payment_method_id": "<saved_method_id>",
  "plan": {
    "amount": 999,
    "currency": "EUR",
    "interval": "month",   // "day" | "week" | "month" | "year"
    "interval_count": 1,
    "trial_period_days": 14
  },
  "description": "Pro Plan Monthly"
}
```

### Subscription States

```
active → overdue → cancelled | finished
```

- `overdue`: automatic retry attempted; webhook `SUBSCRIPTION_OVERDUE` fired
- `cancelled`: stopped by merchant or after failed retries
- `finished`: natural end of billing period

### Manage Subscriptions

```http
GET    /subscriptions/{id}
POST   /subscriptions/{id}/cancel
GET    /subscriptions                  # list all
```

### Subscription Webhooks

| Event | When |
|---|---|
| `SUBSCRIPTION_INITIATED` | First charge processed |
| `SUBSCRIPTION_FINISHED` | Plan period ended |
| `SUBSCRIPTION_CANCELLED` | Subscription cancelled |
| `SUBSCRIPTION_OVERDUE` | Payment failed, retry pending |

## Customer Management

Always create a customer before saving payment methods. The `id` in the response is the `customer_id` you'll use in all subsequent calls (orders, payment methods, subscriptions):

```http
POST /customers
{
  "full_name": "Ion Popescu",
  "email": "ion@example.com",
  "phone": "+40712345678"
}
# Response: { "id": "cust_abc123", ... }
# → use "cust_abc123" as customer_id everywhere below
```

Delete a saved method when customer cancels:

```http
DELETE /customers/{customer_id}/payment-methods/{method_id}
```
