---
name: revolut-integration
description: "Integrates Revolut Merchant API for payment processing in Laravel. Activates when setting up payments, configuring Revolut, creating orders, handling webhooks, processing refunds, or when user mentions Revolut, payment gateway, merchant API, checkout, or online payments."
license: MIT
metadata:
  author: lartisan
---

# Revolut Payment Integration

## When to Apply

Activate this skill when:

- Setting up Revolut payment integration
- Configuring API credentials and environment
- Creating payment orders or checkout flows
- Handling webhook notifications
- Processing refunds or captures
- Testing payment flows in sandbox
- Debugging Revolut API issues
- User mentions: Revolut, payment, checkout, merchant API, refund, webhook

## Documentation

Revolut Merchant API: https://developer.revolut.com/docs/merchant-api/

## Installation

### Initial Setup

Install via Composer:

```bash
composer require revolut/laravel-integration
```

Auto-discovery registers `RevolutServiceProvider` and the `Revolut` facade automatically.

Publish configuration:

```bash
php artisan vendor:publish --tag=revolut-config
```

Run migrations:

```bash
php artisan migrate
```

This creates `revolut_customers` and `revolut_orders` tables.

### Environment Configuration

Add to `.env`:

```env
REVOLUT_API_KEY=sk_sandbox_your_key_here
REVOLUT_ENVIRONMENT=sandbox          # or "production"

REVOLUT_WEBHOOK_SECRET=your_secret
REVOLUT_WEBHOOK_ROUTE=revolut/webhook
```

**Important:** Use sandbox credentials during development. Never commit API keys to version control.

## Configuration (`config/revolut.php`)

| Key | Default | Description |
|-----|---------|-------------|
| `api_key` | `''` | Your Revolut Merchant API key |
| `environment` | `sandbox` | `sandbox` or `production` |
| `webhook_secret` | `''` | HMAC signing secret from Revolut dashboard |
| `webhook_route` | `revolut/webhook` | URI for receiving webhooks |
| `timeout` | `30` | HTTP request timeout in seconds |

## SDK Environment String

The second argument to `RevolutCheckout(public_id, env)` must match your Revolut environment:

| `.env` value | SDK string |
|---|---|
| `sandbox` | `'sandbox'` |
| `production` | `'prod'` |

Pass it dynamically from PHP to avoid hardcoding:

```blade
{{-- In a Blade view --}}
<script>
const revolutEnv = '{{ config('revolut.environment') === 'production' ? 'prod' : 'sandbox' }}';
RevolutCheckout(publicId, revolutEnv)...
</script>
```

Or return it from your backend endpoint:

```php
return response()->json([
    'public_id' => $order->public_id,
    'env'       => config('revolut.environment') === 'production' ? 'prod' : 'sandbox',
]);
```

## Frontend Integration Options

The Revolut Merchant Web SDK supports 7 frontend integration methods. Choose based on your UX requirements:

| Integration | Best for | Skill |
|---|---|---|
| Revolut Checkout | Popup card form over your page | `revolut-checkout` |
| Revolut Pay | One-click button for Revolut account holders | `revolut-pay` |
| Apple Pay & Google Pay | Wallet buttons on supported devices | `revolut-apple-google-pay` |
| Pay by Bank | Open banking / account-to-account | `revolut-pay-by-bank` |
| Card Field | Inline card form with full style control | `revolut-card-field` |
| Card Popup | Revolut-hosted modal overlay | `revolut-card-popup` |
| Promotional Widgets | Buy now banners and payment badges | `revolut-promotional-widgets` |

All SDK methods require a `public_id` (available as `$order->public_id` after `Revolut::createOrder()`).
Load the SDK via `<script src="https://merchant.revolut.com/embed.js"></script>` (or `upsell/bundle.js` for widgets).

See individual skills for implementation details.

## Basic Usage

### Creating a Payment Order

```php
use Lartisan\Revolut\Facades\Revolut;

$order = Revolut::createOrder([
    'amount'             => 1000,        // Amount in smallest unit (1000 = £10.00)
    'currency'           => 'GBP',
    'description'        => 'Order #123',
    'redirect_url'       => route('payment.success'),
    'cancel_redirect_url' => route('payment.cancel'), // Must use cancel_redirect_url, not cancel_url
]);

// Redirect user to hosted checkout page
return redirect()->away($order->checkout_url);
```

### Checking Order Status

```php
$order = Revolut::getOrder('order_abc123');

if ($order->isCompleted()) {
    // Payment confirmed — fulfill order
}
```

Helper methods: `isCompleted()`, `isPending()`, `isAuthorised()`, `isCancelled()`.

Order states flow: `PENDING` → `PROCESSING` → `AUTHORISED` → `COMPLETED`

### Capturing Pre-authorised Payments

```php
// Create order with manual capture
$order = Revolut::createOrder([
    'amount'       => 5000,
    'currency'     => 'GBP',
    'capture_mode' => 'manual',
]);

// Capture when ready (e.g. on shipment)
Revolut::captureOrder($order->revolut_order_id);

// Partial capture
Revolut::captureOrder($order->revolut_order_id, ['amount' => 3000]);
```

### Processing Refunds

```php
// Full refund
Revolut::refundOrder('order_abc123');

// Partial refund
Revolut::refundOrder('order_abc123', [
    'amount'      => 500,
    'description' => 'Customer request',
]);
```

### Customers

```php
// Create customer
$customer = Revolut::createCustomer([
    'email'     => 'jane@example.com',
    'full_name' => 'Jane Smith',
    'phone'     => '+447911123456',
]);

// Link to an order
$order = Revolut::createOrder([
    'amount'      => 2000,
    'currency'    => 'GBP',
    'customer_id' => $customer->revolut_customer_id,
]);
```

## Webhook Handling

### Setup

The package registers the webhook route automatically. Configure the URL in the Revolut Business dashboard:

```
https://yourdomain.com/revolut/webhook
```

Copy the signing secret → set `REVOLUT_WEBHOOK_SECRET` in `.env`.

### Listening to Events

```php
use Lartisan\Revolut\Events\OrderCompleted;

Event::listen(OrderCompleted::class, function (OrderCompleted $event) {
    $orderId = $event->getOrderId();
    // Fulfill the order…
});
```

Available events: `OrderCompleted`, `OrderAuthorised`, `OrderCancelled`, `OrderPaymentFailed`.

### Webhook Signature Validation

All webhooks are automatically validated via HMAC-SHA256. Invalid signatures are rejected with 403.

The `Revolut-Signature` header uses a `v1=` prefix (e.g. `v1=<hmac_sha256>`). The package handles this automatically.

## Facade Reference

```php
Revolut::createOrder(array $params): RevolutOrder
Revolut::getOrder(string $orderId): RevolutOrder
Revolut::captureOrder(string $orderId, array $params = []): RevolutOrder
Revolut::cancelOrder(string $orderId): RevolutOrder
Revolut::refundOrder(string $orderId, array $params = []): array
Revolut::listOrders(array $filters = []): array
Revolut::createCustomer(array $params): RevolutCustomer
Revolut::getCustomer(string $customerId): RevolutCustomer
Revolut::updateCustomer(string $customerId, array $params): RevolutCustomer
Revolut::deleteCustomer(string $customerId): bool
Revolut::listCustomers(array $filters = []): array
```

## Models

### `RevolutOrder`

| Property | Type | Description |
|----------|------|-------------|
| `revolut_order_id` | `string` | Revolut's internal order ID (used for API calls) |
| `public_id` | `string` | Public token passed to the frontend SDK (`RevolutCheckout(public_id, ...)`) |
| `state` | `string` | `PENDING`, `AUTHORISED`, `COMPLETED`, `CANCELLED`, `FAILED` |
| `amount` | `int` | Amount in smallest currency unit |
| `currency` | `string` | ISO 4217 currency code |
| `checkout_url` | `string\|null` | Hosted checkout URL (for redirect-based flows) |
| `formatted_amount` | `string` | Accessor: e.g. `"10.00 GBP"` |

### `RevolutCustomer`

| Property | Type | Description |
|----------|------|-------------|
| `revolut_customer_id` | `string` | Revolut's customer ID |
| `email` | `string\|null` | Customer email |
| `full_name` | `string\|null` | Customer name |

## Error Handling

All API methods throw `Lartisan\Revolut\Exceptions\RevolutException` on failure:

```php
use Lartisan\Revolut\Exceptions\RevolutException;

try {
    $order = Revolut::createOrder([...]);
} catch (RevolutException $e) {
    $e->getMessage();     // Human-readable error
    $e->getStatusCode();  // HTTP status code (404, 422, etc.)
    $e->getErrors();      // Validation errors array
    $e->isUnauthorized(); // true if 401
    $e->isNotFound();     // true if 404
    $e->isRateLimited();  // true if 429
}
```

## MCP Server Integration

If using Claude Desktop or Claude Code, configure MCP for live testing:

```json
{
  "mcpServers": {
    "revolut": {
      "command": "php",
      "args": ["vendor/revolut/laravel-integration/mcp/bin/revolut-mcp-server"],
      "env": {
        "REVOLUT_API_KEY": "sk_sandbox_xxxxx",
        "REVOLUT_ENVIRONMENT": "sandbox"
      }
    }
  }
}
```

## Common Pitfalls

- **Amount is in the smallest currency unit** — £10.00 = `1000` (pence), not `10`
- Using `customer_email` instead of `email` in `createOrder()` (silently wrong field name)
- Using `cancel_url` instead of `cancel_redirect_url` (silently ignored by the API)
- Not validating webhook signatures (security risk)
- Using production API keys in `.env` committed to Git
- Not handling webhook race conditions (webhook arrives before redirect)
- Not storing `order_id` for reconciliation
- Reusing or caching `checkout_url` — create a fresh order each time
- Webhook URL must be HTTPS in production; use `ngrok` for local testing
- Tokens expire — store and refresh Revolut customer IDs; don't recreate customers every session

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Invalid API key" (401) | Check `REVOLUT_API_KEY` in `.env` matches correct environment |
| Webhook signature fails | Verify `REVOLUT_WEBHOOK_SECRET` matches Revolut dashboard |
| HTTP 422 on order creation | Call `$e->getErrors()` — check currency code and amount |
| Orders not in database | Run `php artisan migrate` |
| Checkout URL not working | Don't reuse URLs — `checkout_url` expires |
| Webhook not received | Check URL in Revolut dashboard; must be HTTPS in production |