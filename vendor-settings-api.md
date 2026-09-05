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
  "slug": "my-restaurant",
  "restaurantName": "My Restaurant",
  "legalEntityName": "My GmbH",
  "businessRegistrationNumber": "FN123456a",
  "vatNumber": "ATU12345678",
  "website": "https://myrestaurant.com",
  "country": "Austria",
  "city": "Vienna",
  "postalCode": "1010",
  "address": "Hauptstraße 1",
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
  "acceptPickupCash": true,
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
  "availableLanguages": [
    { "code": "en", "name": "English", "nativeName": "English", "flag": "🇬🇧", "direction": "ltr" },
    { "code": "de", "name": "German", "nativeName": "Deutsch", "flag": "🇩🇪", "direction": "ltr" }
  ],
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
  "postalCode": "8010",
  "address": "Neue Straße 5",
  "phone": "+43 316 123456",
  "description": "Updated description",
  "isLiveAndDiscoverable": true,
  "businessHours": { "monday": { "open": "10:00", "close": "23:00", "isOpen": true } },
  "acceptOnSite": false,
  "acceptPickupCash": false,
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

### Keys That Differ Between Request and Response

Most fields use the same name in both directions. These do not — sending the
response name on update is silently ignored, because unknown keys are dropped
by validation rather than rejected:

| Send on `PUT` | Returned by `GET` |
|---------------|-------------------|
| `totalTablesForReservations` | `totalTables` |
| `logoUrl` | `logo` |
| `coverPhotoUrl` | `coverPhoto` |

`logoUrl`, `coverPhotoUrl`, and `backgroundImageUrl` are normally written by
their dedicated upload endpoints. Send them on `PUT` only as `null`, to clear
an image; passing a full URL back stores that URL verbatim.

### Payment Setting Rules

- `acceptOnSite` enables payment at the restaurant and is the parent setting
  for `acceptPickupCash`.
- `acceptPickupCash` allows pickup and takeaway customers to request cash
  payment at collection.
- Sending `acceptOnSite: false` always stores and returns
  `acceptPickupCash: false`, even if a stale client also sends
  `acceptPickupCash: true`.
- If on-site payments are already disabled, a partial update that attempts to
  enable only `acceptPickupCash` returns `422`.

### Response `422 Unprocessable Content` (pickup-cash dependency)

```json
{
  "message": "Pickup cash can only be enabled when on-site payments are enabled.",
  "errors": {
    "acceptPickupCash": [
      "Pickup cash can only be enabled when on-site payments are enabled."
    ]
  }
}
```

### Local Formatting Rules
- `dateFormat`: `DD.MM.YYYY`, `MM/DD/YYYY`, or `YYYY-MM-DD`.
- `timeFormat`: `24h` or `12h`.
- Vendor-linked customer API dates, timestamps, and opening hours use these saved formats.

### Language Rules
- `dashboardLanguage` controls the authenticated vendor interface.
- `supportedLanguages` controls the language tabs shown to vendors and languages customers may request.
- English is always inserted first into `supportedLanguages`, even if omitted by the client.
- Customer menu requests use `?lang={code}` when explicitly selected, otherwise `Accept-Language`; missing, unsupported, or untranslated languages fall back to English.
- Supported codes are managed by administrators under **Admin → Languages** and returned in `availableLanguages`.
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
| `cover` | file | Required. jpg, jpeg, png, or webp. Max 5 MB. Must be **16:9** aspect ratio, minimum **1200 × 675 px** (recommended **1600 × 900 px**). |

### Response `200 OK`
```json
{
  "coverPhotoUrl": "https://storage.example.com/vendors/1/cover/cover.jpg"
}
```

### Response `422 Unprocessable Content` (invalid dimensions)
Returned when the image is not 16:9 or is smaller than the 1200 × 675 px minimum:
```json
{
  "message": "The cover image must have a 16:9 aspect ratio and be at least 1200×675 px (recommended 1600×900 px).",
  "errors": {
    "cover": [
      "The cover image must have a 16:9 aspect ratio and be at least 1200×675 px (recommended 1600×900 px)."
    ]
  }
}
```

---

## 5. Upload Background Image

**`POST /vendor/{vendorId}/settings/background-image`**

Uploads a menu background image. Previous background image is replaced.

### Path Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID |

### Request
`Content-Type: multipart/form-data`

| Field | Type | Constraints |
|-------|------|-------------|
| `background` | file | Required. jpg, jpeg, png, or webp. Max 5 MB. |

### Response `200 OK`
```json
{
  "backgroundImageUrl": "https://storage.example.com/vendors/1/background/bg.jpg"
}
```

To remove the background image, send `"backgroundImageUrl": null` to
`PUT /vendor/{vendorId}/settings`.

### Error Responses
- `422` — Validation failed (wrong MIME type or size exceeded)

---

## 6. Submit Legal Info for Approval

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
  "postalCode": "1010",
  "address": "Hauptstraße 1",
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
| `postalCode` | Yes, for first submission | Postal code used for the fiskaly managed Unit address |
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

## 7. Get Legal Change Request Status

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
    "postalCode": "1010",
    "address": "Hauptstraße 1"
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

## 7a. Fiscalization — the Registrierkasse

A restaurant in a fiscalized country (`FISKALY_COUNTRIES`, default `AT,DE`) must have a cash register registered before it can go live. **Registration details are part of the vendor's legal identity**: they are submitted and approved together with the rest of it, and adding or changing them always goes back to a Tavlo admin.

- **Austria is the only country that asks the merchant for anything.** Its register is filed through the restaurant's own FinanzOnline account, so the vendor supplies a web-service user (TID, Benutzer-ID, PIN). Those three values can be sent either on `POST /vendor/{id}/legal-info` (Settings, alongside the legal fields) or on `POST /vendor/{id}/fiscal/connect` (activation step two). Both write to the same pending `vendor_request_changes` row. Controlled by `FISKALY_MERCHANT_CREDENTIAL_COUNTRIES`.
- **Every other supported country registers automatically** when a Tavlo admin approves the legal details, so there is no second activation step and nothing is asked of the restaurant. Germany is the first of these.
- **Countries not in `FISKALY_COUNTRIES`** register nothing and are unaffected.

For specialized fiskaly products (`FISKALY_MANAGED_ORGANIZATION_COUNTRIES`, default `AT,DE`), `FISKALY_API_KEY` and `FISKALY_API_SECRET` are the TEST/LIVE credentials of Tavlo's **Group**, not of a restaurant. On first approval Tavlo uses the Management API to:

1. Authenticate the Group key and obtain its Group ID.
2. Create one managed organization (shown as a Unit in fiskaly HUB) for the restaurant, using the approved legal name, VAT number, address, postal code, city, and country.
3. Create a scoped API key under that Unit and store the one-time secret encrypted on the restaurant's `fiscal_devices` row.
4. Use only that Unit key for the restaurant's SIGN AT / SIGN DE resources and receipts.

This isolation prevents one restaurant's SCU/TSS limits and resources from being shared by every Tavlo vendor. Unit discovery uses Tavlo vendor metadata, so retrying after a lost create response reuses the existing Unit instead of creating a duplicate.

Adding a country fiskaly supports is configuration, not a change to this flow: a `services.fiskaly.providers` entry pointing at a `FiscalProvider` class, a base URL, and the country code in `FISKALY_COUNTRIES`. A country listed without a provider fails loudly — admin approval is held with a warning, and its vendors stay blocked from going live rather than silently taking unsigned orders. The manual provisioning command records a failed device for operational retry.

### `GET /vendor/{vendorId}/fiscal/status`

Drives the activation wizard — which steps apply and which are done.

**Auth:** `Bearer {vendorToken}` or an authorized team member token

#### Response `200 OK`
```json
{
  "required": true,
  "country": "AT",
  "needsFinanzOnline": true,
  "connected": false,
  "submitted": false,
  "awaitingApproval": false,
  "needsMerchantAction": true,
  "state": null,
  "serialNumber": null,
  "connectedAt": null,
  "lastError": null,
  "environment": "sandbox",
  "legalInfoSubmitted": true,
  "vatNumber": "ATU12345678",
  "activationComplete": false
}
```

| Field | Meaning |
|---|---|
| `required` | This vendor's country needs a registered cash register. `false` for e.g. `GB`, and while `FISKALY_ENABLED` is off |
| `needsFinanzOnline` | `true` for Austria only — the connect call then requires the three FON values |
| `connected` | A device exists and is `initialized` — receipts are being signed |
| `submitted` | The vendor has supplied their details |
| `awaitingApproval` | Details are on a pending change, waiting on a Tavlo admin |
| `needsMerchantAction` | The vendor still has something to supply — drives whether activation shows a second step |
| `state` | `awaiting_approval` \| `pending` \| `registered` \| `initialized` \| `failed` \| `disabled`, or `null` |
| `lastError` | Populated only while `state` is `failed` |
| `legalInfoSubmitted` | Step one is done (approved on the vendor, or awaiting approval). A rejected submission is `false` so activation returns to the correction form |
| `vatNumber` | Approved value, else the one from the pending legal change |
| `activationComplete` | The vendor has done everything asked of them. Registration may still be waiting on an admin, which is not theirs to chase |

### `POST /vendor/{vendorId}/fiscal/connect`

Adds the FinanzOnline details to the vendor's pending legal change, opening one carrying their current legal values if none is in review. **Nothing is sent to the tax office here** — registration runs on admin approval, against details somebody has checked.

Austria only. Germany and unfiscalized countries return `422`.

Resubmitting before approval updates the same request rather than queuing a second one, so a vendor can correct a wrong PIN.

**Auth:** `Bearer {vendorToken}` or an authorized team member token

#### Request Body
```json
{
  "fonParticipantId": "123456789",
  "fonUserId": "tavlo12345",
  "fonUserPin": "mypin1234"
}
```

These are the credentials of the dedicated **Registrierkassen-Webservice-Benutzer** the restaurant creates in its own FinanzOnline account — never the general FinanzOnline login. The PIN is stored encrypted, never returned by any endpoint, and cleared from the approval history once it has been handed to the device.

| Field | Rules |
|---|---|
| `fonParticipantId` | Required (AT). Teilnehmer-Identifikation, max 100 chars |
| `fonUserId` | Required (AT). 8–12 letters and digits, at least one of each |
| `fonUserPin` | Required (AT). Min 8 chars. FinanzOnline shows it only once, at creation |

#### Response `200 OK`
```json
{
  "submitted": true,
  "connected": false,
  "awaitingApproval": true,
  "state": "awaiting_approval",
  "country": "AT",
  "serialNumber": null,
  "connectedAt": null,
  "changeRequestId": 12,
  "message": "Your details are with our team for review. We will register your cash register once they are approved."
}
```

On approval, the final registration step issues the RKSV **start receipt** (Startbeleg) automatically.

#### Response `422`
```json
{ "message": "Add your legal and tax details before connecting a cash register.", "code": "LEGAL_INFO_REQUIRED" }
```

| `code` | Cause |
|---|---|
| `LEGAL_INFO_REQUIRED` | Legal details have not been submitted |
| `VAT_NUMBER_REQUIRED` | No VAT number on the vendor or the pending legal change |
| `NO_CREDENTIALS_REQUIRED` | This country asks nothing of the merchant (Germany), or is not fiscalized |

Validation errors on the FON fields return the standard `errors` object.

### `POST /vendor/{vendorId}/fiscal/send-instructions`

Emails the FinanzOnline web-service-user steps to the restaurant's accountant.

#### Request Body
```json
{ "email": "accountant@example.com", "name": "Frau Huber", "locale": "de" }
```

`name` is optional. `locale` is `en` (default) or `de`.

#### Response `200 OK`
```json
{ "message": "Instructions sent to accountant@example.com." }
```

### Going live requires a registered cash register

`PUT /vendor/{vendorId}/settings` with `isLiveAndDiscoverable: true` is rejected with `422` and `code: CASH_REGISTER_REQUIRED` unless the vendor's device is `initialized`. The gate also protects existing or operationally provisioned vendors whose device is not ready; without it a restaurant could take orders whose receipts never get signed. Vendors outside `FISKALY_COUNTRIES` are unaffected.

### Admin: approval triggers registration

Approving a vendor's legal change (`POST /admin/vendor/{vendor}/changes/{change}/approve`) writes the approved fields onto the vendor and then registers their cash register, if one has been submitted and is not registered yet.

- Approval and registration are all-or-nothing. If fiskaly rejects vendor-supplied data, none of the legal details are approved; the change becomes `rejected`, its actionable reason is stored in `admin_notes`, and the vendor must correct and resubmit it.
- A Tavlo configuration error or transient provider failure also approves nothing, but leaves the change `pending` so the admin can try approval again after the problem is resolved.
- The managed Unit id, scoped credential, SCU/TSS id, and register/client id are prepared before the approval transaction. If a later registration call fails, those identifiers remain available so a correction or retry resumes the same resources instead of creating duplicates. Legal details themselves remain unapproved.
- `POST /admin/vendor/{vendor}/fiscal/retry` is an operational recovery route for a failed device stranded by manual provisioning. Rejected approval requests are corrected and resubmitted instead.
- Approving a later legal-only change for an already-registered vendor does not re-register it. Changing its fiscal details sends the combined request through approval and registration again.
- Vendors outside `FISKALY_COUNTRIES` register nothing; no provider call is made.
- Austria waits until the vendor has supplied FinanzOnline credentials; approving before that registers nothing and fails nothing. Germany registers on approval alone.
- The FinanzOnline PIN appears in the admin diff as `•••••••• (supplied)`, never in full.

---

## 8. Export Vendor Data

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

## 9. Get Subscription

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

## 10. Stripe Connect — Create Account

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

## 11. Stripe Connect — Get Onboarding Link

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

## 12. Stripe Connect — Get Account Status

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

## 13. Vendor Login and Current User Shape

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
    "slug": "my-restaurant",
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
    "slug": "my-restaurant",
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

## 14. Team Access

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
