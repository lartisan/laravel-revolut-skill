---
name: revolut-pay-by-bank
description: "Implements Pay by Bank (open banking) via the Revolut Merchant Web SDK. Activates when building bank transfer payments, open banking integrations, or when user mentions Pay by Bank, bank transfer, open banking, or account-to-account payments."
license: MIT
metadata:
  author: lartisan
---

# Pay by Bank Integration

## When to Apply

Activate this skill when:

- Accepting bank transfer payments (account-to-account)
- Integrating open banking as a payment method
- Reducing card processing fees with direct bank debits
- User mentions: Pay by Bank, open banking, bank transfer, A2A payments

## Documentation

Pay by Bank: https://developer.revolut.com/docs/sdks/merchant-web-sdk/pay-by-bank

## Overview

Pay by Bank (open banking) lets customers authorise a direct bank debit through their banking app. No card details are needed, and settlement is typically faster than card payments.

## Backend: Create Order

```php
use Lartisan\Revolut\Facades\Revolut;

Route::post('/bank/order', function (Request $request) {
    $order = Revolut::createOrder([
        'amount'      => $request->integer('amount'),
        'currency'    => 'GBP',
        'description' => 'Bank payment',
        'redirect_url'        => route('checkout.success'),
        'cancel_redirect_url' => route('checkout.cancel'),
    ]);

    return response()->json(['public_id' => $order->public_id]);
});
```

## Frontend: Pay by Bank Button

```html
<div id="pay-by-bank"></div>

<script src="https://merchant.revolut.com/embed.js"></script>
<script>
async function mountPayByBank() {
    const res = await fetch('/bank/order', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': document.querySelector('meta[name=csrf-token]').content,
        },
        body: JSON.stringify({ amount: 9999 }),
    });

    const { public_id } = await res.json();

    RevolutCheckout(public_id, 'sandbox').payByBank({
        target: document.getElementById('pay-by-bank'),
        onSuccess() {
            window.location.href = '/checkout/success';
        },
        onError(message) {
            console.error('Pay by Bank failed:', message);
        },
        onCancel() {
            console.log('Pay by Bank cancelled');
        },
    });
}

mountPayByBank();
</script>
```

## Multi-Payment Method Checkout

```html
<div id="pay-by-bank-btn"></div>
<hr>
<button id="card-btn" type="button">Pay by card instead</button>

<script src="https://merchant.revolut.com/embed.js"></script>
<script>
async function initCheckout() {
    const { public_id } = await fetch('/bank/order', { method: 'POST', ... })
        .then(r => r.json());

    const instance = RevolutCheckout(public_id, 'sandbox');
    const callbacks = {
        onSuccess: () => { window.location.href = '/success'; },
        onError: (msg) => { alert(msg); },
    };

    // Render Pay by Bank
    instance.payByBank({ target: document.getElementById('pay-by-bank-btn'), ...callbacks });

    // Fall back to card popup
    document.getElementById('card-btn').addEventListener('click', () => {
        instance.payWithPopup(callbacks);
    });
}

initCheckout();
</script>
```

## Availability

Pay by Bank is available in supported markets only. Check the Revolut dashboard for your country's availability before implementing.

## Webhook: Confirm Payment

Bank payments may take additional time to settle. Listen to the `ORDER_COMPLETED` event:

```php
use Lartisan\Revolut\Events\OrderCompleted;

Event::listen(OrderCompleted::class, function (OrderCompleted $event) {
    $order = RevolutOrder::where('revolut_order_id', $event->getOrderId())->first();
    if ($order) {
        $order->update(['status' => 'paid']);
        // Notify customer
    }
});
```

## Common Pitfalls

- Pay by Bank availability varies by country — check market support first
- Settlement time is longer than cards — don't fulfil instantly on `onSuccess`
- Always confirm via `ORDER_COMPLETED` webhook before fulfilling orders
- The bank selection UI may redirect users to their banking app briefly
- Pay by Bank is **production-only** — it is not available in the sandbox environment
- Hardcoding `'sandbox'` in `RevolutCheckout(public_id, 'sandbox')` — pass the environment from your backend (see `revolut-integration` skill)