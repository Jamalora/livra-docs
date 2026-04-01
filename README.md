# Mofavo External Orders API

Use this endpoint to create orders in Mofavo.

## Endpoint

`POST https://api.mofavo.com/external`

## Required headers

- `Content-Type: application/json`
- `x-mofavo-api-key: <your app api key>`
- `x-mofavo-signature: <base64 hmac>`
- `Authorization: Bearer <jwt>`

### Signature generation

Compute `x-mofavo-signature` as:

- HMAC SHA256 of the **raw request body** (exact JSON string you send)
- Using your **app secret** as the key
- Base64‑encode the HMAC output

## Request body

```json
{
  "data": {
    "status": "draft",
    "customer": {
      "name": "Jane Doe",
      "phone": "+2348012345678",
      "address": "12 Banana Island Road",
      "city": "Lagos",
      "state": "LA"
    },
    "cart": [
      {
        "name": "Premium T-Shirt",
        "quantity": 2,
        "pricePerUnit": 7500
      }
    ],
    "total": {
      "totalPrice": 15000
    }
  }
}
```

### Status rules

`status` must be one of:

- `draft`
- `readyForPackaging`
- `readyForPickUp`

If `status` is not `draft`, your Bearer token must include `contractId`.

### Status meanings

- `draft`: Order needs confirmation in Mofavo before packaging.
- `readyForPackaging`: Order is confirmed (or merchant skips confirmation) and should be packaged in Mofavo.
- `readyForPickUp`: Order is already packaged (or merchant handles packaging outside Mofavo) and is ready for pickup by the delivery partner.

## Response

Success (201):

```json
{
  "id": 12345
}
```

## Error responses

- `400` invalid payload
- `401` missing/invalid signature or token
- `403` invalid API key

Errors return JSON with a short message, for example:

```json
{
  "error": "Invalid payload",
  "issues": [
    {
      "path": "data.customer.name",
      "message": "Required"
    }
  ]
}
```
