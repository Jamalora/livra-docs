# Livra External API Guide (Production)

This document describes external merchant-facing endpoints.

## Outline

- [Shared authentication headers](#shared-authentication-headers)
  - [How to generate `x-signature`](#how-to-generate-x-signature)
- [Create Order](#create-order)
- [Update Order](#update-order)
- [Change Request](#change-request)
- [Status](#status)
  - [What `deliveryStatus` and `orderStatus` mean](#what-deliverystatus-and-orderstatus-mean)

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
  "isFragile": false
}
```

### Rules

- `products` is required as a non-empty string.
- `productsToRetrieve` is optional string (used when exchange flow is needed).
- `name`, `phone`, `city`, `state` must be non-empty strings.
- `amount` must be non-negative.
- `allowOpen`, `isExchange`, `isFragile` must be booleans.
- Orders created here are stored with source `external_api`.

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

Use this to **look up many orders at once**—for tracking screens, merchant dashboards, or background sync—and get a compact view of **delivery outcome** vs **where the shipment is now** (depot, leg toward customer or merchant, etc.).

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

Together they answer two different questions: **whether the “delivery outcome” for this order is still open, completed, or void**, and **where the shipment is in its journey right now**.

**`deliveryStatus`** is the high-level outcome from a delivery perspective:

- **`pending`** — the order is still in play (not finally delivered to the end recipient in the sense we track here, and not written off as a customer decline / return-to-merchant flow).
- **`delivered`** — that outcome is satisfied in our model (for example, the exchange with the customer has already happened and the parcel leg you care about is treated as delivered, even if another leg—such as back to the merchant—is still moving).
- **`cancelled`** — the outcome is no longer a normal forward delivery (for example, the customer refused delivery and the parcel is being handled as a return or stop).

**`orderStatus`** describes **where the order sits in the operational pipeline**—in a depot, on the road, and **which direction** it is heading when it is in transit (toward the customer vs toward the merchant). So you can have combinations such as:

- Exchange already completed with the customer, parcel **now traveling back to the merchant**: `deliveryStatus` can be **`delivered`** while `orderStatus` reflects the current leg, e.g. **`inTransitToMerchant`**.
- Parcel **waiting in a depot** before final delivery to the customer: `deliveryStatus` **`pending`**, `orderStatus` **`inDepot`**.
- Customer **declined** delivery and the parcel is **held in a depot**: `deliveryStatus` **`cancelled`**, `orderStatus` still **`inDepot`** (location/stage) even though the delivery outcome is cancelled.

Results are returned in the **same order** as `orderIds`, excluding any ids that were not found for the merchant.

### Errors

- **400** validation errors (e.g. invalid `orderIds` shape)
- **401** missing/invalid app auth or bearer token
- **500** internal error
