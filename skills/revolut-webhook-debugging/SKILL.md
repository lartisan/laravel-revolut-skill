---
name: revolut-webhook-debugging
description: "Debugs and hardens Revolut webhook delivery, signature verification, and post-payment confirmation flows in Laravel. Activates when troubleshooting invalid signatures, missing confirmations, webhook route issues, or ORDER_COMPLETED handling."
license: MIT
metadata:
  author: lartisan
---

# Revolut Webhook Debugging

## When to Apply

Activate this skill when:

- Debugging `Invalid webhook signature.` errors
- Troubleshooting missing or delayed `ORDER_COMPLETED` confirmations
- Checking the Revolut webhook route, CSRF bypass, or environment wiring
- Verifying raw request body handling for signature verification
- Testing webhook payloads, signatures, or idempotent order completion
- User mentions: webhook, signature, `Revolut-Signature`, `ORDER_COMPLETED`, payment confirmation, callback, or webhook debugging

## Package Architecture

The `revolut/laravel-integration` package handles webhooks automatically:

- The webhook route is registered by `RevolutServiceProvider` at the path set in `REVOLUT_WEBHOOK_ROUTE` (default: `revolut/webhook`).
- Signature verification is delegated to `Lartisan\Revolut\Services\WebhookService`.
- CSRF bypass must be configured in `bootstrap/app.php` for the webhook route.
- Payment is only fully confirmed after the webhook marks the local order as paid — `onSuccess` in the frontend SDK is not sufficient.

## Signature Verification Rules

### Always hash the exact raw body

Use the exact bytes from the incoming request body:

```php
$payload = $request->getContent();
$expected = hash_hmac('sha256', $payload, config('revolut.webhook_secret'));
```

Do **not**:

- recompute the body from `json_encode($request->all())`
- prettify, trim, or reorder JSON
- hash parsed form data instead of the raw request content

### Accept real Revolut header formats

The `Revolut-Signature` header may arrive as:

```text
v1=<hmac>
```

or as a compound header such as:

```text
t=1710000000, v1=<hmac>, v2=<rotated-signature>
```

Always extract the `v1=` value before comparing signatures.

### Compare safely

Use constant-time comparison:

```php
hash_equals($expectedSignature, $signature)
```

## Environment and Routing Checklist

When debugging a live webhook failure, verify these first:

1. `REVOLUT_WEBHOOK_SECRET` matches the signing secret in the Revolut dashboard.
2. The secret belongs to the same environment as `REVOLUT_ENVIRONMENT` (`sandbox` vs `production`).
3. The dashboard webhook URL matches `REVOLUT_WEBHOOK_ROUTE`.
4. The CSRF exemption in `bootstrap/app.php` matches the webhook route.
5. The app is reading the raw request body with `$request->getContent()`.

## Common Failure Modes

### `Invalid webhook signature.`

Usually caused by one of these:

- wrong webhook secret
- production secret used against sandbox webhooks, or the reverse
- signature computed from re-encoded JSON instead of the raw body
- code only handling `v1=<hash>` and not a compound signature header
- testing with a payload string that is not byte-for-byte identical to the request body sent

### `Missing Revolut-Signature header.`

Check that:

- the request really comes from Revolut
- the correct webhook endpoint is registered in the dashboard
- any reverse proxy preserves the `Revolut-Signature` header

### Webhook never reaches the controller

Check that:

- the route exists via `php artisan route:list --name=revolut.webhook`
- the webhook path in `.env` matches the path registered in the dashboard
- CSRF is excluded for that route in `bootstrap/app.php`

### Payment succeeds in UI but order is still pending

That usually means the frontend finished, but the server never processed `ORDER_COMPLETED`.

Verify:

- the webhook arrives and passes signature verification
- the local order can be found by `revolut_order_id`
- the handler marks the order as paid idempotently
- downstream jobs are not masking the real webhook result

## Testing Pattern

Build the payload string first, then sign that exact string:

```php
$payload = json_encode([
    'event' => 'ORDER_COMPLETED',
    'order_id' => 'order_test_123',
    'data' => [
        'state' => 'COMPLETED',
    ],
]);

$signature = 'v1='.hash_hmac('sha256', $payload, config('revolut.webhook_secret'));

$this->call('POST', route('revolut.webhook'), [], [], [], [
    'HTTP_Revolut-Signature' => $signature,
    'CONTENT_TYPE' => 'application/json',
], $payload)->assertOk();
```

### Test the compound header case too

```php
$compoundHeader = 't=1710000000, '.$signature.', v2=rotated-secret-placeholder';
```

### Fake downstream jobs when testing webhook confirmation

At the application layer, prefer:

```php
Bus::fake();
```

This keeps webhook tests focused on verification and order state changes instead of failing on queue-driven side effects like email delivery.

## Browser Test Guidance

For full checkout browser coverage in this project:

1. Create the order through the checkout UI.
2. Complete the hosted Revolut sandbox card flow.
3. Trigger or simulate the `ORDER_COMPLETED` webhook.
4. Revisit the success page and assert the payment is confirmed.

Without the webhook step, the success page may correctly remain in a processing state.

## Useful Verification Commands

```bash
# Confirm route is registered
php artisan route:list --name=revolut.webhook

# Run webhook-related tests
php artisan test --compact --filter=webhook

# Run checkout-related tests
php artisan test --compact --filter=checkout
```

## Typical File Locations

| Purpose | Typical path |
|---|---|
| Route registration | `routes/web.php` or `routes/api.php` |
| CSRF exemption | `bootstrap/app.php` |
| Webhook controller | `app/Http/Controllers/RevolutWebhookController.php` |
| Webhook service (package) | `vendor/revolut/laravel-integration/src/Services/WebhookService.php` |
| Webhook feature tests | `tests/Feature/WebhookTest.php` |
| Checkout browser tests | `tests/Browser/RevolutCheckoutTest.php` |