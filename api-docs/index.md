# API Documentation Demo – Payments API

**Live version:** View the [interactive Swagger UI](index.html) or [raw OpenAPI spec](openapi.yaml).

This demo shows how I approach technical documentation for APIs.  
It includes an OpenAPI 3.0 spec, a “Getting started” guide, and example requests and responses.  
The goal is to demonstrate clear, developer-focused docs, not to build a live API.  

## Key features
- **Authentication:** OAuth2 with token endpoint  
- **Endpoints:**  
  - `POST /payments` – create a payment  
  - `GET /payments/{id}` – retrieve a payment  
- **Webhooks:** `payment.succeeded` event with payload example  

## Getting started

### Step 1 – Get an access token

Request:
```bash
curl -X POST https://auth.example.com/token \
  -d 'grant_type=client_credentials&client_id=YOUR_CLIENT_ID&client_secret=YOUR_CLIENT_SECRET'
```

Success response:
```json
{
  "access_token": "eyJhbGciOi...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

### Step 2 – Create a payment

Request:
```bash
curl -X POST https://api.example.com/v1/payments \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 42.50,
    "currency": "GBP",
    "reference": "INV-2025-0001"
  }'
```

Success response:
```json
{
  "id": "pay_123abc",
  "status": "created",
  "amount": 42.50,
  "currency": "GBP"
}
```

Error response (invalid request):
```json
{
  "error": "invalid_request",
  "message": "The 'currency' field is required."
}
```

### Step 3 – Retrieve a payment

Request:
```bash
curl -X GET https://api.example.com/v1/payments/pay_123abc \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

Success response:
```json
{
  "id": "pay_123abc",
  "status": "succeeded",
  "amount": 42.50,
  "currency": "GBP"
}
```

Error response (not found):
```json
{
  "error": "not_found",
  "message": "No payment found with ID pay_invalid"
}
```
