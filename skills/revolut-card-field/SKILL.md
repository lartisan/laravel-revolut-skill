---
name: revolut-card-field
description: "Implements an embedded card input field via the Revolut Merchant Web SDK. Activates when building a custom card form, embedding card entry in your own UI, styling card inputs, or when user mentions card field, custom card form, embedded card input, or card tokenisation."
license: MIT
metadata:
  author: lartisan
---

# Card Field Integration

## When to Apply

Activate this skill when:

- Building a fully custom card entry form embedded in your page
- Styling card inputs to match your design system
- Avoiding a popup/redirect for card payments
- User mentions: card field, custom card form, embedded card entry, card tokenisation, inline card

## Documentation

Card Field: https://developer.revolut.com/docs/sdks/merchant-web-sdk/card-field

## Overview

Card Field embeds Revolut's secure card input form (number, expiry, CVV) directly in your page. Unlike the popup, the form is mounted inline and styled by you. Card data is handled by Revolut's secure iframe — you never touch raw card numbers.

## Backend: Create Order

```php
use Lartisan\Revolut\Facades\Revolut;

Route::post('/card-field/order', function (Request $request) {
    $order = Revolut::createOrder([
        'amount'      => $request->integer('amount'),
        'currency'    => 'EUR',
        'description' => 'Card payment',
        'redirect_url'        => route('checkout.success'),
        'cancel_redirect_url' => route('checkout.cancel'),
    ]);

    return response()->json(['public_id' => $order->public_id]);
});
```

## Frontend: Embedded Card Field

```html
<form id="card-form">
    <div id="card-field"></div>
    <button type="submit">Pay now</button>
</form>

<script src="https://merchant.revolut.com/embed.js"></script>
<script>
async function initCardField() {
    const { public_id } = await fetch('/card-field/order', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': document.querySelector('meta[name=csrf-token]').content,
        },
        body: JSON.stringify({ amount: 1299 }),
    }).then(r => r.json());

    const instance = RevolutCheckout(public_id, 'sandbox');

    const cardField = instance.createCardField({
        target: document.getElementById('card-field'),
        locale: 'en',
        hidePostcodeField: false,

        onSuccess() {
            window.location.href = '/checkout/success';
        },
        onError(message) {
            document.getElementById('error-msg').textContent = message;
        },
        onValidation(errors) {
            // errors is an array of field validation messages
            console.log('Validation:', errors);
        },
    });

    document.getElementById('card-form').addEventListener('submit', (e) => {
        e.preventDefault();
        cardField.submit(); // Triggers payment
    });
}

initCardField();
</script>
<p id="error-msg" style="color:red"></p>
```

## Styling the Card Field

```javascript
const cardField = instance.createCardField({
    target: document.getElementById('card-field'),
    styles: {
        default: {
            color:       '#1a1a1a',
            fontSize:    '16px',
            fontFamily:  'Inter, sans-serif',
            '::placeholder': { color: '#9ca3af' },
        },
        invalid: {
            color: '#ef4444',
        },
    },
});
```

## Laravel Controller with Validation

```php
namespace App\Http\Controllers;

use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Lartisan\Revolut\Facades\Revolut;

class CardFieldCheckoutController extends Controller
{
    public function createOrder(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'amount'   => ['required', 'integer', 'min:50'],
            'currency' => ['required', 'string', 'size:3'],
        ]);

        $order = Revolut::createOrder([
            'amount'      => $validated['amount'],
            'currency'    => $validated['currency'],
            'description' => 'Checkout #'.uniqid(),
            'redirect_url'        => route('checkout.success'),
            'cancel_redirect_url' => route('checkout.cancel'),
        ]);

        return response()->json(['public_id' => $order->public_id]);
    }
}
```

## `cardField.submit()` vs `payWithPopup()`

| Method | UI style | User experience |
|---|---|---|
| `createCardField()` | Inline, fully styled by you | Card entry on your page |
| `payWithPopup()` | Revolut-hosted overlay | Popup over your page |

Use `createCardField()` when you want full visual control.

## Common Pitfalls

- Calling `cardField.submit()` before the SDK has loaded — wait for `onReady` callback
- Forgetting to handle `onValidation` — users won't know their card details are wrong
- Trying to style the inner iframe content via CSS — use the `styles` option instead
- Not catching `onError` — silent failures confuse users
- Reusing the same `public_id` after an error — create a new order
- Hardcoding `'sandbox'` in `RevolutCheckout(public_id, 'sandbox')` — pass the environment from your backend to avoid real charges in production (see `revolut-integration` skill)