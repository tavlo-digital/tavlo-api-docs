# Vendor Settings API Documentation

Base URL: `https://your-domain.com/api`

Authenticated endpoints require `Authorization: Bearer {token}`. Owner/vendor accounts receive full manager access. Staff accounts are stored in `team_members`, use the same `/api/vendor/login` endpoint, and are restricted to their role pages/actions.

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
  "acceptOnSite": true,
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
  "dashboardLanguage": "de",
  "supportedLanguages": ["en", "de"],
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

`currency` is read-only. It is resolved from the vendor's selected `country`
using the matching `countries.currency` value and is not stored in
`vendor_settings`.

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
  "acceptOnSite": false,
  "stripeEnabled": true,
  "serviceFeeRate": 5,
  "autoGenerateReceipts": true,
  "loyaltyEnabled": true,
  "redemptionRate": 0.05,
  "dashboardLanguage": "de",
  "supportedLanguages": ["en", "de", "it"],
  "dateFormat": "DD.MM.YYYY",
  "timeFormat": "24h",
  "showInTopCustomers": true,
  "dataRetentionDays": 365
}
```

### Response `200 OK`
Returns the full settings object (same shape as GET).

The update endpoint does not accept or persist `currency`. Updating map
coordinates (`latitude` and `longitude`) does not change currency.

### Local Formatting Rules
- `dateFormat`: `DD.MM.YYYY`, `MM/DD/YYYY`, or `YYYY-MM-DD`.
- `timeFormat`: `24h` or `12h`.
- Vendor-linked customer API dates, timestamps, and opening hours use these saved formats.

### Language Rules
- `dashboardLanguage` controls the authenticated vendor interface.
- `supportedLanguages` controls the language tabs shown to vendors and languages customers may request.
- English is always inserted first into `supportedLanguages`, even if omitted by the client.
- Customer menu requests use `?lang={code}` when explicitly selected, otherwise `Accept-Language`; missing, unsupported, or untranslated languages fall back to English.
- Supported codes are `en`, `de`, `it`, `fr`, `ar`, `tr`, `zh`, `ja`, `sr`, `cs`, `es`, and `nl`.
- Unsupported language codes return `422`.

### Response `422 Unprocessable Content` (legal guard)
When `isLiveAndDiscoverable` is set to `true` but the vendor has no approved legal information:
```json
{
  "message": "Legal information must be submitted and approved before your restaurant can go live."
}
```

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

Submits proposed changes to legally sensitive fields for admin review. If the vendor already has a pending request, the same request is updated in place instead of creating a duplicate.

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
  "companyType": "GmbH",
  "restaurantName": "My Restaurant",
  "country": "Austria",
  "city": "Vienna",
  "address": "Hauptstraße 1, 1010 Wien",
  "vendorNotes": "Reason: Company renamed"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `legalEntityName` | Yes, for first submission | Legal registered name |
| `businessRegistrationNumber` | Yes, for first submission | Firmenbuchnummer / Handelsregisternummer |
| `vatNumber` | Yes, for first submission | UID / USt-IdNr. |
| `companyType` | Yes, for first submission | Legal company type, such as GmbH or AG |
| `restaurantName` | Yes, for first submission | Public-facing name |
| `country` | Yes, for first submission | Country name |
| `city` | Yes, for first submission | City |
| `address` | Yes, for first submission | Street address |
| `vendorNotes` | No | Notes for the admin reviewer |

When a pending request already exists, all legal fields are optional. Supplied
fields update the pending request and omitted fields retain their pending values.

### Response `201 Created`
Returned when a new pending request is created.

```json
{
  "message": "Legal info submitted for approval."
}
```

### Response `200 OK`
Returned when an existing pending request is updated.

```json
{
  "message": "Pending legal info updated successfully."
}
```

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
  "legalInfo": {
    "restaurantName": "My Restaurant",
    "legalEntityName": "My GmbH",
    "businessRegistrationNumber": "FN123456a",
    "vatNumber": "ATU12345678",
    "companyType": "GmbH",
    "country": "AT",
    "city": "Vienna",
    "address": "Hauptstraße 1, 1010 Wien"
  },
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

## 12. Vendor Login and Current User Shape

### `POST /vendor/login`

**Auth:** Public.

Logs in either the vendor owner/manager or an active invited team member.

### Request Body
```json
{
  "email": "owner@example.com",
  "password": "password123"
}
```

### Response `200 OK` — owner/manager
```json
{
  "user": {
    "id": 1,
    "vendorId": "1",
    "vendorPublicId": "V-ABC12345",
    "vendor_public_id": "V-ABC12345",
    "actorType": "vendor",
    "role": "manager",
    "name": "Owner Name",
    "restaurantName": "My Restaurant",
    "country": "Austria",
    "phone": "+43 1 2345678",
    "email": "owner@example.com",
    "permissions": ["*"],
    "created_at": "2026-05-08T10:00:00.000Z"
  },
  "token": "plain-text-sanctum-token"
}
```

### Response `200 OK` — staff
```json
{
  "user": {
    "id": 12,
    "vendorId": "1",
    "vendorPublicId": "V-ABC12345",
    "vendor_public_id": "V-ABC12345",
    "actorType": "team_member",
    "role": "kitchen",
    "name": "Kitchen Staff",
    "restaurantName": "My Restaurant",
    "country": "Austria",
    "phone": null,
    "email": "kitchen@example.com",
    "permissions": ["orders.view", "orders.manage", "orders.kitchen"],
    "created_at": "2026-05-08T10:00:00.000Z"
  },
  "token": "plain-text-sanctum-token"
}
```

Staff token abilities include `role:team_member` and `role:{role}`. Owner tokens include `role:vendor` and `role:manager`.

### `GET /vendor/me`

**Auth:** Vendor owner or active team member.

Returns the current user under a `data` key using the same shape shown above.

### Response `200 OK`
```json
{
  "data": {
    "actorType": "team_member",
    "role": "waiter",
    "vendorId": "1",
    "email": "waiter@example.com",
    "permissions": ["orders.view", "orders.manage", "tables.close"]
  }
}
```

### `POST /vendor/profile/password`

**Auth:** Vendor owner or active team member.

Changes the authenticated vendor owner or staff user's password. Staff users, including waiters and kitchen users, may only change their own password.

### Request Body
```json
{
  "current_password": "old-password",
  "password": "new-password123",
  "password_confirmation": "new-password123"
}
```

### Response `200 OK`
```json
{
  "message": "Password changed successfully."
}
```

### Validation Errors
- `422` — current password is incorrect, new password is shorter than 8 characters, or confirmation does not match.
- `401` — missing or invalid Bearer token.

---

## 13. Team Access

Team management endpoints are owner/manager-only. Staff users cannot call them. The vendor owner is not stored as a `team_members` record; the frontend displays the owner as **Manager / full access** from the logged-in vendor user.

Allowed invite roles are static: `kitchen` and `waiter`.

### List Team Members

**`GET /vendor/{vendorId}/team`**

**Auth:** Vendor owner/manager only.

### Response `200 OK`
```json
[
  {
    "id": "12",
    "name": "Kitchen Staff",
    "email": "kitchen@example.com",
    "role": "kitchen",
    "permissions": ["orders.view", "orders.manage", "orders.kitchen"],
    "status": "invited",
    "invitedAt": "2026-05-08T10:00:00.000Z",
    "joinedAt": null
  }
]
```

### Invite Team Member

**`POST /vendor/{vendorId}/team/invite`**

**Auth:** Vendor owner/manager only.

Sends an invitation email. The email must not already exist in `vendors.email` or `team_members.email`.

### Request Body
```json
{
  "email": "waiter@example.com",
  "role": "waiter"
}
```

### Response `201 Created`
```json
{
  "id": "13",
  "name": "Waiter",
  "email": "waiter@example.com",
  "role": "waiter",
  "permissions": ["orders.view", "orders.manage", "tables.close"],
  "status": "invited",
  "invitedAt": "2026-05-08T10:00:00.000Z",
  "joinedAt": null
}
```

### Error Responses
- `422` — email belongs to a vendor account: `"This email already belongs to a vendor account."`
- `422` — email already belongs to a team member: `"A team member with this email already exists."`
- `422` — role is not `kitchen` or `waiter`

### Get Invitation

**`GET /vendor/team/invitations/{token}`**

**Auth:** Public.

### Response `200 OK`
```json
{
  "email": "waiter@example.com",
  "role": "waiter",
  "status": "invited",
  "vendorId": "1",
  "vendorName": "My Restaurant"
}
```

### Accept Invitation

**`POST /vendor/team/invitations/{token}/accept`**

**Auth:** Public.

Creates the staff password, activates the member, clears the invitation token, and allows login through `/vendor/login`.

### Request Body
```json
{
  "password": "new-password",
  "password_confirmation": "new-password"
}
```

### Response `200 OK`
```json
{
  "message": "Invitation accepted. You can now sign in.",
  "member": {
    "email": "waiter@example.com",
    "role": "waiter",
    "status": "active",
    "vendorId": "1",
    "vendorName": "My Restaurant"
  }
}
```

### Resend Invitation

**`POST /vendor/{vendorId}/team/{memberId}/resend`**

**Auth:** Vendor owner/manager only.

Regenerates the invitation token and sends a new invitation email. Only `invited` members can be resent.

### Request Body
None.

### Response `200 OK`
Returns the updated team member object.

### Update Team Member

**`PATCH /vendor/{vendorId}/team/{memberId}`**

**Auth:** Vendor owner/manager only.

### Request Body
```json
{
  "name": "Front Waiter",
  "role": "waiter",
  "status": "active"
}
```

Allowed `role`: `kitchen`, `waiter`. Allowed `status`: `invited`, `active`, `suspended`.

### Response `200 OK`
Returns the updated team member object.

### Delete Team Member

**`DELETE /vendor/{vendorId}/team/{memberId}`**

**Auth:** Vendor owner/manager only.

### Response `200 OK`
```json
{
  "message": "Team member removed"
}
```

---

## Stripe Disconnect

**`POST /vendor/{vendorId}/stripe/disconnect`**

Disconnects the vendor's Stripe Connect Express account. Deletes the account on Stripe, removes the account ID from the database, and disables online payments.

### Auth
`Authorization: Bearer {token}` — vendor owner only.

### Path Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID |

### Response `200 OK`
```json
{
  "message": "Stripe account disconnected successfully."
}
```

### Error `422`
```json
{
  "message": "No Stripe account is connected."
}
```

---

## Delete Account

**`POST /vendor/{vendorId}/settings/delete-account`**

Deactivates the vendor account. Revokes all API tokens, sets vendor status to `deactivated`, takes the restaurant offline, and disables discoverability.

### Auth
`Authorization: Bearer {token}` — vendor owner only.

### Path Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID |

### Request Body
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `password` | string | Yes | The vendor's current password for confirmation |

### Response `200 OK`
```json
{
  "message": "Your account has been deactivated successfully."
}
```

### Error `422`
```json
{
  "message": "The password you entered is incorrect."
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
