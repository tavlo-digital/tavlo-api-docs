# Vendor Settings API Documentation

Base URL: `https://your-domain.com/api`

All endpoints require authentication via `Authorization: Bearer {token}` with a valid vendor Sanctum token.

---

## 1. Get Vendor Settings

**`GET /vendor/{vendorId}/settings`**

Returns the full vendor settings object for a single vendor.

### Path Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID |

### Response `200 OK`
```json
{
  "id": "1",
  "vendorPublicId": "abc123",
  "restaurantName": "My Restaurant",
  "legalEntityName": "My GmbH",
  "businessRegistrationNumber": "FN123456a",
  "vatNumber": "ATU12345678",
  "website": "https://myrestaurant.com",
  "country": "Austria",
  "city": "Vienna",
  "address": "Hauptstraße 1, 1010 Wien",
  "phone": "+43 1 2345678",
  "email": "vendor@example.com",
  "status": "active",
  "liveStatus": "live",
  "description": "Welcome to our restaurant",
  "logo": "https://storage.example.com/vendors/1/logo/logo.png",
  "coverPhoto": "https://storage.example.com/vendors/1/cover/cover.jpg",
  "backgroundImageUrl": null,
  "isLiveAndDiscoverable": true,
  "businessHours": { "monday": { "open": "09:00", "close": "22:00", "isOpen": true }, "...": "..." },
  "paymentCollectionModel": "on-site",
  "acceptCash": true,
  "acceptCashTakeaway": true,
  "acceptCard": false,
  "acceptApplePay": false,
  "acceptGooglePay": false,
  "acceptVisa": true,
  "acceptMastercard": true,
  "acceptAmex": false,
  "acceptBankTransfer": false,
  "stripeEnabled": false,
  "stripeAccountId": null,
  "stripeOnboardingComplete": false,
  "currency": "EUR",
  "serviceFeeRate": 0,
  "invoicePrefix": "INV",
  "nextInvoiceNumber": 1,
  "autoGenerateReceipts": true,
  "companyType": null,
  "firstInvoiceIssued": false,
  "numberOfTables": 20,
  "tablePrefix": "T",
  "enableSharedBasket": false,
  "maxGuestsPerTable": 8,
  "enableReservations": true,
  "totalTables": 20,
  "maxTableCapacity": 6,
  "autoAcceptOrders": false,
  "estimatedPrepTime": 20,
  "maxOrdersPerSlot": 10,
  "allowGuestOrdering": true,
  "requirePhoneNumber": false,
  "minOrderAmount": 0,
  "maxOrderAmount": null,
  "inventoryTrackingEnabled": true,
  "autoStockDeduction": true,
  "allowNegativeStock": false,
  "autoMarkUnavailableCritical": true,
  "notifyEmailNewOrder": true,
  "notifyEmailReview": false,
  "notifySmsNewOrder": false,
  "notifyPushNewOrder": true,
  "notifyPushOrderReady": false,
  "notificationEmail": "vendor@example.com",
  "enableReviews": true,
  "enableMenuReviews": true,
  "allowAnonymousReviews": false,
  "defaultLanguage": "en",
  "supportedLanguages": ["en"],
  "loyaltyEnabled": false,
  "pointsPerEuro": 10,
  "minimumRedemptionPoints": 100,
  "pointValue": 0.01,
  "redemptionRate": 0.01,
  "pointsExpiryDays": null,
  "showInTopCustomers": false,
  "menuTheme": "default",
  "primaryColor": "#000000",
  "accentColor": "#F97316",
  "menuLayout": "grid",
  "dateFormat": "DD.MM.YYYY",
  "timeFormat": "24h",
  "dataRetentionDays": null
}
```

---

## 2. Update Vendor Settings

**`PUT /vendor/{vendorId}/settings`**

Updates one or more vendor settings fields. Only supplied fields are updated (partial update).

### Path Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID |

### Request Body (all fields optional)
```json
{
  "restaurantName": "New Name",
  "website": "https://newsite.com",
  "city": "Graz",
  "address": "Neue Straße 5",
  "phone": "+43 316 123456",
  "description": "Updated description",
  "isLiveAndDiscoverable": true,
  "businessHours": { "monday": { "open": "10:00", "close": "23:00", "isOpen": true } },
  "paymentCollectionModel": "online",
  "acceptCash": false,
  "acceptCard": true,
  "acceptVisa": true,
  "acceptMastercard": true,
  "acceptAmex": false,
  "acceptBankTransfer": true,
  "stripeEnabled": true,
  "currency": "EUR",
  "serviceFeeRate": 5,
  "autoGenerateReceipts": true,
  "loyaltyEnabled": true,
  "redemptionRate": 0.05,
  "dateFormat": "DD.MM.YYYY",
  "timeFormat": "24h",
  "showInTopCustomers": true,
  "dataRetentionDays": 365
}
```

### Response `200 OK`
Returns the full settings object (same shape as GET).

---

## 3. Upload Logo

**`POST /vendor/{vendorId}/settings/logo`**

Uploads a logo image. Previous logo is replaced.

### Path Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID |

### Request
`Content-Type: multipart/form-data`

| Field | Type | Constraints |
|-------|------|-------------|
| `logo` | file | Required. jpg, jpeg, png, or webp. Max 2 MB. |

### Response `200 OK`
```json
{
  "logoUrl": "https://storage.example.com/vendors/1/logo/logo.png"
}
```

### Error Responses
- `422` — Validation failed (wrong MIME type or size exceeded)

---

## 4. Upload Cover Photo

**`POST /vendor/{vendorId}/settings/cover-photo`**

Uploads a cover photo. Previous cover photo is replaced.

### Path Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID |

### Request
`Content-Type: multipart/form-data`

| Field | Type | Constraints |
|-------|------|-------------|
| `cover` | file | Required. jpg, jpeg, png, or webp. Max 5 MB. |

### Response `200 OK`
```json
{
  "coverPhotoUrl": "https://storage.example.com/vendors/1/cover/cover.jpg"
}
```

---

## 5. Submit Legal Info for Approval

**`POST /vendor/{vendorId}/legal-info`**

Submits proposed changes to legally sensitive fields (legal entity name, registration number, VAT number) for admin review. Only one pending request is allowed at a time.

### Path Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID |

### Request Body
```json
{
  "legalEntityName": "My GmbH",
  "businessRegistrationNumber": "FN123456a",
  "vatNumber": "ATU12345678",
  "restaurantName": "My Restaurant",
  "country": "Austria",
  "city": "Vienna",
  "address": "Hauptstraße 1, 1010 Wien",
  "vendorNotes": "Reason: Company renamed"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `legalEntityName` | Yes | Legal registered name |
| `businessRegistrationNumber` | Yes | Firmenbuchnummer / Handelsregisternummer |
| `vatNumber` | Yes | UID / USt-IdNr. |
| `restaurantName` | No | Public-facing name |
| `country` | No | Country name |
| `city` | No | City |
| `address` | No | Street address |
| `vendorNotes` | No | Notes for the admin reviewer |

### Response `201 Created`
```json
{
  "message": "Legal info submitted for approval."
}
```

### Error Responses
- `422` — A request is already pending: `"A legal info change request is already pending review."`

---

## 6. Get Legal Change Request Status

**`GET /vendor/{vendorId}/legal-info/status`**

Returns the most recent legal change request for this vendor.

### Path Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID |

### Response `200 OK`
```json
{
  "id": 1,
  "status": "pending",
  "adminNotes": null,
  "vendorNotes": "Company renamed",
  "reviewedAt": null,
  "createdAt": "2024-04-05T10:00:00.000Z"
}
```

Status values: `pending` | `approved` | `rejected`

Returns `null` if no requests exist.

---

## 7. Export Vendor Data

**`GET /vendor/{vendorId}/settings/export`**

Returns a JSON export of all vendor settings data for download.

### Path Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID |

### Response `200 OK`
```json
{
  "exportedAt": "2024-04-05T10:00:00.000Z",
  "vendor": { "...full settings object..." }
}
```

Response includes `Content-Disposition: attachment; filename="vendor-{vendorId}-data.json"` header.

---

## 8. Get Subscription

**`GET /vendor/{vendorId}/subscription`**

Returns current subscription plan details.

### Response `200 OK`
```json
{
  "id": "1",
  "planName": "Pro",
  "billingCycle": "monthly",
  "status": "active",
  "currentPeriodStart": "2024-01-01T00:00:00.000Z",
  "currentPeriodEnd": "2024-02-01T00:00:00.000Z",
  "autoRenew": true
}
```

Returns `null` if no active subscription.

---

## 9. Stripe Connect — Create Account

**`POST /vendor/{vendorId}/stripe/connect`**

Creates a Stripe Express account for the vendor. If an account already exists, returns the existing ID.

> **Requires** the `stripe/stripe-php` package: `composer require stripe/stripe-php`
> Set `STRIPE_SECRET` in `.env`.

### Response `200 OK`
```json
{
  "stripeAccountId": "acct_1NwXXXXXXXXXXXXX",
  "alreadyExists": false
}
```

---

## 10. Stripe Connect — Get Onboarding Link

**`POST /vendor/{vendorId}/stripe/onboarding-link`**

Generates a Stripe account onboarding URL to redirect the vendor to complete setup.

### Request Body
```json
{
  "refreshUrl": "https://your-frontend.com/vendor/settings?stripe=refresh",
  "returnUrl": "https://your-frontend.com/vendor/settings?stripe=complete"
}
```

### Response `200 OK`
```json
{
  "onboardingUrl": "https://connect.stripe.com/setup/s/..."
}
```

### Error Responses
- `422` — No Stripe account exists (call `/stripe/connect` first)

---

## 11. Stripe Connect — Get Account Status

**`GET /vendor/{vendorId}/stripe/status`**

Returns the current Stripe Connect account status and syncs `stripe_onboarding_complete` to the database.

### Response `200 OK`
```json
{
  "connected": true,
  "stripeAccountId": "acct_1NwXXXXXXXXXXXXX",
  "onboardingComplete": true,
  "chargesEnabled": true,
  "payoutsEnabled": true
}
```

When not connected:
```json
{
  "connected": false,
  "onboardingComplete": false
}
```

---

## Error Handling

All error responses follow this shape:
```json
{
  "message": "Error description",
  "errors": {
    "fieldName": ["Validation error detail"]
  }
}
```

| HTTP Status | Meaning |
|-------------|---------|
| `401` | Unauthenticated — missing or invalid bearer token |
| `403` | Forbidden — authenticated vendor accessing another vendor's data |
| `404` | Vendor not found |
| `422` | Validation failed |
| `500` | Server error |
