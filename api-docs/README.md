# API Documentation Demo – Payments API

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
```bash
curl -X POST https://auth.example.com/token \
  -d 'grant_type=client_credentials&client_id=YOUR_CLIENT_ID&client_secret=YOUR_CLIENT_SECRET'
```

### Step 2 - Create a payment

curl -X POST https://api.example.com/v1/payments \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 42.50,
    "currency": "GBP",
    "reference": "INV-2025-0001"
  }'
```

