# In-Person Payments

## Overview

Revolut supports in-person payments via:

1. **Push Payments to Terminal** — server-to-server REST API pushes a payment intent to a physical Revolut Terminal device
2. **Revolut Pay QR Code** — customer scans QR on terminal screen with Revolut app

Both require a Revolut Terminal device and Merchant account with in-person payments enabled.

## Push Payments to Terminal

A pure server-to-server integration. No terminal SDK, drivers, or physical cables needed — only internet connectivity.

### Flow

```
1. Your POS/server creates an order via Merchant API
2. Push payment intent to the terminal device
3. Customer taps/inserts card on the terminal
4. Terminal processes and reports back via webhook
```

### Step 1: Get Terminal Device IDs

Before pushing a payment, you need the `device_id` of the target terminal:

```http
GET /terminals
Authorization: Bearer <SECRET_KEY>
```

Returns a list of registered terminal devices with their `device_id`, `name`, `status`, and `location`. Pick the one you want to push to.

### Step 2: Create Order

```http
POST /orders
Authorization: Bearer <SECRET_KEY>

{
  "amount": 3500,
  "currency": "RON",
  "capture_mode": "automatic",
  "description": "Table 12 - Lunch"
}
```

### Step 3: Push to Terminal

```http
POST /terminal-payments
Authorization: Bearer <SECRET_KEY>

{
  "order_id": "<order_id>",
  "device_id": "<terminal_device_id>"
}
```

The terminal displays the payment amount and prompts the customer to pay.

### Step 4: Track Payment Status

Poll the order or listen for webhook:

```http
GET /orders/{order_id}
```

Wait for `state: "completed"`.

Or configure webhook for `ORDER_COMPLETED`.

### Cancel a Terminal Payment

If the customer walks away before paying:

```http
POST /terminal-payments/{terminal_payment_id}/cancel
```

## Revolut Pay QR Code (In-Person)

The Revolut Terminal can display a QR code on the payment screen. The customer scans it with the Revolut app and completes the payment using their Revolut balance or RevPoints.

This is configured at the terminal level — no additional API integration needed beyond the standard push payment flow. The terminal automatically displays the QR when Revolut Pay is enabled in the merchant dashboard.

## Manage Payments on Terminal

```http
GET  /terminal-payments/{id}       # retrieve specific terminal payment
GET  /terminal-payments            # list all (filter by device_id, status)
POST /terminal-payments/{id}/cancel
```

## Tips

- Test with sandbox using a virtual terminal device (available in sandbox dashboard)
- Always handle `ORDER_PAYMENT_DECLINED` webhook for failed in-person payments
- For multi-terminal setups, use `location_id` to filter terminals by location
