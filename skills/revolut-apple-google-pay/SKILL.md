---
name: revolut-apple-google-pay
description: "Implements Apple Pay and Google Pay wallet buttons via the Revolut Merchant Web SDK. Activates when building wallet payments, adding Apple Pay or Google Pay buttons, or when user mentions Apple Pay, Google Pay, wallet payments, or digital wallets."
license: MIT
metadata:
  author: lartisan
---

# Apple Pay & Google Pay Integration

## When to Apply

Activate this skill when:

- Adding Apple Pay or Google Pay buttons to a checkout
- Implementing wallet-based payments
- Offering native device payment methods
- User mentions: Apple Pay, Google Pay, wallet, digital wallet, device payments

## Documentation

Apple Pay: https://developer.revolut.com/docs/sdks/merchant-web-sdk/apple-pay
Google Pay: https://developer.revolut.com/docs/sdks/merchant-web-sdk/google-pay

## Overview

Revolut's SDK renders Apple Pay and Google Pay buttons when the customer's browser and device support them. The buttons only appear automatically — you do not render a fallback; show an alternative payment method instead.

## Prerequisites

- **Apple Pay**: Requires HTTPS + domain verification with Apple via the Revolut dashboard.
- **Google Pay**: Requires HTTPS and registration in the Google Pay & Wallet Console.

## Backend: Create Order

```php
use Lartisan\Revolut\Facades\Revolut;

Route::post('/wallet/order', function (Request $request) {
    $order = Revolut::createOrder([
        'amount'      => $request->integer('amount'),
        'currency'    => 'GBP',
        'description' => 'Wallet checkout',
        'redirect_url'        => route('checkout.success'),
        'cancel_redirect_url' => route('checkout.cancel'),
    ]);

    return response()->json(['public_id' => $order->public_id]);
});
```

## Frontend: Apple Pay Button

```html
<div id="apple-pay-btn"></div>

<script src="https://merchant.revolut.com/embed.js"></script>
<script>
async function mountApplePay() {
    const res = await fetch('/wallet/order', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': document.querySelector('meta[name=csrf-token]').content,
        },
        body: JSON.stringify({ amount: 2999 }),
    });

    const { public_id } = await res.json();

    RevolutCheckout(public_id, 'sandbox').applePay({
        target: document.getElementById('apple-pay-btn'),
        buttonStyle: 'black',   // 'black' | 'white' | 'white-with-line'
        buttonType:  'buy',     // 'buy' | 'pay' | 'check-out' | 'book' | 'subscribe' | 'plain'
        onSuccess() {
            window.location.href = '/checkout/success';
        },
        onError(message) {
            console.error('Apple Pay failed:', message);
        },
    });
}

mountApplePay();
</script>
```

## Frontend: Google Pay Button

```html
<div id="google-pay-btn"></div>

<script src="https://merchant.revolut.com/embed.js"></script>
<script>
async function mountGooglePay() {
    const res = await fetch('/wallet/order', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': document.querySelector('meta[name=csrf-token]').content,
        },
        body: JSON.stringify({ amount: 2999 }),
    });

    const { public_id } = await res.json();

    RevolutCheckout(public_id, 'sandbox').googlePay({
        target: document.getElementById('google-pay-btn'),
        buttonColor: 'black',   // 'black' | 'white'
        buttonType:  'buy',     // 'buy' | 'pay' | 'checkout' | 'book' | 'subscribe' | 'plain'
        onSuccess() {
            window.location.href = '/checkout/success';
        },
        onError(message) {
            console.error('Google Pay failed:', message);
        },
    });
}

mountGooglePay();
</script>
```

## Showing Both with Fallback

The wallet buttons render asynchronously. Use `canMakePayment()` to check availability before deciding whether to show a fallback:

```html
<div id="apple-pay-btn"></div>
<div id="google-pay-btn"></div>
<div id="standard-checkout" style="display:none">
    <button id="pay-btn">Pay by card</button>
</div>

<script src="https://merchant.revolut.com/embed.js"></script>
<script>
async function mountWalletButtons() {
    const { public_id } = await fetch('/wallet/order', { method: 'POST', ... })
        .then(r => r.json());

    const instance = RevolutCheckout(public_id, 'sandbox');

    const callbacks = {
        onSuccess: () => { window.location.href = '/success'; },
        onError:   (msg) => { alert(msg); },
    };

    // Check support before mounting — avoids showing empty placeholders
    const [appleSupported, googleSupported] = await Promise.all([
        instance.applePay({ target: document.getElementById('apple-pay-btn'), ...callbacks })
            .canMakePayment().catch(() => false),
        instance.googlePay({ target: document.getElementById('google-pay-btn'), ...callbacks })
            .canMakePayment().catch(() => false),
    ]);

    if (!appleSupported && !googleSupported) {
        document.getElementById('standard-checkout').style.display = 'block';
    }
}

mountWalletButtons();
</script>
```

> **Note:** If `canMakePayment()` is not available in the SDK version you are using, use the `onImmediatePaymentMethodsReady` callback or observe DOM mutations on the target containers as a fallback detection strategy. Avoid `setTimeout` — it is unreliable on slow connections.

## Apple Pay Domain Verification (Production)

1. Download the domain verification file from your Revolut Business dashboard.
2. Serve it at: `https://yourdomain.com/.well-known/apple-developer-merchantid-domain-association`
3. In Laravel, add a route:

```php
Route::get('/.well-known/apple-developer-merchantid-domain-association', function () {
    return response()->file(storage_path('app/apple-pay-domain-association'))
        ->header('Content-Type', 'text/plain');
});
```

## Common Pitfalls

- Buttons only appear on supported devices/browsers — always provide a standard checkout fallback
- Apple Pay requires HTTPS + verified domain even in testing on real devices
- Google Pay requires your site to be registered in Google Pay console before production use
- Buttons render asynchronously — use `canMakePayment()` rather than `setTimeout` to detect support
- Using `'prod'` environment before completing platform registration
- Hardcoding `'sandbox'` in `RevolutCheckout(public_id, 'sandbox')` — pass the environment from your backend (see `revolut-integration` skill)