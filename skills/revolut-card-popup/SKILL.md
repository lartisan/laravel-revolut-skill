---
name: revolut-card-popup
description: "Implements the Revolut Card Popup modal checkout. Activates when building a popup/modal payment flow, showing a card form in an overlay, or when user mentions card popup, modal checkout, popup payment, or overlay checkout."
license: MIT
metadata:
  author: lartisan
---

# Card Popup Integration

## When to Apply

Activate this skill when:

- Opening a card payment form in a modal/popup over your page
- Keeping users on your page during checkout without a full-page redirect
- Combining quick popup UX with a custom trigger button
- User mentions: card popup, modal checkout, overlay payment, popup card form

## Documentation

Revolut Checkout (popup mode): https://developer.revolut.com/docs/sdks/merchant-web-sdk/revolut-checkout

## Overview

The Card Popup opens a Revolut-hosted modal overlay for card entry. It is the simplest integration — one JS call opens the secure payment form over your page. Use `payWithPopup()` for the popup; use `createCardField()` for an inline form (see the `revolut-card-field` skill).

## Backend: Create Order

```php
use Lartisan\Revolut\Facades\Revolut;

Route::post('/popup/order', function (Request $request) {
    $order = Revolut::createOrder([
        'amount'      => $request->integer('amount'),
        'currency'    => 'USD',
        'description' => 'Popup checkout',
        'redirect_url'        => route('checkout.success'),
        'cancel_redirect_url' => route('checkout.cancel'),
    ]);

    return response()->json(['public_id' => $order->public_id]);
});
```

## Frontend: Open the Popup on Button Click

```html
<button id="open-checkout" type="button">Pay $29.99</button>
<p id="payment-status"></p>

<script src="https://merchant.revolut.com/embed.js"></script>
<script>
document.getElementById('open-checkout').addEventListener('click', async () => {
    const btn = document.getElementById('open-checkout');
    btn.disabled = true;
    btn.textContent = 'Loading…';

    try {
        const { public_id } = await fetch('/popup/order', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-TOKEN': document.querySelector('meta[name=csrf-token]').content,
            },
            body: JSON.stringify({ amount: 2999 }),
        }).then(r => r.json());

        RevolutCheckout(public_id, 'sandbox').payWithPopup({
            onSuccess() {
                document.getElementById('payment-status').textContent = 'Payment successful!';
                window.location.href = '/checkout/success';
            },
            onError(message) {
                document.getElementById('payment-status').textContent = 'Error: ' + message;
                btn.disabled = false;
                btn.textContent = 'Pay $29.99';
            },
            onCancel() {
                btn.disabled = false;
                btn.textContent = 'Pay $29.99';
            },
        });
    } catch (err) {
        btn.disabled = false;
        btn.textContent = 'Pay $29.99';
        console.error(err);
    }
});
</script>
```

## Passing Customer Details

Pre-fill customer information to speed up the checkout experience:

```javascript
RevolutCheckout(public_id, 'sandbox').payWithPopup({
    name:   'Jane Smith',
    email:  'jane@example.com',
    phone:  '+14155550100',
    locale: 'en',   // BCP 47 language tag
    savePaymentMethodFor: 'merchant',  // saves card for future charges

    onSuccess() { window.location.href = '/success'; },
    onError(msg)  { alert(msg); },
    onCancel()    { /* re-enable button */ },
});
```

## Blade Component

```blade
{{-- Create order, pass $order->public_id to view --}}
<x-revolut::checkout
    :public-id="$order->public_id"
    button-text="Pay now"
    button-class="btn btn-primary btn-lg"
/>
```

## Popup vs Card Field Comparison

| | Popup | Card Field |
|---|---|---|
| UI location | Overlay over your page | Inline in your page |
| Styling control | Revolut-hosted, limited | Full CSS control |
| Setup complexity | Minimal | Moderate |
| Best for | Quick integration | Custom design systems |

## Common Pitfalls

- Creating the order on page load instead of on button click — `public_id` may expire
- Not disabling the trigger button while the popup is open — users may open it twice
- Catching network errors but not SDK errors — wrap the whole block in try/catch
- Forgetting `onCancel` — the button stays disabled if users close the popup
- Using `checkout_url` as a fallback — it is a different redirect-based flow
- Hardcoding `'sandbox'` in `RevolutCheckout(public_id, 'sandbox')` — pass the environment from your backend so production uses `'prod'` (see `revolut-integration` skill)