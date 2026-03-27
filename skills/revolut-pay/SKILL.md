---
name: revolut-pay
description: "Implements Revolut Pay one-click payment button. Activates when building express checkout, one-click payments, fast checkout buttons, or when user mentions Revolut Pay, express payments, one-click, or saved payment methods."
license: MIT
metadata:
  author: lartisan
---

# Revolut Pay Integration

## When to Apply

Activate this skill when:

- Building a one-click or express checkout button
- Using saved Revolut account payment methods
- Adding a fast-pay option alongside the regular checkout
- User mentions: Revolut Pay, one-click, express checkout, fast payments

## Documentation

Revolut Pay: https://developer.revolut.com/docs/sdks/merchant-web-sdk/revolut-pay

## Overview

Revolut Pay renders a branded button that lets Revolut account holders pay in one click using their saved payment details or Revolut balance.

## Backend: Create Order

```php
use Lartisan\Revolut\Facades\Revolut;

Route::post('/express/order', function (Request $request) {
    $order = Revolut::createOrder([
        'amount'      => $request->integer('amount'),
        'currency'    => 'EUR',
        'description' => 'Express checkout',
        'redirect_url'        => route('checkout.success'),
        'cancel_redirect_url' => route('checkout.cancel'),
    ]);

    return response()->json(['public_id' => $order->public_id]);
});
```

## Frontend: Revolut Pay Button

```html
<div id="revolut-pay"></div>

<script src="https://merchant.revolut.com/embed.js"></script>
<script>
async function mountRevolutPay() {
    const res = await fetch('/express/order', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': document.querySelector('meta[name=csrf-token]').content,
        },
        body: JSON.stringify({ amount: 4999 }),
    });

    const { public_id } = await res.json();

    RevolutCheckout(public_id, 'sandbox').revolutPay({
        target: document.getElementById('revolut-pay'),
        buttonStyle: {
            size:    'large',  // 'small' | 'medium' | 'large'
            variant: 'dark',   // 'light' | 'dark'
            radius:  'small',  // 'none'  | 'small'  | 'round'
        },
        onSuccess() {
            window.location.href = '/checkout/success';
        },
        onError(message) {
            alert('Payment failed: ' + message);
        },
    });
}

mountRevolutPay();
</script>
```

## Laravel Blade Component

```blade
{{-- Create order first, pass public_id to view --}}
<x-revolut::pay-button
    :public-id="$order->public_id"
    size="large"
    variant="dark"
    radius="small"
/>
```

## Button Style Options

| Property | Values |
|---|---|
| `size` | `small`, `medium`, `large` |
| `variant` | `light`, `dark` |
| `radius` | `none`, `small`, `round` |

## With Delivery Address Collection

```javascript
RevolutCheckout(public_id, 'sandbox').revolutPay({
    target: document.getElementById('revolut-pay'),
    delivery: {
        collectPhoneNumber:    true,
        collectShippingAddress: true,
    },
    onSuccess() { window.location.href = '/success'; },
    onError(msg) { console.error(msg); },
});
```

## Product Page Express Checkout

```php
// Controller
public function expressOrder(Product $product): JsonResponse
{
    $order = Revolut::createOrder([
        'amount'      => $product->price_pence,
        'currency'    => 'GBP',
        'description' => $product->name,
        'merchant_order_ext_ref' => 'product-'.$product->id.'-'.uniqid(),
    ]);

    return response()->json(['public_id' => $order->public_id]);
}
```

```blade
{{-- Blade view --}}
<div id="revolut-pay-{{ $product->id }}"></div>

<script src="https://merchant.revolut.com/embed.js"></script>
<script>
fetch('/products/{{ $product->id }}/express-order', {
    method: 'POST',
    headers: { 'X-CSRF-TOKEN': '{{ csrf_token() }}' },
})
.then(r => r.json())
.then(({ public_id }) => {
    RevolutCheckout(public_id, '{{ Revolut::sdk()->getEnvironment() }}').revolutPay({
        target: document.getElementById('revolut-pay-{{ $product->id }}'),
        onSuccess: () => { window.location.href = '/orders'; },
    });
});
</script>
```

## Common Pitfalls

- Not creating a fresh order per button mount — `public_id` expires
- Mounting the button before the DOM element exists — wrap in `DOMContentLoaded`
- Forgetting delivery options for physical goods
- Relying on `onSuccess` alone without webhook confirmation
- Hardcoding `'sandbox'` in JS — pass the environment value from your backend so it automatically switches to `'prod'` in production (see `revolut-integration` skill for the pattern)