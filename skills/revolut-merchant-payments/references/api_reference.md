# Revolut Merchant API Reference

## Endpoints Summary

### Orders
| Method | Path | Description |
|---|---|---|
| POST | `/orders` | Create a new order |
| GET | `/orders/{id}` | Retrieve an order |
| PATCH | `/orders/{id}` | Update an order |
| POST | `/orders/{id}/payments` | Pay for an order |
| POST | `/orders/{id}/capture` | Capture an authorised order |
| POST | `/orders/{id}/refund` | Refund a completed order |
| POST | `/orders/{id}/cancel` | Cancel an order |

### Customers
| Method | Path | Description |
|---|---|---|
| POST | `/customers` | Create a customer |
| GET | `/customers/{id}` | Retrieve a customer |
| PATCH | `/customers/{id}` | Update a customer |
| DELETE | `/customers/{id}` | Delete a customer |
| GET | `/customers/{id}/payment-methods` | List saved payment methods |
| DELETE | `/customers/{id}/payment-methods/{method_id}` | Delete a saved method |

### Subscriptions
| Method | Path | Description |
|---|---|---|
| POST | `/subscriptions` | Create a subscription |
| GET | `/subscriptions` | List subscriptions |
| GET | `/subscriptions/{id}` | Retrieve a subscription |
| POST | `/subscriptions/{id}/cancel` | Cancel a subscription |

### Webhooks
| Method | Path | Description |
|---|---|---|
| POST | `/webhooks` | Register a webhook |
| GET | `/webhooks` | List webhooks |
| GET | `/webhooks/{id}` | Retrieve a webhook |
| PUT | `/webhooks/{id}` | Update a webhook |
| DELETE | `/webhooks/{id}` | Delete a webhook |

### Terminal / In-Person
| Method | Path | Description |
|---|---|---|
| GET | `/terminals` | List terminal devices |
| POST | `/terminal-payments` | Push payment to terminal |
| GET | `/terminal-payments/{id}` | Retrieve terminal payment |
| GET | `/terminal-payments` | List terminal payments |
| POST | `/terminal-payments/{id}/cancel` | Cancel terminal payment |

### Apple Pay
| Method | Path | Description |
|---|---|---|
| POST | `/apple-pay/merchant-registration` | Register domain for Apple Pay |

## Order Object Fields

```json
{
  "id": "ord_abc123",
  "public_id": "pub_xyz789",
  "type": "payment",
  "state": "completed",
  "checkout_url": "https://checkout.revolut.com/payment/...",
  "amount": 1000,
  "outstanding_amount": 0,
  "currency": "EUR",
  "settled_currency": "EUR",
  "capture_mode": "automatic",
  "customer_email": "user@example.com",
  "customer_id": "cust_123",
  "description": "Order #123",
  "merchant_order_ext_ref": "your-internal-id",
  "created_at": "2026-03-25T09:00:00Z",
  "updated_at": "2026-03-25T09:01:00Z",
  "payments": [...]
}
```

### Order Types
- `payment` — standard customer payment
- `payment_request` — request sent to customer
- `refund`
- `chargeback`
- `chargeback_reversal`
- `credit_reimbursement`

### Capture Modes
- `automatic` — captured immediately on authorisation
- `manual` — remains in `authorised` state until explicitly captured (max 7 days)

## Payment Object States (full list)

`pending` → `authentication_challenge` → `authentication_verified` → `authorisation_started` → `authorisation_passed` → `authorised` → `capture_started` → `captured` → `refund_validated` → `refund_started` → `cancellation_started` → `declining` → `completing` → `cancelling` → `failing` → `completed` → `declined` → `soft_declined` → `cancelled` → `failed`

## Common HTTP Error Codes

| Code | Meaning |
|---|---|
| 400 | Bad request — check request body |
| 401 | Unauthorized — invalid API key |
| 404 | Not found — invalid ID |
| 409 | Conflict — order already in terminal state |
| 422 | Unprocessable — validation error (e.g., >10 webhooks) |
| 429 | Rate limited — back off and retry |
| 500 | Revolut internal error — retry with backoff |

## Rate Limits

Revolut enforces per-merchant rate limits. Use `Retry-After` header when receiving 429 responses. Implement exponential backoff for retries.

## Pagination

List endpoints support cursor-based pagination:

```http
GET /orders?limit=20&starting_after=ord_abc123
```

Parameters:
- `limit` — number of results (default 20, max 100)
- `starting_after` — cursor for next page (last `id` from previous response)
