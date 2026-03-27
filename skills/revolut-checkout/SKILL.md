---
name: revolut-checkout
description: "Implements Revolut Checkout embedded payment widget. Activates when building embedded checkout, customizing payment UI, handling payment events, or when user mentions embedded checkout, payment widget, custom checkout UI, or Revolut Checkout SDK."
license: MIT
metadata:
  author: lartisan
---

# Revolut Checkout Integration

## When to Apply

Activate this skill when:

- Building embedded or popup checkout experience
- Handling payment success/failure events on the frontend
- Implementing checkout without a full-page redirect
- User mentions: embedded checkout, Revolut Checkout, payment widget, popup checkout, custom UI

## Documentation

Revolut Checkout SDK: https://developer.revolut.com/docs/sdks/merchant-web-sdk/revolut-checkout

## Overview

Revolut Checkout embeds the payment interface in your page via a popup. The user stays on your site while the payment form appears in a Revolut-hosted overlay.

## Backend: Create Order and Return `public_id`

```php
use Lartisan\Revolut\Facades\Revolut;

Route::post('/checkout/order', function (Request $request) {
    $order = Revolut::createOrder([
        'amount'      => $request->integer('amount'), // pence/cents
        'currency'    => 'GBP',
        'description' => 'Order #'.$request->integer('order_id'),
        'redirect_url'        => route('checkout.success'),
        'cancel_redirect_url' => route('checkout.cancel'),
        'merchant_order_ext_ref' => (string) $request->integer('order_id'),
    ]);

    return response()->json(['public_id' => $order->public_id]);
});
```

## Frontend: Popup Checkout

```html
<button id="pay-btn" type="button">Pay now</button>

<script src="https://merchant.revolut.com/embed.js"></script>
<script>
document.getElementById('pay-btn').addEventListener('click', async () => {
    const res = await fetch('/checkout/order', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': document.querySelector('meta[name=csrf-token]').content,
        },
        body: JSON.stringify({ amount: 1999, order_id: 42 }),
    });

    const { public_id } = await res.json();

    RevolutCheckout(public_id, 'sandbox').payWithPopup({
        onSuccess() {
            window.location.href = '/checkout/success';
        },
        onError(message) {
            alert('Payment failed: ' + message);
        },
        onCancel() {
            console.log('Payment cancelled');
        },
    });
});
</script>
```

## Laravel Blade Component

```blade
{{-- After creating the order and passing public_id to the view: --}}
<x-revolut::checkout
    :public-id="$order->public_id"
    button-text="Pay £19.99"
    button-class="btn btn-primary"
/>
```

## Customisation Options

```javascript
RevolutCheckout(public_id, 'sandbox').payWithPopup({
    name:   'Jane Smith',
    email:  'jane@example.com',
    phone:  '+447700900000',
    locale: 'en',           // BCP 47 language tag
    savePaymentMethodFor: 'merchant', // persist card for future charges

    onSuccess() { /* fulfill order */ },
    onError(message) { /* show error */ },
    onCancel() { /* restore cart */ },
});
```

## Environment Values

| Config value | SDK environment |
|---|---|
| `sandbox` | `'sandbox'` |
| `production` | `'prod'` |

Use `Revolut::sdk()->getEnvironment()` to get the correct string from Laravel config.

## Confirm Payment Server-Side

Never rely solely on `onSuccess`. Always confirm via webhook:

```php
use Lartisan\Revolut\Events\OrderCompleted;

Event::listen(OrderCompleted::class, function (OrderCompleted $event) {
    $order = RevolutOrder::where('revolut_order_id', $event->getOrderId())->first();
    $order?->update(['status' => 'paid']);
});
```

## Common Pitfalls

- Using `revolut_order_id` instead of `public_id` — the SDK needs `public_id`
- Caching `public_id` between sessions — create a fresh order each time
- Using `'prod'` environment in development — causes real charges
- Trusting `onSuccess` without webhook validation
- Not setting CSRF token in `fetch()` headers
- Forgetting `cancel_redirect_url` (using `cancel_url` is silently ignored)