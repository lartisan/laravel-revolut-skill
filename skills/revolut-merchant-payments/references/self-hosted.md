# Self-Hosted Checkout & 3DS Flow

Use when building a fully custom checkout UI where you manage card data collection and 3DS challenges manually.

## Flow Overview

```
1. Customer fills in card details (your UI + Revolut Widget for card fields)
2. Your server: POST /orders → get order_id + public_id
3. Your server: POST /orders/{id}/payments → start payment
4. Check payment state:
   - authentication_challenge → redirect to 3DS
   - authorised → capture (if manual) or complete
   - completed → done
```

## Step 1: Create Order

```http
POST /orders
Authorization: Bearer <SECRET_KEY>

{
  "amount": 2500,
  "currency": "EUR",
  "capture_mode": "manual",
  "description": "Order #456"
}
```

Save `id` (server use) and `public_id` (client use).

## Step 2: Collect Card Data

Use the Revolut Card Element to collect PCI-compliant card data. The widget tokenizes everything in the browser — you never handle raw card numbers.

```js
const revolut = await RevolutCheckout(PUBLIC_KEY);
const card = revolut.card({
  target: document.getElementById('card-element'),
  publicToken: orderPublicId,  // public_id from Step 1
});
```

On form submit, call `card.submit()` with a handler that receives the tokenized card data and forwards it to your server:

```js
document.getElementById('pay-btn').addEventListener('click', () => {
  card.submit({
    // card.submit POSTs directly to Revolut on your behalf in standard mode,
    // but in self-hosted mode you retrieve tokens via the onSuccess callback
    // and send them to your own backend endpoint
    onSuccess({ cardNumberToken, cvvToken, expiryMonth, expiryYear, cardholderName }) {
      fetch('/your-server/pay', {
        method: 'POST',
        body: JSON.stringify({ cardNumberToken, cvvToken, expiryMonth, expiryYear, cardholderName }),
      });
    },
    onError(error) { /* show error */ },
  });
});
```

## Step 3: Initiate Payment (Server-Side)

Your server receives the tokens from Step 2 and posts them to Revolut. Also include `return_url` — this is where Revolut redirects the customer after 3DS completes.

```http
POST /orders/{order_id}/payments
Authorization: Bearer <SECRET_KEY>

{
  "payment_method": {
    "type": "card",
    "card_number_token": "<token from widget>",
    "expiry_month": "12",
    "expiry_year": "2027",
    "security_code_token": "<cvv token>",
    "cardholder_name": "Ion Popescu"
  },
  "browser_info": {
    "accept_header": "text/html",
    "color_depth": 24,
    "java_enabled": false,
    "language": "en-US",
    "screen_height": 900,
    "screen_width": 1440,
    "timezone": -120,
    "user_agent": "<user agent string>"
  },
  "return_url": "https://yoursite.com/checkout/complete"
}
```

`browser_info` is required for 3DS2 fingerprinting. `return_url` is where the bank redirects after the 3DS challenge — Revolut appends the `order_id` as a query parameter so you know which order to look up.

## Step 4: Handle 3DS Challenge

If the response state is `authentication_challenge`:

```json
{
  "state": "authentication_challenge",
  "authentication_challenge": {
    "acs_url": "https://acs.bank.com/3ds/challenge",
    "creq": "<challenge request token>"
  }
}
```

Redirect the customer to `acs_url` with the `creq` parameter:

```html
<form method="POST" action="{acs_url}">
  <input type="hidden" name="creq" value="{creq}" />
</form>
<script>document.forms[0].submit();</script>
```

After 3DS, the bank redirects back to your `return_url` (set in Step 3). Poll the order:

```http
GET /orders/{order_id}
```

Wait for state to reach `authorised` or `completed`.

## Step 5: Capture (manual mode)

```http
POST /orders/{order_id}/capture
Authorization: Bearer <SECRET_KEY>

{ "amount": 2500 }
```

## Error Handling

| State | Meaning | Action |
|---|---|---|
| `declined` | Hard decline | Show error, do not retry same card |
| `soft_declined` | Soft decline (e.g., insufficient funds) | Suggest customer try again or use different method |
| `failed` | Technical failure | Retry with exponential backoff |
| `authentication_challenge` | 3DS required | Redirect to ACS URL |

## Idempotency

Add `Idempotency-Key` header to avoid duplicate charges on retries:

```
Idempotency-Key: <unique-uuid-per-request>
```
