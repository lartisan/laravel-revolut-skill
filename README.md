# laravel-revolut-skill

Claude Code skills for integrating the [Revolut Merchant API](https://developer.revolut.com/docs/merchant-api/) in Laravel applications via the [`revolut/laravel-integration`](https://github.com/lartisan/revolut-laravel-integration) package.

## Requirements

- Laravel 10+
- PHP 8.2+
- [`revolut/laravel-integration`](https://github.com/lartisan/revolut-laravel-integration) installed in your project

## Installation

```bash
/plugin marketplace add lartisan/laravel-revolut-skill
/plugin install laravel-revolut-skill@lartisan
```

## Skills

| Skill | Description |
|---|---|
| `revolut-integration` | Core setup: installation, configuration, orders, refunds, customers, webhooks |
| `revolut-checkout` | Embedded popup checkout via `payWithPopup()` |
| `revolut-card-field` | Inline card form with full CSS control |
| `revolut-card-popup` | Revolut-hosted modal overlay for card entry |
| `revolut-pay` | One-click Revolut Pay button for Revolut account holders |
| `revolut-apple-google-pay` | Apple Pay and Google Pay wallet buttons |
| `revolut-pay-by-bank` | Open banking / account-to-account payments |
| `revolut-promotional-widgets` | Buy Now banners and payment badges (no order required) |
| `revolut-webhook-debugging` | Signature verification, CSRF setup, and ORDER_COMPLETED debugging |
| `revolut-merchant-payments` | Full API reference: order lifecycle, 3DS, self-hosted checkout, recurring payments, in-person terminals |

## Usage

Skills activate automatically based on context. You can also invoke them explicitly:

```
/laravel-revolut-skill:revolut-integration
/laravel-revolut-skill:revolut-webhook-debugging
```

## License

MIT
