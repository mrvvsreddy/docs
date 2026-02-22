# Add-on Purchases

Purchase additional resources for your account via the `/addon-checkout` endpoint.

## Available Add-ons

| Add-on | ID | Price | Unit |
|--------|----|-------|------|
| Actions | `actions` | ₹200 | 200 actions |
| Vector DB Storage | `vector-db` | ₹1,999 | 1 GB |

---

## POST /addon-checkout

Create a checkout session for add-on purchases.

**Request:**
```json
{
  "addonId": "actions",
  "quantity": 3,
  "redirect_to": "https://app.maedo.in/org/billing"
}
```

**Response:**
```json
{
  "payment_link": "https://checkout.dodopayments.com/session_addon123",
  "paymentId": "session_addon123"
}
```

**Notes:**
- Quantity must be between 1 and 100
- Only one add-on type per checkout (Dodo limitation)
- If multiple add-ons are selected, the first one is processed

---

## Provisioning

After successful payment, resources are automatically provisioned:

- **Actions**: Added to your action ledger immediately
- **Vector DB Storage**: Added to your storage quota

The webhook handler verifies the payment amount matches the expected price before provisioning.
