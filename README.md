# Livra External API Guide (Production)

This document describes external merchant-facing endpoints.

## Outline

- [Shared authentication headers](#shared-authentication-headers)
  - [How to generate `x-signature`](#how-to-generate-x-signature)
- [Location fields (zone, city, state)](#location-fields-zone-city-state)
- [Create Order](#create-order)
- [Update Order](#update-order)
- [Change Request](#change-request)
- [Status](#status)
- [Order status webhooks](#order-status-webhooks)

---

## Shared authentication headers

All routes in this document use the same auth model unless a section says otherwise.

These endpoints require two authentication layers (this is intentional separation of responsibilities):

- `x-api-key` + `x-signature` authenticate the **sending app / delivery platform** (for example: your internal system, or a delivery company that is outsourcing delivery with us).
- `Authorization: Bearer <merchantToken>` authenticates the **merchant** on whose behalf the request is being made.

This means:

- you must have an app-level credential to prove the request really came from an approved integration partner
- you must also provide a merchant-scoped JWT so the request is authorized for the correct merchant

Headers:

- `Content-Type: application/json`
- `x-api-key: <apiKey>`
- `x-signature: <hexHmac>`
- `Authorization: Bearer <merchantToken>`

`x-signature` must be `HMAC-SHA256(rawRequestBody, apiSecret)` encoded as lowercase hex (optionally prefixed with `sha256=`).

### How to generate `x-signature`

Important: the signature must be computed over the **raw HTTP request body bytes** (exact JSON string you send), not a parsed/re-serialized object.

#### JavaScript (Node.js)

```js
import crypto from "node:crypto";

const apiSecret = process.env.LIVRA_API_SECRET; // provided by Livra

// Must be the exact string you send in the HTTP body:
const rawBody = JSON.stringify(payload);

const signature = crypto
  .createHmac("sha256", apiSecret)
  .update(rawBody, "utf8")
  .digest("hex");
// Optional supported format:
// const signature = `sha256=${crypto.createHmac("sha256", apiSecret).update(rawBody, "utf8").digest("hex")}`;
```

#### Python

```python
import hmac
import hashlib

api_secret = "YOUR_LIVRA_API_SECRET"  # provided by Livra

# Must be the exact string you send in the HTTP body:
raw_body = json.dumps(payload, separators=(",", ":"), ensure_ascii=False)

signature = hmac.new(api_secret.encode("utf-8"), raw_body.encode("utf-8"), hashlib.sha256).hexdigest()
# Optional supported format:
# signature = "sha256=" + signature
```

#### PHP

```php
<?php
$apiSecret = getenv("LIVRA_API_SECRET"); // provided by Livra

// Must be the exact string you send in the HTTP body:
$rawBody = json_encode($payload, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);

$signature = hash_hmac("sha256", $rawBody, $apiSecret);
// Optional supported format:
// $signature = "sha256=" . $signature;
```

If you have any issues generating `x-signature`, contact us at `ops@mofavo.com`.

## Location fields (zone, city, state)

Whenever you supply **`zone`**, **`city`**, or **`state`**—on **Create Order**, **Update Order**, or inside an **`ADDRESS_CHANGE`** in **Change Request**—use values from Livra’s Tunisia reference dataset shipped with this documentation ([`tn.json`](./tn.json) in the repository). Each entry has `zone`, `city`, and `admin_name`; send **`admin_name` as `state`**.

## Create Order

Use this when a **new shipment** should be registered with Livra from your app (for example right after checkout). Merchant and partner context come from the bearer token; you supply recipient and parcel details.

- **URL:** `https://external-api.livra.tn/external_create_order`
- **Method:** `POST`

### Request body

```json
{
  "products": "string",
  "productsToRetrieve": "string",
  "name": "string",
  "phone": "string",
  "phone2": "",
  "street": "",
  "zone": "",
  "city": "string",
  "state": "string",
  "zipcode": "",
  "deliveryInstructions": "",
  "amount": 12.5,
  "allowOpen": true,
  "isExchange": false,
  "isFragile": false,
  "callback_link": ""
}
```

### Rules

- `products` is required as a non-empty string.
- `productsToRetrieve` is optional string (used when exchange flow is needed).
- `name`, `phone`, `city`, `state` must be non-empty strings.
- `amount` must be non-negative.
- `allowOpen`, `isExchange`, `isFragile` must be booleans.
- `callback_link` is optional. When set to a URL Livra can reach, we send [Order status webhooks](#order-status-webhooks) to that URL whenever a meaningful change occurs on the order. Omit the field or use an empty string if you do not want callbacks.
- Orders created here are stored with source `external_api`.
- **Location:** See [Location fields (zone, city, state)](#location-fields-zone-city-state).

### Success

- **201**: `{ "orderId": <number> }`

### Errors

- **400** one of:
  - `merchant_not_found`
  - `no_active_contract`
  - `sender_not_found`
  - validation errors
- **401** missing/invalid app auth or bearer token
- **500** internal error

## Update Order

Use this to **correct or adjust an order** that is still **waiting for pickup** at the merchant (patch-style: only send fields that change). Once the order has moved past that stage, use Change Request instead.

- **URL:** `https://external-api.livra.tn/external_update_order`
- **Method:** `POST`

### Request body

Patch-style payload. Only `orderId` is required.

```json
{
  "orderId": 1234,
  "products": "string",
  "productsToRetrieve": "string",
  "name": "string",
  "phone": "string",
  "phone2": "",
  "street": "",
  "zone": "",
  "city": "string",
  "state": "string",
  "zipcode": "",
  "deliveryInstructions": "",
  "amount": 12.5,
  "allowOpen": true,
  "isExchange": false,
  "isFragile": false
}
```

### Constraints

- `orderId` must exist.
- Existing order must be in `readyForPickUp`.
- Missing fields keep existing values.
- If you send `zone`, `city`, or `state`, follow [Location fields (zone, city, state)](#location-fields-zone-city-state).

### Success

- **200**: `{ "orderId": <number> }`

### Errors

- **400** one of:
  - `order_not_found`
  - `order_update_not_permitted`
  - plus create-order style validation errors
- **403** one of:
  - `merchant_token_mismatch`
  - `partner_token_mismatch`
- **401** missing/invalid app auth or bearer token
- **500** internal error

## Change Request

Use this when the parcel is already **in the network** (in a depot or in transit) and you need Livra to apply a **structured change** (phone, amount, address, delivery date, etc.). This creates a pending request for operations to act on.

- **URL:** `https://external-api.livra.tn/external_change_request`
- **Method:** `POST`

### Request body

```json
{
  "orderId": 1234,
  "changes": [
    {
      "type": "PHONE_CHANGE",
      "oldValue": "+971500000000",
      "newValue": "+971511111111"
    }
  ],
  "comment": "Customer requested phone correction",
  "makeRegular": false
}
```

### Rules

- `orderId` must be a positive integer.
- `changes` must be non-empty.
- Allowed `changes[].type`:
  - `PHONE_CHANGE`
  - `PHONE2_CHANGE`
  - `AMOUNT_CHANGE`
  - `ADDRESS_CHANGE`
  - `ALLOW_OPEN_CHANGE`
  - `DELIVERY_DATE_CHANGE`
- For **`ADDRESS_CHANGE`**, any `zone`, `city`, or `state` you include in the old/new address payload must follow [Location fields (zone, city, state)](#location-fields-zone-city-state).
- Order must be in `inDepot` or `inTransit`.
- Exchange-completed orders are blocked.

### Success

- **201**: `{ "ok": true }`

### Errors

- **400** one of:
  - `order_not_found`
  - `order_status_not_eligible_for_change_request`
  - `exchange_already_completed_change_request_not_allowed`
  - `no_changes_provided`
  - `merchant_token_mismatch`
  - `partner_token_mismatch`
  - validation errors
- **401** missing/invalid app auth or bearer token
- **500** internal error

## Status

Use this to **look up many orders at once**—for tracking screens, merchant dashboards, or background sync—and get a compact view of **delivery outcome** vs **where the shipment is now** (depot, leg toward customer or merchant, etc.). [Order status webhooks](#order-status-webhooks) reuse the same **`ok`** and **`orders`** shape as this **200** response, and add a **`timestamp`** field (not returned by Status).

- **URL:** `https://external-api.livra.tn/external_status`
- **Method:** `POST`
- **Auth:** same as other external routes (`x-api-key`, `x-signature`, `Authorization: Bearer`)

### Request body

```json
{
  "orderIds": [1234, 5678]
}
```

### Rules

- `orderIds` is required and must be an array (it may be empty).
- Each element must be a positive integer.
- Only orders that belong to the **merchant** associated with the bearer token are returned. IDs that do not exist or are not visible for that merchant are omitted (no error per id).

### Success

- **200**:

```json
{
  "ok": true,
  "orders": [
    {
      "id": 1234,
      "deliveryStatus": "pending",
      "orderStatus": "inDepot"
    }
  ]
}
```

### What `deliveryStatus` and `orderStatus` mean

Together they answer two different questions: whether the **delivery outcome** for this order is still open, completed, or void, and **where the shipment is** in its journey right now (depot, on the road, and which direction when in transit).

#### `deliveryStatus`

High-level outcome from a delivery perspective:

| Value | Meaning |
| --- | --- |
| `pending` | The order is still in play — not finally delivered to the end recipient in the sense we track here, and not written off as a customer decline / return-to-merchant flow. |
| `delivered` | That outcome is satisfied in our model (for example, the exchange with the customer has already happened and the parcel leg you care about is treated as delivered, even if another leg — such as back to the merchant — is still moving). |
| `cancelled` | The outcome is no longer a normal forward delivery (for example, the customer refused delivery and the parcel is being handled as a return or stop). |

#### `orderStatus`

Where the order sits in the **operational pipeline** — in a depot, on the road, and which direction it is heading when in transit (toward the customer vs toward the merchant):

| Value | Meaning |
| --- | --- |
| `readyForPickUp` | The order has been created and is waiting to be collected by a courier. |
| `inDepot` | The parcel is held at a depot (before first-mile pickup, between legs, or after a customer decline). |
| `inTransitToCustomer` | The parcel is on the road heading toward the customer. |
| `inTransitToMerchant` | The parcel is on the road heading back to the merchant (return or post-exchange leg). |
| `delivered` | The order has been delivered to the customer. |
| `returned` | The order has been returned to the merchant. |
| `exchange-returned` | The exchange was declined by the customer, and the parcel sent for the exchange has returned to the merchant. |
| `exchange-completed` | The exchange happened with the customer, and the parcel collected from the customer has returned to the merchant. |
| `cancelled` | The order has been voided (e.g. a dummy order created as part of an exchange flow). |
| _(other values)_ | New or internal statuses — treat unknown values gracefully. |

#### How the two combine

Examples of valid combinations:

- **Exchange already completed with the customer, parcel now traveling back to the merchant:** `deliveryStatus` can be `delivered` while `orderStatus` reflects the current leg, e.g. `inTransitToMerchant`.
- **Parcel waiting in a depot before final delivery to the customer:** `deliveryStatus` `pending`, `orderStatus` `inDepot`.
- **Customer declined delivery and the parcel is held in a depot:** `deliveryStatus` `cancelled`, `orderStatus` may still be `inDepot` — location/stage in the pipeline can differ even when the delivery outcome is cancelled.

Results are returned in the **same order** as `orderIds`, excluding any ids that were not found for the merchant.

### Errors

- **400** validation errors (e.g. invalid `orderIds` shape)
- **401** missing/invalid app auth or bearer token
- **500** internal error

---

## Order status webhooks

When you include **`callback_link`** on [Create Order](#create-order), Livra calls that URL with an outbound webhook on every meaningful change to the order.

The JSON body matches the [Status](#status) **200** response (`ok`, `orders` with `id`, `deliveryStatus`, `orderStatus`) and includes an extra **`timestamp`** (see below). See [What `deliveryStatus` and `orderStatus` mean](#what-deliverystatus-and-orderstatus-mean) for semantics.

### Request format

```
POST <your-callback-url>
Content-Type: application/json
X-Webhook-ID: <delivery-uuid>
X-Webhook-Signature: <hmac-hex>
```

### Payload

```json
{
  "ok": true,
  "timestamp": "2026-05-05T11:23:00Z",
  "orders": [
    {
      "id": 1234,
      "deliveryStatus": "pending",
      "orderStatus": "inTransitToCustomer"
    }
  ]
}
```

`timestamp` is when the event was detected on the platform, in UTC ISO 8601. On retries the timestamp reflects the **original event time**, not the retry time — use it to understand when something changed, not when you received the notification.

### Verifying signatures

Every request includes an `X-Webhook-Signature` header containing an **HMAC-SHA256** of the **raw request body**, hex-encoded, using your **Livra API secret** (the same secret you use to sign requests to Livra).

Always verify this header before processing the payload.

#### Examples

**Node.js**

```js
const crypto = require('crypto');

function verifySignature(secret, rawBody, signature) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signature)
  );
}

// Express example
app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['x-webhook-signature'];
  if (!verifySignature(process.env.WEBHOOK_SECRET, req.body, sig)) {
    return res.status(401).send('Invalid signature');
  }
  const event = JSON.parse(req.body);
  // process event...
  res.sendStatus(200);
});
```

**Python**

```python
import hmac, hashlib

def verify_signature(secret: str, raw_body: bytes, signature: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        raw_body,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

**Go**

```go
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
)

func verifySignature(secret, signature string, body []byte) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    expected := hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(expected), []byte(signature))
}
```

> **Important:** always read the raw request body for signature verification. Parsing the JSON first and re-serialising it may produce a different byte sequence and cause verification to fail.

### Responding to events

Reply with any **2xx status code** to acknowledge successful delivery. The response body is ignored.

If your endpoint returns a non-2xx status or does not respond within **10 seconds**, the delivery is retried automatically.

### Retry schedule

Failed deliveries are retried with exponential backoff:

| Attempt | Delay before retry |
| --- | --- |
| 1 | 30 seconds |
| 2 | 5 minutes |
| 3 | 30 minutes |
| 4 | 2 hours |
| 5 | 8 hours |

After 5 failed attempts the delivery is marked permanently failed and no further retries are made. The platform team can manually re-queue a delivery on request.

### Identifying deliveries

Each delivery has a unique UUID sent in the `X-Webhook-ID` header. Use this to deduplicate events if your endpoint receives the same delivery more than once.

### Testing your endpoint

A quick way to simulate a webhook delivery locally:

```bash
SECRET="your-secret"
BODY='{"ok":true,"timestamp":"2026-05-05T11:23:00Z","orders":[{"id":1234,"deliveryStatus":"pending","orderStatus":"inTransitToCustomer"}]}'
SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -X POST https://your-endpoint.example.com/webhook \
  -H "Content-Type: application/json" \
  -H "X-Webhook-ID: test-$(uuidgen)" \
  -H "X-Webhook-Signature: $SIG" \
  -d "$BODY"
```
