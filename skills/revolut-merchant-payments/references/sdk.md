# Revolut Merchant Web SDK

## Table of Contents
1. Installation
2. Card Payments
3. Revolut Pay
4. Apple Pay
5. Google Pay
6. Pay by Bank
7. Widget Configuration Options

## 1. Installation

```html
<!-- CDN (all pages that need payments) -->
<script src="https://merchant.revolut.com/embed.js"></script>
```

Or via npm:
```bash
npm install @revolut/checkout
```

```js
import RevolutCheckout from '@revolut/checkout';
```

Initialize:
```js
const revolut = await RevolutCheckout(PUBLIC_KEY, 'prod');
// sandbox: RevolutCheckout(PUBLIC_KEY, 'sandbox')
```

## 2. Card Payments

```js
const card = revolut.card({
  target: document.getElementById('card-element'),
  onSuccess() { /* payment complete */ },
  onError(error) { /* handle error */ },
  savePaymentMethodFor: 'customer',  // optional: 'customer' | 'merchant'
});

// Trigger payment
await card.pay();
```

To use card with a specific order:
```js
const card = revolut.card({
  target: el,
  publicToken: orderPublicId,   // from POST /orders response
});
```

## 3. Revolut Pay

Revolut users pay with their Revolut balance or saved cards. Supports Fast Checkout (pre-fills shipping info).

```js
const revolutPay = revolut.revolutPay({
  target: document.getElementById('revolut-pay-button'),
  currency: 'EUR',
  totalAmount: 1000,
  onSuccess() {},
  onError(error) {},
  // Optional: Fast Checkout
  requestShipping: true,
  onShippingAddressChange(address, actions) {
    // update shipping options
    actions.resolve([{ id: 'standard', label: 'Standard', amount: 300 }]);
  },
});
```

## 4. Apple Pay

Requires:
1. Apple Pay merchant registration via Revolut dashboard
2. Domain verification file hosted at `/.well-known/apple-developer-merchantid-domain-association`

```js
const applePay = revolut.paymentRequest({
  target: el,
  request: {
    label: 'My Store',
    total: { amount: 1000, currency: 'EUR' },
  },
  buttonType: 'plain',     // 'plain' | 'buy' | 'donate' | 'check-out'
  buttonColor: 'black',
  onSuccess() {},
  onError(error) {},
});
```

Check availability before rendering:
```js
const canMakePayment = await revolut.paymentRequest.canMakePayment();
if (canMakePayment.applePay) {
  // show Apple Pay button
}
```

Register domain:
```http
POST /apple-pay/merchant-registration
{ "domain": "yourdomain.com" }
```

## 5. Google Pay

Google Pay is handled via the same `paymentRequest` widget — no extra registration needed.

```js
const canMakePayment = await revolut.paymentRequest.canMakePayment();
if (canMakePayment.googlePay) {
  // show Google Pay button
}
```

## 6. Pay by Bank

Direct bank transfer. **Not available in sandbox — production only.**
Does not support refunds via Merchant API.

```js
const payByBank = revolut.payByBank({
  target: document.getElementById('pay-by-bank'),
  onSuccess() {},
  onError(error) {},
  onCancel() {},
});
```

The widget opens a pop-up redirecting the customer to their bank to authorise the payment.

## 7. Widget Configuration Options

Common options across widgets:

| Option | Type | Description |
|---|---|---|
| `target` | HTMLElement | DOM element to mount widget into |
| `publicToken` | string | Order `public_id` from POST /orders |
| `onSuccess` | function | Called on successful payment |
| `onError` | function | Called with error object on failure |
| `onCancel` | function | Called when customer cancels |
| `savePaymentMethodFor` | `'customer'` \| `'merchant'` | Save card for future use |
| `locale` | string | UI language (e.g., `'en'`, `'ro'`) |

### Controlling which payment methods appear

Configure the order and visibility of payment methods in the Revolut Business dashboard under Merchant settings.
