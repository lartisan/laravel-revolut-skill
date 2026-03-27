---
name: revolut-promotional-widgets
description: "Implements Revolut promotional widgets (banners, badges, instalments). Activates when adding payment banners, promotional messages, instalment options, or when user mentions promotional widgets, buy now pay later banners, payment badges, or Revolut promotional UI."
license: MIT
metadata:
  author: lartisan
---

# Revolut Promotional Widgets

## When to Apply

Activate this skill when:

- Adding promotional payment banners to product pages
- Showing instalment/BNPL payment option messaging
- Displaying payment method badges
- User mentions: promotional widgets, buy now pay later, instalment banner, payment badge, Revolut widget

## Documentation

Promotional Widgets: https://developer.revolut.com/docs/sdks/merchant-web-sdk/widgets

## Overview

Revolut's promotional widgets render marketing UI elements — banners and badges — that inform customers about available payment options (e.g. instalments). Unlike payment integrations, widgets do **not** require a `public_id` or an active order; they can be mounted statically on any page.

## SDK Loading (Widgets Namespace)

Widgets use a dedicated SDK entry point, separate from the payment SDK:

```html
<script src="https://merchant.revolut.com/upsell/bundle.js"></script>
```

## Buy Now Banner

Shows a "Pay in 3 instalments" or similar promotional message on a product page:

```html
<div id="revolut-banner"></div>

<script src="https://merchant.revolut.com/upsell/bundle.js"></script>
<script>
RevolutCheckout.widgets.createBuyNowBanner({
    target:   document.getElementById('revolut-banner'),
    amount:   2999,      // Amount in smallest currency unit
    currency: 'GBP',
    locale:   'en',
});
</script>
```

## Payment Badge

Displays a small payment method badge (e.g. "Pay with Revolut") for trust-building:

```html
<div id="revolut-badge"></div>

<script src="https://merchant.revolut.com/upsell/bundle.js"></script>
<script>
RevolutCheckout.widgets.createPaymentBadge({
    target:   document.getElementById('revolut-badge'),
    currency: 'EUR',
    locale:   'en',
});
</script>
```

## Laravel Blade Integration

### Product Page Banner (Blade View)

```blade
<div class="product-card">
    <h2>{{ $product->name }}</h2>
    <p class="price">£{{ number_format($product->price / 100, 2) }}</p>

    {{-- Revolut promotional banner --}}
    <div id="revolut-promo-{{ $product->id }}"></div>

    <button type="button" id="buy-btn">Add to cart</button>
</div>

<script src="https://merchant.revolut.com/upsell/bundle.js"></script>
<script>
RevolutCheckout.widgets.createBuyNowBanner({
    target:   document.getElementById('revolut-promo-{{ $product->id }}'),
    amount:   {{ $product->price }},
    currency: '{{ $currency ?? 'GBP' }}',
    locale:   '{{ app()->getLocale() }}',
});
</script>
```

### Multiple Products on a Listing Page

```blade
@foreach ($products as $product)
<div class="product-item" data-price="{{ $product->price }}">
    <h3>{{ $product->name }}</h3>
    <div id="banner-{{ $product->id }}"></div>
</div>
@endforeach

<script src="https://merchant.revolut.com/upsell/bundle.js"></script>
<script>
document.querySelectorAll('.product-item').forEach(function (el) {
    var id     = el.querySelector('[id^="banner-"]').id;
    var amount = parseInt(el.dataset.price, 10);

    RevolutCheckout.widgets.createBuyNowBanner({
        target:   document.getElementById(id),
        amount:   amount,
        currency: 'GBP',
        locale:   'en',
    });
});
</script>
```

## Conditional Display

Only show widgets for amounts above the minimum instalment threshold:

```blade
@if ($product->price >= 3000)  {{-- £30.00 minimum --}}
    <div id="revolut-promo"></div>
    <script src="https://merchant.revolut.com/upsell/bundle.js"></script>
    <script>
    RevolutCheckout.widgets.createBuyNowBanner({
        target:   document.getElementById('revolut-promo'),
        amount:   {{ $product->price }},
        currency: 'GBP',
    });
    </script>
@endif
```

## Widgets vs Payment SDK

| | Promotional Widgets | Payment SDK |
|---|---|---|
| Script URL | `upsell/bundle.js` | `embed.js` |
| Requires `public_id` | No | Yes |
| Requires order creation | No | Yes |
| Purpose | Marketing UI | Taking payments |

## Common Pitfalls

- Using `embed.js` instead of `upsell/bundle.js` — widgets are in a different bundle
- Mounting widgets before the DOM element exists — use `DOMContentLoaded`
- Showing banners for amounts below the instalment minimum — the widget renders empty or with a fallback message
- Not localising the widget — pass `locale` to match the customer's language