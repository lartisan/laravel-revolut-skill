# Webhooks

## Setup

Register up to **10 webhook endpoints**.

```http
POST /webhooks
Authorization: Bearer <SECRET_KEY>

{
  "url": "https://yoursite.com/webhooks/revolut",
  "events": [
    "ORDER_COMPLETED",
    "ORDER_AUTHORISED",
    "ORDER_PAYMENT_DECLINED",
    "SUBSCRIPTION_INITIATED",
    "SUBSCRIPTION_CANCELLED",
    "SUBSCRIPTION_OVERDUE"
  ]
}
```

### Manage Webhooks

```http
GET    /webhooks              # list all
GET    /webhooks/{id}         # retrieve one
PUT    /webhooks/{id}         # update URL or events
DELETE /webhooks/{id}         # remove
```

## Event Catalog

### Order Events

| Event | Triggered when |
|---|---|
| `ORDER_COMPLETED` | Payment fully captured and settled |
| `ORDER_AUTHORISED` | Payment authorised but not yet captured |
| `ORDER_PAYMENT_DECLINED` | Payment was declined (hard or soft) |
| `ORDER_CANCELLED` | Order cancelled before capture |

### Subscription Events

| Event | Triggered when |
|---|---|
| `SUBSCRIPTION_INITIATED` | First billing cycle charge processed |
| `SUBSCRIPTION_FINISHED` | Subscription billing period ended normally |
| `SUBSCRIPTION_CANCELLED` | Subscription was cancelled |
| `SUBSCRIPTION_OVERDUE` | Scheduled charge failed; retry pending |

## Webhook Payload Structure

```json
{
  "event": "ORDER_COMPLETED",
  "timestamp": "2026-03-25T10:00:00Z",
  "order_id": "ord_abc123",
  "merchant_order_ext_ref": "your-internal-ref",
  "data": {
    "id": "ord_abc123",
    "state": "completed",
    "amount": 1000,
    "currency": "EUR",
    "created_at": "2026-03-25T09:58:00Z",
    "updated_at": "2026-03-25T10:00:00Z"
  }
}
```

`merchant_order_ext_ref` — your own reference ID if you set `merchant_order_ext_ref` when creating the order. Useful for correlating Revolut orders with your internal order IDs without querying your database.

## Signature Verification

Revolut signs webhook payloads with HMAC-SHA256. Always verify before processing.

The `Revolut-Signature` header contains a timestamp and one or more signatures in the format:
```
t=1711360800,v1=a1b2c3d4...
```

To verify, extract the `v1` signature and compute HMAC-SHA256 over `{timestamp}.{raw_body}`:

```python
import hmac
import hashlib

def verify_webhook(payload_body: bytes, revolut_signature: str, secret: str) -> bool:
    # Parse header: "t=1711360800,v1=abc123..."
    parts = dict(item.split('=', 1) for item in revolut_signature.split(','))
    timestamp = parts.get('t', '')
    signature = parts.get('v1', '')

    # Signed payload is: "{timestamp}.{body}"
    signed_payload = f"{timestamp}.".encode() + payload_body

    expected = hmac.new(
        secret.encode(),
        signed_payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

```python
# Django/Flask example
revolut_signature = request.headers.get('Revolut-Signature')
if not verify_webhook(request.body, revolut_signature, WEBHOOK_SECRET):
    return HttpResponse(status=401)
```

The webhook signing secret is found in the Revolut Business dashboard under Developer → Webhooks.

## Handling Webhooks Reliably

1. **Respond with 200 immediately** — process async in a queue (Celery, BullMQ, etc.)
2. **Idempotency** — store processed `order_id + event` pairs to skip duplicates
3. **Retry logic** — Revolut retries failed webhooks with exponential backoff
4. **Order of events** — do not assume events arrive in order; always fetch the current order state

```python
# Pseudocode: idempotent handler
def handle_webhook(event_type, order_id, data):
    key = f"{event_type}:{order_id}"
    if already_processed(key):
        return  # skip duplicate
    mark_processed(key)

    if event_type == "ORDER_COMPLETED":
        fulfill_order(order_id)
    elif event_type == "SUBSCRIPTION_OVERDUE":
        notify_customer_payment_failed(data)
```

## Testing Webhooks

Use the sandbox environment + a tunnel tool to expose localhost:

```bash
# Using ngrok
ngrok http 3000
# Use the HTTPS ngrok URL as your webhook URL in sandbox
```

Trigger test events from the Revolut Business sandbox dashboard or by completing test payments.
