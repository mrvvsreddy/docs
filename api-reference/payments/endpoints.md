# Payment Endpoints

## POST /checkout

Create a one-time payment checkout session.

**Request:**
```json
{
  "productId": "pdt_0NYKIOqBQRYk3rl0fGXoK",
  "successUrl": "https://app.maedo.in/org/billing?success=true",
  "cancelUrl": "https://app.maedo.in/org/billing?cancelled=true"
}
```

**Response:**
```json
{
  "checkoutUrl": "https://checkout.dodopayments.com/session_abc123",
  "paymentId": "session_abc123"
}
```

---

## POST /subscribe

Create a subscription checkout session.

**Request:**
```json
{
  "productId": "pdt_0NWotHy5pkU3opL66knuJ",
  "successUrl": "https://app.maedo.in/org/billing?success=true",
  "cancelUrl": "https://app.maedo.in/org/billing?cancelled=true"
}
```

**Response:**
```json
{
  "checkoutUrl": "https://checkout.dodopayments.com/session_xyz789",
  "subscriptionId": "session_xyz789"
}
```

---

## GET /subscription

Get the user's active subscription.

**Response:**
```json
{
  "subscription": {
    "id": "sub_123",
    "status": "active",
    "current_period_end": "2026-03-20T00:00:00Z"
  },
  "hasActiveSubscription": true
}
```

---

## GET /history

Get payment history with pagination.

**Query Parameters:**
- `limit` (optional): Number of results (default: 50, max: 100)
- `offset` (optional): Offset for pagination (default: 0)

**Response:**
```json
{
  "payments": [
    {
      "id": "pay_123",
      "provider_payment_id": "session_abc123",
      "amount": 200,
      "currency": "INR",
      "status": "succeeded",
      "metadata": { "addon_type": "actions" },
      "created_at": "2026-02-20T10:30:00Z"
    }
  ],
  "total": 1,
  "limit": 50,
  "offset": 0
}
```

---

## GET /setup

Redirect handler for post-signup plan purchase.

**Query Parameters:**
- `plan`: Plan name (e.g., `growth`)
- `redirect_to`: URL to redirect after checkout

**Response:** 302 redirect to Dodo checkout

---

## POST /webhook

Dodo Payments webhook endpoint. No authentication required (verified via signature).

**Supported Events:**
- `payment.succeeded`
- `payment.failed` / `payment.cancelled`
- `payment.processing`
- `subscription.active` / `subscription.cancelled` / `subscription.expired`
