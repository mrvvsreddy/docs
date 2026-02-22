# Payments API Overview

Maedo uses **Dodo Payments** for all payment processing. The API supports one-time payments, subscriptions, and add-on purchases.

## Base URL

```
https://to.maedo.in/api/v1/payments
```

## Authentication

All payment endpoints except webhooks require authentication via Cognito JWT token passed in the `Authorization` header:

```http
Authorization: Bearer <your_jwt_token>
```

## Available Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/checkout` | POST | ✅ | Create one-time payment |
| `/subscribe` | POST | ✅ | Create subscription |
| `/addon-checkout` | POST | ✅ | Purchase add-ons |
| `/subscription` | GET | ✅ | Get active subscription |
| `/history` | GET | ✅ | Get payment history |
| `/setup` | GET | ✅ | Direct checkout redirect |
| `/webhook` | POST | ❌ | Dodo webhook endpoint |

## Add-ons

Purchase additional resources for your account:

- **Actions**: ₹200 per 200 actions
- **Vector DB Storage**: ₹1,999 per 1 GB
