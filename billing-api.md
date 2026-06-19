# Billing & Subscription API

All endpoints require authentication via a Sanctum Bearer token issued by the `auth:vendor` guard.

**Base URL prefix:** `/api/vendor/{vendorId}/billing`

`vendorId` accepts either the `vendor_public_id` (e.g. `V-ABC12345`) or the numeric primary key.

---

## Table of Contents

1. [Get Subscription Plans](#1-get-subscription-plans)
2. [Create Checkout Session](#2-create-checkout-session)
3. [Get Billing Details](#3-get-billing-details)
4. [List Invoices](#4-list-invoices)
5. [Download Invoice](#5-download-invoice)
6. [Get Usage Stats](#6-get-usage-stats)
7. [Upgrade Plan](#7-upgrade-plan)
8. [Change Billing Cycle](#8-change-billing-cycle)
9. [Update Payment Method](#9-update-payment-method)
10. [Cancel Subscription](#10-cancel-subscription)
11. [Get Billing Portal URL](#11-get-billing-portal-url)

---

## 1. Get Subscription Plans

Return all active subscription plans with their features. No `vendorId` required — plans are global.

**Request**

```
GET /api/vendor/billing/plans
Authorization: Bearer {token}
```

**Response 200 OK**

```json
{
  "data": [
    {
      "id": 1,
      "name": "Basic",
      "description": "For small restaurants",
      "monthlyPrice": 99.00,
      "yearlyPrice": 950.00,
      "currency": "EUR",
      "isPopular": false,
      "maxUsers": 3,
      "features": [
        {
          "name": "QR code ordering",
          "description": "Customers scan & order",
          "category": "ordering",
          "isInherited": false
        }
      ]
    }
  ]
}
```

**Errors**

| Status | Condition                |
|--------|--------------------------|
| 401    | Missing or invalid token |

---

## 2. Create Checkout Session

Create a Stripe Checkout session to start a subscription payment. The API first
creates a local subscription with `status = pending`, then returns the URL the
frontend should use to redirect the vendor to Stripe.

**Request**

```
POST /api/vendor/{vendorId}/billing/checkout-session
Authorization: Bearer {token}
Content-Type: application/json

{
  "planId": 2,
  "billingCycle": "monthly"
}
```

**Body Parameters**

| Name           | Type    | Required | Description                              |
|----------------|---------|----------|------------------------------------------|
| `planId`       | integer | yes      | ID of the target `subscription_plans` row |
| `billingCycle` | string  | yes      | `monthly` or `yearly`                     |

**Response 200 OK**

```json
{
  "checkoutUrl": "https://checkout.stripe.com/c/pay/cs_live_...",
  "subscriptionId": "42"
}
```

The pending row is linked to the Stripe Checkout session through
`stripe_checkout_session_id`. A successful webhook or the authenticated
`verify-checkout` fallback activates that same row. The scheduled
`subscriptions:reconcile-stale` command also polls pending Checkout sessions
older than 10 minutes, so a missed webhook does not leave a paid subscription
unrecoverable.

**Errors**

| Status | Condition                                                    |
|--------|--------------------------------------------------------------|
| 401    | Missing or invalid token                                     |
| 403    | Token belongs to another vendor                              |
| 422    | Validation failure, or Stripe price not configured for plan  |
| 502    | Stripe Checkout session creation failed                      |
| 503    | Stripe is not configured on the server                       |

---

## 3. Get Billing Details

Retrieve full subscription details including plan, billing cycle, payment method metadata, and next billing date.

**Request**

```
GET /api/vendor/{vendorId}/billing
Authorization: Bearer {token}
```

**Response 200 OK**

```json
{
  "subscriptionId": 1,
  "planId": 2,
  "planName": "Professional",
  "billingCycle": "monthly",
  "status": "active",
  "price": 49.00,
  "currency": "EUR",
  "maxUsers": 10,
  "interval": "month",
  "nextBillingDate": "2026-05-05",
  "autoRenew": true,
  "stripeSubscriptionId": null,
  "paymentMethod": {
    "brand": "visa",
    "last4": "4242",
    "expMonth": "12",
    "expYear": "2028",
    "isDefault": true
  },
  "billingEmail": "billing@example.com",
  "paymentMethodUpdated": "2026-04-05T10:00:00"
}
```

> `paymentMethod` and `billingEmail` are `null` when no payment method is stored.

**Errors**

| Status | Condition                                          |
|--------|----------------------------------------------------|
| 401    | Missing or invalid token                           |
| 403    | Token belongs to a different vendor                |
| 404    | Vendor not found                                   |

---

## 4. List Invoices

Return a paginated list of invoices for the vendor's active subscription.

**Request**

```
GET /api/vendor/{vendorId}/billing/invoices
Authorization: Bearer {token}
```

**Query Parameters**

| Name    | Type    | Default | Description              |
|---------|---------|---------|--------------------------|
| page    | integer | 1       | Page number              |

**Response 200 OK**

```json
{
  "data": [
    {
      "id": 12,
      "amount": 49.00,
      "status": "paid",
      "issue_date": "2026-03-05",
      "due_date": "2026-03-10",
      "pdf_url": "https://...",
      "stripe_hosted_url": "https://...",
      "vat": "9.31"
    }
  ],
  "total": 1,
  "perPage": 10,
  "currentPage": 1,
  "lastPage": 1
}
```

> Returns `{ "data": [], "total": 0, "perPage": 10, "currentPage": 1, "lastPage": 1 }` when no subscription exists.

**Errors**

| Status | Condition                       |
|--------|---------------------------------|
| 401    | Missing or invalid token        |
| 403    | Token belongs to another vendor |

---

## 5. Download Invoice

Retrieve the URL to a PDF or Stripe-hosted invoice page.

**Request**

```
GET /api/vendor/{vendorId}/billing/invoices/{invoiceId}/download
Authorization: Bearer {token}
```

**Response 200 OK**

```json
{
  "url": "https://stripe.com/invoice/abc123"
}
```

**Errors**

| Status | Condition                                              |
|--------|--------------------------------------------------------|
| 401    | Missing or invalid token                               |
| 403    | Token belongs to another vendor                        |
| 404    | Invoice not found, or PDF/hosted URL not yet available |

---

## 6. Get Usage Stats

Return current-period usage counters for the vendor's account.

**Request**

```
GET /api/vendor/{vendorId}/billing/usage
Authorization: Bearer {token}
```

**Response 200 OK**

```json
{
  "activeTables": 8,
  "ordersThisMonth": 142,
  "staffAccounts": 4,
  "qrCodes": 9
}
```

| Field            | Description                                                   |
|------------------|---------------------------------------------------------------|
| `activeTables`   | `restaurant_tables` where `is_active = true`                  |
| `ordersThisMonth`| Orders placed in the current calendar month                   |
| `staffAccounts`  | `team_members` where `status = active`                        |
| `qrCodes`        | Active tables + 1 if a takeaway QR exists                     |

**Errors**

| Status | Condition                       |
|--------|---------------------------------|
| 401    | Missing or invalid token        |
| 403    | Token belongs to another vendor |

---

## 7. Upgrade Plan

Switch the vendor's subscription to a different plan. Records a `plan_changed` event.

**Request**

```
POST /api/vendor/{vendorId}/billing/upgrade
Authorization: Bearer {token}
Content-Type: application/json

{
  "planId": 3
}
```

**Body Parameters**

| Name     | Type    | Required | Description                               |
|----------|---------|----------|-------------------------------------------|
| `planId` | integer | ✓        | ID of the target `subscription_plans` row |

**Response 200 OK**

```json
{
  "message": "Plan upgraded successfully.",
  "subscription": { /* same shape as Get Billing Details */ }
}
```

**Errors**

| Status | Condition                                    |
|--------|----------------------------------------------|
| 401    | Missing or invalid token                     |
| 403    | Token belongs to another vendor              |
| 422    | `planId` is missing, not an integer, does not exist in `subscription_plans`, or is already the active plan |
| 404    | Plan not found (race condition)              |

---

## 8. Change Billing Cycle

Switch between `monthly` and `yearly` billing. Recalculates `next_billing_date` and records a `cycle_changed` event.

**Request**

```
PATCH /api/vendor/{vendorId}/billing/cycle
Authorization: Bearer {token}
Content-Type: application/json

{
  "cycle": "yearly"
}
```

**Body Parameters**

| Name    | Type   | Required | Allowed values         |
|---------|--------|----------|------------------------|
| `cycle` | string | ✓        | `monthly` \| `yearly`  |

**Response 200 OK**

```json
{
  "message": "Billing cycle updated successfully.",
  "subscription": { /* same shape as Get Billing Details */ }
}
```

**Errors**

| Status | Condition                                             |
|--------|-------------------------------------------------------|
| 401    | Missing or invalid token                              |
| 403    | Token belongs to another vendor                       |
| 422    | `cycle` value is invalid, already matches current cycle, or no active subscription exists |
| 404    | No subscription found (race condition)                |

---

## 9. Update Payment Method

Store or replace the vendor's default payment method. The previous default (if any) is demoted before the new record is saved. Uses `stripe_payment_method_id` as the upsert key.

**Request**

```
POST /api/vendor/{vendorId}/billing/payment-method
Authorization: Bearer {token}
Content-Type: application/json

{
  "stripePaymentMethodId": "pm_1OEbVm2eZvKYlo2C5nJBW1dv",
  "cardBrand": "visa",
  "last4": "4242",
  "expMonth": "12",
  "expYear": "2028",
  "billingEmail": "billing@example.com"
}
```

**Body Parameters**

| Name                    | Type   | Required | Description                       |
|-------------------------|--------|----------|-----------------------------------|
| `stripePaymentMethodId` | string | ✓        | Stripe `pm_*` identifier          |
| `cardBrand`             | string |          | e.g. `visa`, `mastercard`         |
| `last4`                 | string |          | Last 4 digits (exactly 4 chars)   |
| `expMonth`              | string |          | Expiry month (`01`–`12`)          |
| `expYear`               | string |          | 4-digit expiry year               |
| `billingEmail`          | string |          | Email tied to billing             |

**Response 200 OK**

```json
{
  "message": "Payment method updated successfully.",
  "paymentMethod": {
    "brand": "visa",
    "last4": "4242",
    "expMonth": "12",
    "expYear": "2028",
    "isDefault": true
  }
}
```

**Errors**

| Status | Condition                                          |
|--------|----------------------------------------------------|
| 401    | Missing or invalid token                           |
| 403    | Token belongs to another vendor                    |
| 422    | Validation failure (missing `stripePaymentMethodId`, invalid email, etc.) |

---

## 10. Cancel Subscription

Mark the vendor's subscription as cancelled; sets `status = cancelled`, `cancelled_at = now()`, `auto_renew = false`. Records a `subscription_cancelled` event.

**Request**

```
POST /api/vendor/{vendorId}/billing/cancel
Authorization: Bearer {token}
```

No request body required.

**Response 200 OK**

```json
{
  "message": "Subscription cancelled successfully.",
  "subscription": { /* same shape as Get Billing Details */ }
}
```

**Errors**

| Status | Condition                                               |
|--------|---------------------------------------------------------|
| 401    | Missing or invalid token                                |
| 403    | Token belongs to another vendor                         |
| 422    | Subscription is already cancelled                       |
| 404    | No subscription found                                   |

---

## 11. Get Billing Portal URL

Generate a Stripe Customer Portal session URL. Returns `null` for the URL when the Stripe integration is not configured.

**Request**

```
POST /api/vendor/{vendorId}/billing/portal
Authorization: Bearer {token}
```

No request body required.

**Response 200 OK (Stripe configured)**

```json
{
  "url": "https://billing.stripe.com/session/abc123"
}
```

**Response 200 OK (Stripe not configured)**

```json
{
  "url": null,
  "message": "Billing portal is not yet configured. Please contact support to update your payment method."
}
```

**Errors**

| Status | Condition                       |
|--------|---------------------------------|
| 401    | Missing or invalid token        |
| 403    | Token belongs to another vendor |

---

## Common Error Shapes

**401 Unauthenticated**
```json
{ "message": "Unauthenticated." }
```

**403 Forbidden**
```json
{ "message": "Forbidden." }
```

**404 Not Found**
```json
{ "message": "No query results for model [App\\Models\\Vendor]." }
```

**422 Unprocessable Content** (validation)
```json
{
  "message": "The plan id field is required.",
  "errors": {
    "planId": ["The plan id field is required."]
  }
}
```

**422 Unprocessable Content** (business rule)
```json
{ "message": "The selected plan is already active." }
```

---

## Database Schema Reference

| Table               | Key columns added                                                              |
|---------------------|--------------------------------------------------------------------------------|
| `subscriptions`     | `stripe_checkout_session_id`, `stripe_customer_id`, `cancelled_at`, `paused_at` |
| `invoices`          | `vat`, `pdf_url`, `stripe_invoice_id`, `stripe_hosted_url`                    |
| `payment_methods`   | New table: `vendor_id`, `card_brand`, `last4`, `exp_month`, `exp_year`, `stripe_payment_method_id`, `is_default`, `billing_email` |
| `subscription_events` | `event_type` values: `plan_changed`, `cycle_changed`, `subscription_cancelled` |
