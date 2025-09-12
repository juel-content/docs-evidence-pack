# Get Started

**Quickstart**  
Secure card tokenization in 5 minutes

This quickstart will walk you through tokenizing card data and using tokens for payments. By the end, you'll have created tokens, processed payments, and handled webhooks.

---

## Set up your environment

Configure your API credentials:

* **Sandbox environment**
* **Production environment**

**Sandbox (recommended for testing)**

```bash
export TOKEN_API_BASE="https://sandbox.api.bank.example/v1"
export TOKEN_API_KEY="sk_sandbox_123"
```

**Production**

```bash
export TOKEN_API_BASE="https://api.bank.example/v1"
export TOKEN_API_KEY="sk_live_your_key_here"
```

Use test card `4111111111111111` with any future expiry and CVV for sandbox.

---

## Create a **Token**

Transform sensitive card data into a secure token:

1. **Make the request:**

```bash
curl -sS "$TOKEN_API_BASE/tokens" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $TOKEN_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "type": "card",
    "card": {
      "pan": "4111111111111111",
      "exp_month": "12",
      "exp_year": "2030",
      "cardholder_name": "J Doe"
    }
  }'
```

2. **You'll receive a token:**

```json
{
  "id": "tkn_3yKxk1d8Hq3N7V",
  "type": "card",
  "status": "active",
  "last4": "1111",
  "network": "visa"
}
```

3. **Save the token ID** - you'll use this instead of the card number

---

## **Process payments** with your token

1. **Use the token in payment requests:**

```bash
curl -sS "$TOKEN_API_BASE/payments/charges" \
  -H "Authorization: Bearer $TOKEN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 2599,
    "currency": "GBP",
    "source_token": "tkn_3yKxk1d8Hq3N7V"
  }'
```

2. **Payment processes without exposing card data**
3. **Your servers never see the raw PAN** - keeping you out of PCI scope

---

## Handle **Webhooks**

1. **Set up an endpoint** to receive token events
2. **Subscribe to events** like `token.created` and `token.deleted`
3. **Verify signatures** using HMAC-SHA256:

```javascript
function verifyWebhook(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload, 'utf8')
    .digest('hex');
    
  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expectedSignature, 'hex')
  );
}
```

---

## Bonus

Advanced features:

**Reveal masked card data**  
**Delete tokens when done**  
**Set up hosted fields**

---

## Next steps

Explore these guides to learn more:

**API Reference**  
Complete endpoint documentation

**Hosted Fields**  
Client-side tokenization

**Security Guide**  
Production best practices

Learn all **tokenization concepts** and start building!
