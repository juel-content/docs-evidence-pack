# Payments API

A simple, developer-friendly API for creating charges, managing customers, and issuing refunds.

- **Base URL (Live):** `https://api.paymentspro.com/v1`
- **Base URL (Sandbox):** `https://sandbox.paymentspro.com/v1`
- **Content type:** `application/json`
- **Auth:** Bearer token (API key)
- **Versioning:** Header `X-API-Version: 2025-09-01` (default if omitted)
- **Idempotency:** Header `Idempotency-Key: <uuid>` on POSTs to avoid duplicate charges
- **Rate limit:** 1000 requests/minute per API key

---

## Quick start

1. **Get an API key** (Dashboard → *Developers* → *API Keys*).
2. **Use the sandbox URL** while building and testing.
3. Call the API with your key in the `Authorization` header:

```bash
curl -X GET "https://sandbox.paymentspro.com/v1/health" \
  -H "Authorization: Bearer sk_test_123"

# Response: 200 OK
{ "status": "ok", "version": "2025-09-01" }
```

## Authentication

Send your API key in the Authorization header:

```
Authorization: Bearer <YOUR_API_KEY>
```

**Key types:**
- `sk_test_*` - Sandbox environment keys
- `sk_live_*` - Live environment keys

**Scopes:** All API keys have the following scopes:
- `payments:read` - View payments and refunds
- `payments:write` - Create payments and refunds
- `customers:read` - View customer data
- `customers:write` - Create and update customers
- `webhooks:read` - View webhook configurations

**Security notes:**
- Never hard-code keys in frontend code or mobile apps
- Rotate leaked keys immediately via the dashboard
- Use environment variables to store keys securely

## Errors

We use standard HTTP status codes. Error responses follow this shape:

```json
{
  "error": {
    "type": "invalid_request_error",
    "code": "amount_too_small",
    "message": "Amount must be at least 50 (minor units)",
    "param": "amount",
    "request_id": "req_9j8k7l"
  }
}
```

| HTTP | Type | Example Code | When it happens |
|------|------|--------------|-----------------|
| 400 | `invalid_request_error` | `missing_param` | Required field missing/invalid |
| 401 | `authentication_error` | `invalid_api_key` | Bad/expired API key |
| 402 | `card_error` | `insufficient_funds` | Payment failed (card/issuer decline) |
| 403 | `permission_error` | `scope_missing` | Key lacks required scope |
| 404 | `resource_missing` | `customer_not_found` | Resource ID doesn't exist |
| 409 | `idempotency_error` | `key_reused` | Reused key with different payload |
| 429 | `rate_limit_error` | `too_many_requests` | Rate limit exceeded |
| 500 | `api_error` | `internal_server_error` | Server error - retry with backoff |

### Retry Logic

For 5xx errors and rate limits (429), implement exponential backoff:
- First retry: 1 second
- Second retry: 2 seconds  
- Third retry: 4 seconds
- Maximum: 3 retries

## Pagination

List endpoints use cursor-based pagination.

**Query params:** `?limit=20&starting_after=cus_123`

**Response format:**
```json
{
  "data": [ /* items */ ],
  "has_more": true,
  "next_cursor": "cus_abc"
}
```

**Limits:** 1-100 items per page (default: 20)

## Webhooks

Receive asynchronous events when important actions occur.

### Event Types

| Event | Description |
|-------|-------------|
| `payment.succeeded` | Payment completed successfully |
| `payment.failed` | Payment attempt failed |
| `payment.requires_action` | Payment needs additional customer action |
| `refund.created` | Refund was initiated |
| `refund.succeeded` | Refund completed successfully |
| `refund.failed` | Refund attempt failed |
| `customer.created` | New customer was created |
| `customer.updated` | Customer information was modified |

### Setup

1. Expose a public endpoint: `POST https://yourapp.com/webhooks/payments`
2. Configure the URL in your dashboard
3. Verify signatures to ensure requests come from us

### Signature Verification

We send JSON payloads with a signature header:
```
X-Signature: t=1736720000,v1=hexhmac
```

Verify by computing an HMAC with your webhook secret:

```javascript
import crypto from "crypto";
import express from "express";

const app = express();
app.use(express.json({ type: "application/json" }));

app.post("/webhooks/payments", (req, res) => {
  const signature = req.header("X-Signature");
  const secret = process.env.WEBHOOK_SECRET; // keep safe
  
  // Get timestamp and signature from header
  const elements = signature.split(",");
  const timestamp = elements[0].split("=")[1];
  const expectedSig = elements[1].split("=")[1];
  
  // Create payload string
  const payload = `${timestamp}.${JSON.stringify(req.body)}`;
  
  // Compute expected signature
  const computed = crypto.createHmac("sha256", secret)
    .update(payload).digest("hex");
  
  if (computed !== expectedSig) {
    return res.status(400).send("Invalid signature");
  }
  
  // Verify timestamp (prevent replay attacks)
  if (Math.abs(Date.now() / 1000 - parseInt(timestamp)) > 300) {
    return res.status(400).send("Request too old");
  }
  
  const event = req.body; // { type: "payment.succeeded", data: {...} }
  
  // Handle event based on type
  switch (event.type) {
    case "payment.succeeded":
      // Update order status, send confirmation email, etc.
      break;
    case "payment.failed":
      // Notify customer, retry payment, etc.
      break;
    // ... handle other events
  }
  
  res.sendStatus(204);
});

app.listen(3000);
```

---

## Endpoints

## Customers

### Create a customer

`POST /customers`

Create a reusable customer object for storing payment methods and billing information.

**Request body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `email` | string | no | Must be valid email format |
| `name` | string | no | Customer's full name |
| `phone` | string | no | Customer's phone number |
| `metadata` | object | no | Free-form key/value pairs (max 50 keys) |

**Response 201 Created:**
```json
{
  "id": "cus_123",
  "email": "dev@example.com",
  "name": "Dev User",
  "phone": "+44 7700 900123",
  "created": 1736720000,
  "updated": 1736720000,
  "metadata": {}
}
```

**Examples:**

```bash
# curl
curl -X POST "https://sandbox.paymentspro.com/v1/customers" \
  -H "Authorization: Bearer sk_test_123" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "dev@example.com",
    "name": "Dev User",
    "metadata": {
      "user_id": "12345",
      "plan": "premium"
    }
  }'
```

### Retrieve a customer

`GET /customers/{id}`

**Response 200 OK:**
```json
{
  "id": "cus_123",
  "email": "dev@example.com",
  "name": "Dev User",
  "phone": "+44 7700 900123",
  "created": 1736720000,
  "updated": 1736720000,
  "metadata": {
    "user_id": "12345",
    "plan": "premium"
  }
}
```

### Update a customer

`PATCH /customers/{id}`

**Request body:** Same fields as create (all optional)

### List customers

`GET /customers`

**Query parameters:**
- `limit` (integer): Number of results (1-100, default: 20)
- `starting_after` (string): Cursor for pagination
- `email` (string): Filter by email address
- `created[gte]` (integer): Filter by creation timestamp

**Response 200 OK:**
```json
{
  "data": [
    {
      "id": "cus_123",
      "email": "dev@example.com",
      "name": "Dev User",
      "created": 1736720000,
      "metadata": {}
    }
  ],
  "has_more": false,
  "next_cursor": null
}
```

### Delete a customer

`DELETE /customers/{id}`

**Response 204 No Content**

---

## Payments

### Create a payment

`POST /payments`

Charge a payment method for the specified amount.

**Request body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `amount` | integer | yes | Amount in minor units (e.g. £12.34 → 1234) |
| `currency` | string | yes | ISO 4217 currency code (GBP, USD, EUR) |
| `source` | object | yes | Payment method details |
| `customer_id` | string | no | Associate with existing customer |
| `description` | string | no | Payment description |
| `metadata` | object | no | Free-form key/value pairs |

**Source object types:**

**Card:**
```json
{
  "type": "card",
  "token": "tok_visa_4242"
}
```

**Bank transfer:**
```json
{
  "type": "bank_transfer",
  "bank_account": {
    "account_number": "12345678",
    "sort_code": "12-34-56",
    "account_holder_name": "John Doe"
  }
}
```

**Digital wallet:**
```json
{
  "type": "apple_pay",
  "token": "tok_apple_abc123"
}
```

**Successful response 201 Created:**
```json
{
  "id": "pay_123",
  "amount": 1234,
  "currency": "GBP",
  "status": "succeeded",
  "source": {
    "type": "card",
    "last4": "4242",
    "brand": "visa",
    "exp_month": 12,
    "exp_year": 2025
  },
  "customer_id": "cus_123",
  "description": "Order #12345",
  "created": 1736720000,
  "metadata": {}
}
```

**Status values:**
- `succeeded` - Payment completed successfully
- `pending` - Payment is being processed
- `requires_action` - Needs customer action (3D Secure, etc.)
- `failed` - Payment failed

### Retrieve a payment

`GET /payments/{id}`

### List payments

`GET /payments`

**Query parameters:**
- `limit`, `starting_after` (pagination)
- `customer_id` (string): Filter by customer
- `status` (string): Filter by status
- `created[gte]`, `created[lte]` (integer): Filter by date range

### Confirm a payment

`POST /payments/{id}/confirm`

Used when a payment has status `requires_action`.

**Request body:**
```json
{
  "return_url": "https://yourapp.com/payment/success"
}
```

---

## Refunds

### Create a refund

`POST /refunds`

Issue a full or partial refund for a successful payment.

**Request body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `payment_id` | string | yes | The payment to refund |
| `amount` | integer | no | Refund amount (defaults to full amount) |
| `reason` | string | no | One of: `duplicate`, `fraudulent`, `requested_by_customer` |
| `metadata` | object | no | Free-form key/value pairs |

**Response 201 Created:**
```json
{
  "id": "rfnd_123",
  "payment_id": "pay_123",
  "amount": 1234,
  "reason": "requested_by_customer",
  "status": "succeeded",
  "created": 1736720000,
  "metadata": {}
}
```

**Status values:**
- `succeeded` - Refund completed successfully
- `pending` - Refund is being processed
- `failed` - Refund failed

### Retrieve a refund

`GET /refunds/{id}`

### List refunds

`GET /refunds`

**Query parameters:**
- `limit`, `starting_after` (pagination)
- `payment_id` (string): Filter by payment
- `status` (string): Filter by status

---

## Common Response Fields

**Timestamps:** All `created` and `updated` fields are Unix timestamps (seconds since epoch).

**IDs:** All resource IDs use prefixes:
- `cus_` for customers
- `pay_` for payments  
- `rfnd_` for refunds
- `tok_` for tokens

**Metadata:** Object with up to 50 key-value pairs. Keys must be strings, values can be strings, numbers, or booleans.

---

## 🔗 Resources & Support

| Resource | Link |
|----------|------|
| ** Changelog** | [docs/CHANGELOG.md](docs/CHANGELOG.md) |
| ** Status Page** | [status.paymentspro.com](https://status.paymentspro.com) |
| ** Support** | support@paymentspro.com |
| ** Dashboard** | [dashboard.paymentspro.com](https://dashboard.paymentspro.com) |

---

## 🤝 Contributing

This is a portfolio piece demonstrating API documentation skills. Feel free to use it as inspiration for your own documentation projects!
