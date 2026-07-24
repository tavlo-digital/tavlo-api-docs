```

```

# Customer API Documentation

## Base URL

```
/api/customer
```

## Authentication

All authenticated endpoints require a Bearer token via Laravel Sanctum.

```
Authorization: Bearer {token}
```

## Redis Performance Layer

When enabled, customer reads use a versioned Redis response cache. Cache keys
include route, query string, locale and authenticated customer ID. Successful
customer/vendor/admin mutations, Stripe webhooks and completed background
commands advance the cache version. Redis failures fall back to the normal
database response path.

Cart item add/update/remove and order item share/unshare can use the ordered
Redis command path. These endpoints return `202` after enqueueing, while the
existing Pusher notification delivers the authoritative cart or `state_patch`.
Commands for one table session are sequenced and processed under a session
lock. Order draft/confirm, pay-for/release and intent creation wait briefly for
that session's earlier commands, preventing a fast navigation from overtaking a
cart change.

The async command path is only used when a queue worker is actually consuming
the command queue. A running worker refreshes a heartbeat on every poll; if the
newest heartbeat is stale (no worker draining the queue), these endpoints
automatically fall back to synchronous processing and return their normal
`201`/`200` response instead of `202`. This means a stopped or crashed worker
never causes cart writes to be silently lost. The staleness threshold is
`CUSTOMER_COMMAND_WORKER_MAX_AGE` seconds (default `15`; set to `0` to disable
the guard and always queue when the async system is enabled).

Payment intent creation/update/cancel/verify, cash payment requests, order
confirmation and coverage assignment remain authoritative synchronous database
operations. They are not acknowledged before their database transaction (and,
where applicable, Stripe operation) succeeds.

Relevant environment variables:

```dotenv
CUSTOMER_API_CACHE_ENABLED=true
CUSTOMER_API_CACHE_STORE=redis
CUSTOMER_API_CACHE_TTL=120
CUSTOMER_ASYNC_COMMANDS_ENABLED=true
CUSTOMER_COMMANDS_CONNECTION=redis
CUSTOMER_COMMANDS_QUEUE=realtime
CUSTOMER_COMMAND_STATUS_TTL=3600
CUSTOMER_COMMAND_BARRIER_TIMEOUT_MS=2000
```

## Media URLs

All media fields (`logo_url`, `cover_photo_url`, `image_url`, `profile_picture`, review `images`, etc.) are returned as **absolute URLs** pointing at the backend, e.g. `http://localhost:8000/media/vendors/1/logo/abc.png`. Files are publicly accessible — no signed token or auth header is required to load them, so the frontend can use the URL directly in `<img>` tags.

## Pricing — VAT-Inclusive (Gross)

**All customer-facing prices are VAT-inclusive (gross).** The database stores net prices; the API computes gross prices at response time:

```
gross = net × (1 + vat_rate / 100)
```

This applies to: `price`, `discounted_price`, `unit_price`, `line_total`, `my_share`, paid addon `price`, and modifier option `price_adjustment`.

VAT rates are resolved from the `tax_categories` table by the vendor's country and the item's `tax_category`. Default rates for Austria: `food` = 10%, `beverage_non_alcoholic` = 20%, `beverage_alcoholic` = 20%.

### `tax_groups` Array

Order-level responses include a `tax_groups` array that groups all items by tax category. For customer-facing endpoints, each item's contribution is divided by its share count (`1 + count(shared_order_ids)`), so tax_groups reflect per-share amounts — not full item prices. Vendor endpoints use full item prices.

| Field          | Type   | Description                                  |
| -------------- | ------ | -------------------------------------------- |
| `code`         | string | Letter code (A, B, C...) for receipt display |
| `label`        | string | Human-readable label, e.g. "Food (10%)"      |
| `tax_category` | string | Tax category slug                            |
| `vat_rate`     | float  | VAT percentage                               |
| `vat_amount`   | float  | Total VAT for this group                     |
| `gross_amount` | float  | Total gross (VAT-inclusive) for this group   |
| `net_amount`   | float  | Total net (VAT-exclusive) for this group     |

### `totals` Object

| Field         | Type   | Description                                                                                                                                           |
| ------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `net_total`   | float  | Sum of all`net_amount` across tax groups                                                                                                              |
| `vat_total`   | float  | Sum of all`vat_amount` across tax groups                                                                                                              |
| `service_fee` | float  | `gross_total × service_fee_rate%` (from vendor settings)                                                                                              |
| `total_tips`  | float  | Sum of`tip_amount` from the order(s). In table history: sum across all orders for one person. In order detail/history/receipt: the single order's tip |
| `grand_total` | float  | `gross_total + service_fee + total_tips`                                                                                                              |
| `currency`    | string | Only present in order detail responses                                                                                                                |

### Currency Resolution

- Restaurant and table-scan `currency` values come from the restaurant's selected `vendors.country`, matched to `countries.currency`.
- Map latitude and longitude do not determine or update currency.
- Order, payment, refund, and receipt currency values are snapshots stored on the transaction so historical records do not change when a restaurant changes country.

## Vendor Date and Time Formatting

Customer responses connected to a restaurant use that vendor's saved `dateFormat` and `timeFormat` from **Vendor Settings → Language → Local Formatting**.

| Saved setting        | Response example     |
| -------------------- | -------------------- |
| `DD.MM.YYYY` + `24h` | `18.04.2026 14:32`   |
| `MM/DD/YYYY` + `12h` | `04/18/2026 2:32 PM` |
| `YYYY-MM-DD` + `24h` | `2026-04-18 14:32`   |

This applies to restaurant opening hours, reviews, table sessions, table/order history, receipts, order tracking, reservations, customer reviews, notifications, and loyalty transactions. In responses containing data from several restaurants, each row uses its own vendor's settings.

Request values remain canonical for validation and storage. For example, reservation requests still send `date` as `YYYY-MM-DD` and `time` as `HH:mm`; only the returned values use the vendor-selected display format.

Vendor-independent technical or customer-owned values do not use restaurant settings:

- `/health.timestamp` remains ISO8601 because no vendor is associated with the request.
- `date_of_birth` remains `YYYY-MM-DD`.
- GDPR request timestamps remain standard API timestamps because they belong to the customer account, not a restaurant.

## Customer Language for Cart, Orders, and Notifications

Cart, table/order history, order detail, tracking, receipt item rows, and notifications resolve customer-facing menu/customization names from the `Accept-Language` header. These endpoints do **not** use `?lang=`.

Clients should prefer ID-based cart selections:

```json
{
    "menu_item_id": 42,
    "paid_addons": [{ "id": 5 }],
    "free_addons": [{ "id": 8 }],
    "removed_items": [{ "id": 11 }],
    "selected_modifiers": [{ "modifier_group_id": 1, "option_ids": [2] }]
}
```

The cart stores IDs plus price/tax snapshots where needed. Display names are resolved dynamically from menu translations for the requested language. Legacy name-based add-on/removable submissions are still accepted.

---

## 0. Health & Diagnostics (Public)

### 0.1 Ping

**GET** `/api/customer/ping`

Simple reachability check.

**Response (200):**

```json
{
    "message": "pong"
}
```

---

### 0.2 Health Check

**GET** `/api/customer/health`

Returns API and database status.

**Response (200):**

```json
{
    "status": "healthy",
    "database": true,
    "timestamp": "2026-04-18T12:00:00+00:00"
}
```

**Response (503) — degraded:**

```json
{
    "status": "degraded",
    "database": false,
    "timestamp": "2026-04-18T12:00:00+00:00"
}
```

---

## 1. Authentication

### 1.1 Register (Email/Password)

**POST** `/api/customer/register`

Creates a new customer account and **emails a 6-digit OTP** to verify the email
address. The account is created immediately and a token is returned so the app
can proceed, but `email_verified_at` stays `null` until the code is confirmed via
[§1.1.1 Verify Email OTP](#111-verify-email-otp). Email verification uses an OTP
code — **no verification link is sent**. Login is **not** blocked while an account
is unverified.

**Body:**

```json
{
    "first_name": "John",
    "last_name": "Doe",
    "phone": "+43123456789",
    "email": "john@example.com",
    "password": "password123",
    "password_confirmation": "password123"
}
```

**Response (201):**

```json
{
    "user": {
        "id": 1,
        "customer_public_id": "cust_abc123...",
        "first_name": "John",
        "last_name": "Doe",
        "email": "john@example.com",
        "phone": "+43123456789",
        "account_type": "registered",
        "registration_source": "email",
        "email_verified_at": null
    },
    "token": "1|abc123..."
}
```

**Validation:**

- `first_name`: required, string, max 255
- `last_name`: required, string, max 255
- `phone`: required, string, max 30, unique
- `email`: required, email, max 255, unique
- `password`: required, string, min 8, confirmed

**Notes:**

- A registration OTP is emailed on success. Codes are 6 digits and expire after
  10 minutes.
- Verify the email with [§1.1.1](#111-verify-email-otp) and resend the code with
  [§1.1.2](#112-resend-email-otp).

---

### 1.1.1 Verify Email OTP

**POST** `/api/customer/register/verify-otp`

Confirms the email address using the 6-digit code sent at registration. On success
`email_verified_at` is set.

**Body:**

```json
{
    "email": "john@example.com",
    "otp": "123456"
}
```

**Validation:**

- `email`: required, email
- `otp`: required, string (the 6-digit code)

**Response (200):**

```json
{
    "message": "Email address verified.",
    "user": {
        "id": 1,
        "email": "john@example.com",
        "email_verified_at": "2026-07-24T10:15:00+00:00"
    }
}
```

**Response (200) — already verified:**

```json
{
    "message": "Email address is already verified.",
    "user": { "id": 1, "email": "john@example.com" }
}
```

**Errors:**

- `422` (`otp`): The code is invalid, expired, or has been used. After 5 wrong
  attempts the code is burned and a new one must be requested.
- `422` (`email`): No account found for this email address.

---

### 1.1.2 Resend Email OTP

**POST** `/api/customer/register/resend-otp`

Sends a fresh registration OTP to an unverified account. The response is always the
same generic message so the endpoint does not reveal whether an email is registered.

**Body:**

```json
{
    "email": "john@example.com"
}
```

**Validation:**

- `email`: required, email

**Response (200):**

```json
{
    "message": "If the account exists and is unverified, a verification code has been sent."
}
```

**Errors:**

- `422` (`email`): A new code was requested before the resend cooldown (60 seconds)
  elapsed — the message states how long to wait.

---

### 1.2 Login as Guest

**POST** `/api/customer/guest`

Creates a brand-new guest customer and returns a Bearer token. The customer is created with `account_type = "guest"` and a system-generated unique email, phone, and password — none of those values need to be supplied. The client may optionally pass a display name.

**Body (all fields optional):**

```json
{
    "first_name": "Alice",
    "last_name": "Smith"
}
```

**Validation:**

- `first_name`: nullable, string, max 255 (defaults to `"Guest"` when omitted)
- `last_name`: nullable, string, max 255 (defaults to a random 5–6 character uppercase string when omitted)

**Response (201):**

```json
{
    "user": {
        "id": 42,
        "customer_public_id": "cust_abc123...",
        "first_name": "Alice",
        "last_name": "Smith",
        "email": "guest_a1b2c3d4e5@tavlo.guest",
        "phone": "guest-a1b2c3d4e5",
        "account_type": "guest",
        "registration_source": "guest"
    },
    "token": "5|guesttoken..."
}
```

**Notes:**

- Each call creates a **new** guest customer — the endpoint is not idempotent. The caller should persist the returned token and reuse it instead of calling this endpoint repeatedly.
- The generated `email` and `phone` are placeholders, only used to satisfy the unique constraints on `customers.email` / `customers.phone`. They are not real contact addresses.
- A guest can later be upgraded to a registered account by the standard registration flow (out of scope here).

---

### 1.3 Login (Email/Password)

**POST** `/api/customer/login`

**Body:**

```json
{
    "email": "john@example.com",
    "password": "password123"
}
```

**Response (200):**

```json
{
  "user": { ... },
  "token": "2|xyz456..."
}
```

**Errors:**

- `422`: Invalid credentials

---

### 1.4 Social Register (Google / Apple / Facebook)

**POST** `/api/customer/social/register`

Creates a new customer or links a social account to an existing email. The `access_token` is verified server-side against the provider's API before proceeding.

**Body:**

```json
{
    "provider": "google",
    "access_token": "ya29.a0AfH6SM...",
    "first_name": "John",
    "last_name": "Doe",
    "phone": "+43123456789"
}
```

**Response (200):**

```json
{
  "user": { ... },
  "token": "3|token..."
}
```

**Validation:**

- `provider`: required, in: `google`, `apple`, `facebook`
- `access_token`: required, string — the token from NextAuth (access token for Google/Facebook, identity token for Apple)
- `first_name`: nullable, string — fallback if provider doesn't return a name (e.g. Apple)
- `last_name`: nullable, string
- `phone`: nullable, string

**Token Verification:**

- **Google**: Verified via `googleapis.com/oauth2/v3/userinfo` — extracts `sub`, `email`, `given_name`, `family_name`
- **Apple**: JWT identity token verified using Apple's public keys from `appleid.apple.com/auth/keys` — validates signature, issuer, and expiry; extracts `sub`, `email`
- **Facebook**: Verified via `graph.facebook.com/me` — extracts `id`, `email`, `first_name`, `last_name`

**Behavior:**

- If the social provider ID already exists → returns existing user with new token
- If the provider returned an email and a customer with that email exists but has no social link → links social account to existing customer
- If neither exists → creates new customer. `phone` and `email` are stored as `null` when not provided/returned by the provider (Apple omits the email after the first authorization). Email linking is skipped when the provider returns no email.
- `email_verified_at` is set on creation only when the provider returned an email
- Returns `422` with `access_token` error if the token is invalid or expired
- Returns `422` with `email` error if a unique-constraint race occurs that cannot be resolved to an existing account

---

### 1.5 Social Login (Google / Apple / Facebook)

**POST** `/api/customer/social/login`

The `access_token` is verified server-side against the provider's API.

**Body:**

```json
{
    "provider": "google",
    "access_token": "ya29.a0AfH6SM..."
}
```

**Response (200):**

```json
{
  "user": { ... },
  "token": "4|token..."
}
```

**Validation:**

- `provider`: required, in: `google`, `apple`, `facebook`
- `access_token`: required, string

**Errors:**

- `422` (`access_token`): Invalid or expired token
- `422` (`provider`): No account found for this social provider — user must register first

---

### 1.6 Get Current User

**GET** `/api/customer/me` 🔒

**Response (200):**

```json
{
    "id": 1,
    "customer_public_id": "cust_abc123...",
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@example.com",
    "phone": "+43123456789",
    "gender": "male",
    "date_of_birth": "1990-01-15",
    "address": "123 Main St",
    "profile_picture": null,
    "account_type": "registered"
}
```

---

### 1.7 Logout

**POST** `/api/customer/logout` 🔒

Revokes the current access token.

**Body:**

```json
{}
```

**Response (200):**

```json
{
    "message": "Logged out."
}
```

---

### 1.8 Logout All Devices

**POST** `/api/customer/logout-all` 🔒

Revokes all access tokens for the customer.

**Body:**

```json
{}
```

**Response (200):**

```json
{
    "message": "Logged out from all devices."
}
```

---

### 1.9 Authorize Customer Pusher Channel

**POST** `/api/customer/broadcasting/auth` 🔒

Authorizes the authenticated customer to subscribe to their private Pusher channel. Laravel
Echo calls this endpoint automatically when joining `private-customer.{customer_id}`.

**Authentication:** required (`Authorization: Bearer {customer_token}`).

**Request body:**

```json
{
    "socket_id": "1234.5678",
    "channel_name": "private-customer.42"
}
```

**Response (200):**

```json
{
    "auth": "pusher-app-key:channel-signature"
}
```

The customer may authorize only the channel matching their authenticated customer ID. The response
contains a signature bound to that socket and channel; the Pusher application secret is never
returned to the client.

**Errors:**

- `401` — missing or invalid customer Bearer token.
- `403` — attempted subscription to another customer's channel.
- `422` — missing or invalid `socket_id` / `channel_name`.
- `503` — Pusher is not selected or its application key/secret is not configured.

Customer Pusher messages use event name `.notification.created` and contain the same `event_id`,
`event`, `message`, and `metadata` values as the persisted notification. During the dual-publish
rollout, clients deduplicate the Pusher and Supabase copies using `metadata.event_id`. Customer
state updates do not depend on the notification row being inserted first.

When a waiter creates an order or adds items to an existing unpaid waiter order, every active
customer at that table receives `order_updated`. Its metadata contains a `participant` with
`customer_id: null` and `name: "Waiter"`, plus an authoritative `state_patch` containing the full
changed order and its visible items. The frontend inserts the waiter participant first and then
applies the patch directly to table history, order detail, and tracking caches. No notification-list,
orders, or table-history request is made in response to this Pusher event.

---

### 1.10 Forgot Password (Send OTP)

**POST** `/api/customer/password/forgot`

Starts the OTP-based password reset flow. If the email belongs to a registered
(non-guest) account, a **6-digit OTP is emailed** — **no reset link is sent**. The
response is always the same generic message so the endpoint does not reveal whether
an email is registered.

**Authentication:** public route; no Bearer token required.

**Body:**

```json
{
    "email": "john@example.com"
}
```

**Validation:**

- `email`: required, email

**Response (200):**

```json
{
    "message": "If an account exists for this email, a reset code has been sent."
}
```

**Notes:**

- Codes are 6 digits and expire after 10 minutes.
- Guest accounts do not receive a reset code.
- A resend cooldown of 60 seconds applies per email; requesting a new code too soon
  returns `422` with an `email` error stating how long to wait.

---

### 1.11 Verify Password OTP

**POST** `/api/customer/password/verify-otp`

Optional convenience step: checks a password-reset code **without consuming it**, so
the client can validate the code on one screen before collecting the new password on
the next. The same code is still valid for [§1.12 Reset Password](#112-reset-password).

**Authentication:** public route; no Bearer token required.

**Body:**

```json
{
    "email": "john@example.com",
    "otp": "123456"
}
```

**Validation:**

- `email`: required, email
- `otp`: required, string (the 6-digit code)

**Response (200):**

```json
{
    "message": "Code verified."
}
```

**Errors:**

- `422` (`otp`): The code is invalid, expired, or has been used.

---

### 1.12 Reset Password

**POST** `/api/customer/password/reset`

Completes the reset: verifies the OTP and sets the new password. The code is
single-use, so it cannot be replayed after a successful reset. All of the customer's
existing access tokens are revoked, so they must log in again with the new password.

**Authentication:** public route; no Bearer token required.

**Body:**

```json
{
    "email": "john@example.com",
    "otp": "123456",
    "password": "brand-new-pass",
    "password_confirmation": "brand-new-pass"
}
```

**Validation:**

- `email`: required, email
- `otp`: required, string (the 6-digit code)
- `password`: required, string, min 8, confirmed

**Response (200):**

```json
{
    "message": "Password has been reset. Please log in with your new password."
}
```

**Errors:**

- `422` (`otp`): The code is invalid, expired, or has been used. After 5 wrong
  attempts the code is burned and a new one must be requested via §1.10.
- `422` (`email`): No account found for this email address.
- `422` (`password`): Fails validation (too short, or confirmation mismatch).

---

## 2. Profile

### 2.1 Get Profile Overview

**GET** `/api/customer/profile` 🔒

Returns profile info, recent restaurants, and loyalty overview.

**Response (200):**

```json
{
    "profile": {
        "id": 1,
        "customer_public_id": "cust_abc123",
        "first_name": "Sara",
        "last_name": "Khan",
        "phone": "+43123456789",
        "email": "sara@example.com",
        "gender": "female",
        "date_of_birth": "1990-01-15",
        "address": "123 Main Street, Vienna",
        "profile_picture": "http://localhost:8000/media/customers/1/avatar/abc123.jpg",
        "monthly_orders": 2,
        "orders_count": 18
    },
    "recent_restaurants": [
        {
            "id": 1,
            "vendor_public_id": "V-ABC123",
            "restaurant_name": "Café Central",
            "slug": "cafe-central"
        }
    ],
    "loyalty_overview": [
        {
            "id": 1,
            "customer_id": 1,
            "vendor_id": 1,
            "points_balance": 150,
            "vendor": { "vendor_public_id": "...", "restaurant_name": "..." }
        }
    ]
}
```

- `monthly_orders`: number of paid, non-draft orders placed by the authenticated customer in the current calendar month.
- `orders_count`: total number of paid, non-draft orders placed by the authenticated customer.

---

### 2.2 Update Profile

**PATCH / PUT** `/api/customer/profile` 🔒

**JSON body (all optional):**

```json
{
    "first_name": "Sara",
    "last_name": "Khan",
    "gender": "male",
    "date_of_birth": "1990-01-15",
    "address": "123 Main Street, Vienna"
}
```

**Multipart body (all optional):**
Use `multipart/form-data` when uploading a new profile image. Send the file using the same `profile_picture` field name:

| Field             | Type   | Notes                                                                        |
| ----------------- | ------ | ---------------------------------------------------------------------------- |
| `first_name`      | string | Optional                                                                     |
| `last_name`       | string | Optional                                                                     |
| `gender`          | string | Optional:`male`, `female`, `other`, `prefer_not_to_say`                      |
| `date_of_birth`   | date   | Optional, before today                                                       |
| `address`         | string | Optional                                                                     |
| `profile_picture` | file   | Optional image file. Backend stores it and returns the generated public URL. |

**Validation:**

- `first_name`: nullable, string, max 255
- `last_name`: nullable, string, max 255
- `gender`: nullable, in: `male`, `female`, `other`, `prefer_not_to_say`
- `date_of_birth`: nullable, date, before today
- `address`: nullable, string, max 500
- `profile_picture`: nullable. For multipart uploads: image file (`jpg`, `jpeg`, `png`, `webp`), max 4MB. For JSON/backward compatibility: string URL/path, max 500.

**Profile picture behavior:**

- If `profile_picture` is sent as a file, the backend stores it under the customer avatar media directory and saves the stored path.
- Responses return `profile_picture` as an absolute `/media/...` URL.
- Do not send a generated URL for new uploads. Send the image file in `multipart/form-data`.
- Sending a string URL/path in JSON is only supported for backward compatibility.
- If `profile_picture` is omitted, the current picture is unchanged.
- If `profile_picture` is sent as `null`, the picture value is cleared.

**Response (200):**

```json
{
  "message": "Profile updated.",
  "user": { ... }
}
```

---

### 2.3 Change Phone Number

**PUT** `/api/customer/profile/change-phone` 🔒

**Body:**

```json
{
    "new_number": "+43123456789"
}
```

**Validation:**

- `new_number`: required, string, max 30, unique among customer phone numbers except the authenticated customer

**Response (200):**

```json
{
  "message": "Phone number updated.",
  "user": { ... }
}
```

---

### 2.4 Change Email Address

**PUT / POST** `/api/customer/profile/change-email` 🔒

**Body:**

```json
{
    "current_email": "old@example.com",
    "new_email": "new@example.com"
}
```

**Validation:**

- `current_email`: required, email, must match the authenticated customer's current email address
- `new_email`: required, email, max 255, unique among customer email addresses except the authenticated customer

**Response (200):**

```json
{
  "message": "Email address updated.",
  "user": { ... }
}
```

---

### 2.5 Change Password

**POST** `/api/customer/profile/password` 🔒

**Body:**

```json
{
    "current_password": "old-password",
    "password": "new-password123",
    "password_confirmation": "new-password123"
}
```

**Response (200):**

```json
{
    "message": "Password changed successfully."
}
```

---

## 3. Restaurant Browsing (Public)

### 3.1 List Categories

**GET** `/api/customer/categories`

Returns all active menu categories across discoverable restaurants (deduplicated by name).

**Response (200):**

```json
[
    {
        "id": 1,
        "name": "Burger",
        "slug": "burger",
        "icon": "http://localhost:8000/media/categories/burger.png"
    },
    {
        "id": 2,
        "name": "Pizza",
        "slug": "pizza",
        "icon": "http://localhost:8000/media/categories/pizza.png"
    },
    { "id": 3, "name": "Starters", "slug": "starters", "icon": null }
]
```

**Notes:**

- Only categories from restaurants with `is_live_and_discoverable = true` are included.
- Categories are deduplicated by name and sorted alphabetically.
- `icon` is the absolute URL of the category icon image, or `null` if no icon is set.
- Use the `id` value as the `cuisine` filter param in List Restaurants.

---

### 3.2 List Restaurants

**GET** `/api/customer/restaurants`

**Query Parameters:**

| Param          | Type   | Description                                                              |
| -------------- | ------ | ------------------------------------------------------------------------ |
| `search`       | string | Search by restaurant name or city                                        |
| `city`         | string | Filter by city                                                           |
| `cuisine`      | int    | Filter by menu category ID                                               |
| `price_range`  | int    | Price bracket:`1` = €0–10, `2` = €10–25, `3` = €25–50, `4` = €50+        |
| `service_type` | string | `dine_in`, `takeaway`, or `reservation`                                  |
| `rating`       | float  | Minimum average rating (e.g.`4`)                                         |
| `distance`     | float  | Max distance in km (requires`latitude` and `longitude`)                  |
| `latitude`     | float  | Customer's current latitude                                              |
| `longitude`    | float  | Customer's current longitude                                             |
| `sort_by`      | string | `name` (default), `distance` (requires `latitude`/`longitude`), `rating` |
| `per_page`     | int    | Items per page (default: 20)                                             |
| `page`         | int    | Page number                                                              |

**Response (200):**

```json
{
    "data": [
        {
            "vendor_public_id": "V-ABC123",
            "slug": "buffalo-burger",
            "restaurant_name": "Buffalo Burger",
            "description": "Authentic smash burgers since 2010.",
            "city": "Vienna",
            "address": "Herrengasse 14",
            "latitude": 48.2092,
            "longitude": 16.3666,
            "logo_url": "http://localhost:8000/media/vendors/1/logo/iOSwbdVpswLKtayOyYNvAIpp1OKbtZamP6XQOuKg.png",
            "cover_photo_url": "http://localhost:8000/media/vendors/1/cover/Enl7hMpbuJ6GoYZ60TV415fa75oSHcTLreFlOgxf.png",
            "currency": "EUR",
            "cuisines": ["burger", "Fast food"],
            "price_label": "Budget-friendly",
            "avg_rating": 4.2,
            "review_count": 890,
            "is_open": true,
            "today_hours": "10:45 – 20:45",
            "business_hours": {
                "monday": {
                    "open": "10:45",
                    "close": "20:45",
                    "closed": false
                },
                "tuesday": {
                    "open": "10:45",
                    "close": "20:45",
                    "closed": false
                },
                "wednesday": {
                    "open": "10:45",
                    "close": "20:45",
                    "closed": false
                },
                "thursday": {
                    "open": "10:45",
                    "close": "20:45",
                    "closed": false
                },
                "friday": {
                    "open": "10:45",
                    "close": "22:00",
                    "closed": false
                },
                "saturday": {
                    "open": "11:00",
                    "close": "22:00",
                    "closed": false
                },
                "sunday": { "closed": true }
            },
            "payment_methods": {
                "on-site": true,
                "stripe": false
            },
            "loyalty": {
                "enabled": true,
                "points_per_euro": 20
            },
            "service_types": ["dine_in", "takeaway", "reservation"],
            "distance_km": 1.8,
            "is_favorite": false
        }
    ],
    "current_page": 1,
    "last_page": 3,
    "per_page": 20,
    "total": 48
}
```

**Notes:**

- `description` is the restaurant's profile description from vendor settings. `null` if the vendor has not set one.
- `cuisines` is derived from the restaurant's active menu categories.
- `price_label` is computed from the average menu item price: `Budget-friendly` (≤€10), `Mid-range` (€10–25), `Fine dining` (€25–50), `Premium` (€50+). `null` if no menu items.
- `latitude` / `longitude` are the restaurant's coordinates (may be `null` if not set by the vendor).
- `currency` is derived from the restaurant's selected country via `countries.currency`.
- `is_open` is computed from `business_hours` for the current day/time, evaluated in the vendor's timezone. Windows that span midnight (e.g. `18:00`–`02:00`) are handled correctly, including in the early-morning hours after midnight. `today_hours` shows today's open–close range, or `null` if closed today.
- `business_hours` is a per-day map. Open/close values use the vendor's `24h` or `12h` setting; each day is otherwise shaped as `{ "open": "...", "close": "...", "closed": false }` or `{ "closed": true }`.
- `distance_km` is only returned when `latitude` and `longitude` are provided in the request.
- `cuisine` filter matches by menu category ID.
- `price_range` filter checks if the vendor has active menu items in the given price bracket.
- `service_type` filter:
    - `dine_in` — vendor has at least one row in `restaurant_tables`.
    - `takeaway` — vendor has a takeaway QR configured (`vendor_takeaway_qrs`).
    - `reservation` — vendor's setting `enable_reservations = true`.
- `service_types` (response) — array containing any combination of `dine_in`, `takeaway`, `reservation` indicating which services this restaurant offers (computed from the same sources as the filter).
- Only restaurants with `is_live_and_discoverable = true` are returned.

---

### 3.3 Restaurant Profile

**GET** `/api/customer/restaurants/{vendorPublicId}`

**Query Parameters (optional):**

| Param       | Type  | Description                                 |
| ----------- | ----- | ------------------------------------------- |
| `latitude`  | float | Customer's current latitude (for distance)  |
| `longitude` | float | Customer's current longitude (for distance) |

**Response (200):**

```json
{
    "vendor_public_id": "V-ABC123",
    "slug": "buffalo-burger",
    "restaurant_name": "Buffalo Burger",
    "description": "Authentic smash burgers since 2010.",
    "city": "Maadi",
    "address": "Maadi Street 9, Building 86, next to Al-Ezzabi Pharmacy",
    "latitude": 29.9602,
    "longitude": 31.2569,
    "logo_url": "http://localhost:8000/media/vendors/1/logo/iOSwbdVpswLKtayOyYNvAIpp1OKbtZamP6XQOuKg.png",
    "cover_photo_url": "http://localhost:8000/media/vendors/1/cover/Enl7hMpbuJ6GoYZ60TV415fa75oSHcTLreFlOgxf.png",
    "currency": "EUR",
    "cuisines": ["burger", "Fast food"],
    "avg_rating": 4.2,
    "review_count": 890,
    "is_open": true,
    "today_hours": "10:45 – 20:45",
    "business_hours": {
        "monday": { "open": "10:00", "close": "22:00", "closed": false },
        "tuesday": { "open": "10:00", "close": "22:00", "closed": false },
        "wednesday": { "open": "10:00", "close": "22:00", "closed": false },
        "thursday": { "open": "10:00", "close": "22:00", "closed": false },
        "friday": { "open": "10:00", "close": "23:00", "closed": false },
        "saturday": { "open": "11:00", "close": "23:00", "closed": false },
        "sunday": { "open": "11:00", "close": "21:00", "closed": false }
    },
    "distance_km": 1.8,
    "payment_methods": {
        "on-site": true,
        "stripe": false
    },
    "loyalty": {
        "enabled": true,
        "points_per_euro": 20
    },
    "service_types": ["dine_in", "takeaway", "reservation"]
}
```

**Notes:**

- `is_open` is computed from the vendor's `business_hours` for the current day/time, evaluated in the vendor's timezone. Windows that span midnight (e.g. `18:00`–`02:00`) are handled correctly, including in the early-morning hours after midnight.
- `today_hours` shows today's open–close range, or `null` if closed today.
- `today_hours` and `business_hours` use the vendor's saved time format.
- `distance_km` is only returned when `latitude` and `longitude` are provided.
- `description` is the restaurant's profile description from vendor settings. `null` if the vendor has not set one.
- `cuisines` is derived from the restaurant's active menu categories.
- `is_favorite` is `true` when the request is authenticated and the customer has favorited this restaurant; `false` otherwise (including for unauthenticated requests). Not shown above — also returned in the response payload.

---

### 3.4 Get Restaurant Categories

**GET** `/api/customer/restaurants/{vendorPublicId}/categories`

**Request Header:** `Accept-Language: de`

**Response (200):**

```json
[
    {
        "id": 1,
        "name": "Vorspeisen",
        "slug": "starters",
        "icon": "http://localhost:8000/media/categories/starters.png",
        "sort_order": 1
    },
    { "id": 2, "name": "Mains", "slug": "mains", "icon": null, "sort_order": 2 }
]
```

**Notes:**

- `icon` is the absolute URL of the category icon image, or `null` if no icon is set.
- `name` uses a vendor-specific category translation when available, then the master category translation, and finally the English/base name.
- The response includes `Content-Language` with the resolved locale.
- The response includes `Vary: Accept-Language` for language-aware caching.

---

### 3.5 Get Restaurant Menu

**GET** `/api/customer/restaurants/{vendorPublicId}/menu`

**Request Header:** `Accept-Language: de`

**Query Parameters:**

| Param         | Type   | Description                                    |
| ------------- | ------ | ---------------------------------------------- |
| `category_id` | int    | Filter by category (optional)                  |
| `search`      | string | Search by item name (optional)                 |
| `lang`        | string | Requested menu locale, for example`en` or `de` |

**Response (200):**

```json
[
    {
        "id": 42,
        "name": "4 Piece chicken Box",
        "description": "4 Piece of hand-breaded original chicken...",
        "image_url": "http://localhost:8000/media/menu-items/42/photo/abc123.jpg",
        "price": 20.89,
        "has_discount": false,
        "discount_percent": null,
        "discounted_price": null,
        "vat_rate": 10,
        "tax_category": "food",
        "rating": 4.4,
        "review_count": 252,
        "ordered_count": 1200,
        "popularity_rank": 4,
        "calories": 680,
        "fat": 32.5,
        "carbs": 45.0,
        "protein": 38.0,
        "dietary_preference": "Vegetarian",
        "paid_addons": [
            { "id": 5, "name": "Extra cheese", "price": 1.65, "vat_rate": 10 }
        ],
        "free_addons": ["Ketchup"],
        "free_addon_options": [{ "id": 8, "name": "Ketchup" }],
        "removable_items": ["Onions"],
        "removable_item_options": [{ "id": 11, "name": "Onions" }],
        "category": {
            "id": 1,
            "name": "Burger",
            "slug": "burger",
            "icon": "http://localhost:8000/media/categories/burger.png"
        },
        "allergens": ["Gluten", "Eggs"],
        "tags": ["Popular", "Spicy"],
        "modifier_groups": [
            {
                "id": 1,
                "name": "Choose your side",
                "type": "single",
                "is_required": true,
                "min_selection": 1,
                "max_selection": 1,
                "vat_rate": 10,
                "options": [
                    { "id": 1, "name": "Fries", "price_adjustment": 0.0 },
                    { "id": 2, "name": "Onion Rings", "price_adjustment": 1.65 }
                ]
            }
        ]
    }
]
```

**Notes:**

- **All prices are VAT-inclusive (gross).** The database stores net prices; the API computes gross = net × (1 + vat_rate/100) at response time using the `tax_categories` table for the vendor's country.
- Both `category_id` and `search` are optional. If omitted, all active and currently available menu items are returned.
- The restaurant must be live/discoverable (`vendor_settings.is_live_and_discoverable = true`); hidden restaurants return 404 for menu, menu item detail, categories, tables, reviews, and about endpoints.
- `popularity_rank` is computed from `ordered_count` (e.g. `4` = "#4 most liked").
- `rating` is a percentage-style approval score (e.g. `88%` with `252` reviews).
- `discount_percent` and `discounted_price` are only present when `has_discount` is `true`.
- `vat_rate` is resolved from the `tax_categories` table for the vendor's country and the item's `tax_category` (e.g. food=10%, beverage=20% in Austria).
- Each item includes its `category` for grouping/display.
- `paid_addons` include `id` and `vat_rate` per add-on. Prices are gross.
- `free_addon_options` and `removable_item_options` include IDs for cart submissions.
- `modifier_groups` include `vat_rate` at the group level. Option `price_adjustment` values are gross.
- `free_addons` and `removable_items` remain legacy name arrays for display compatibility.
- `modifier_groups` are reusable vendor-defined option groups attached to the item. Only active groups and options are returned.
- Item names/descriptions, category names, modifier group names, option names, allergen names, tag labels, and dietary preference names all use the same resolved locale.
- The response includes `Content-Language` with the resolved locale.
- The response includes `Vary: Accept-Language` for language-aware caching.

---

### 3.6 Get Menu Item Detail

**GET** `/api/customer/restaurants/{vendorPublicId}/menu/{itemId}`

**Request Header:** `Accept-Language: de`

**Response (200):**

```json
{
    "id": 42,
    "name": "4 Piece chicken Box",
    "description": "4 Piece of hand-breaded original chicken with our special sauce and coleslaw.",
    "image_url": "http://localhost:8000/media/menu-items/42/photo/abc123.jpg",
    "price": 20.89,
    "has_discount": true,
    "discount_percent": 15.0,
    "discounted_price": 17.75,
    "vat_rate": 10,
    "tax_category": "food",
    "available": true,
    "rating": 4.4,
    "review_count": 252,
    "ordered_count": 1200,
    "calories": 680,
    "fat": 32.5,
    "carbs": 45.0,
    "protein": 38.0,
    "dietary_preference": "Vegetarian",
    "paid_addons": [
        { "id": 5, "name": "Extra cheese", "price": 1.65, "vat_rate": 10 }
    ],
    "free_addons": ["Ketchup"],
    "free_addon_options": [{ "id": 8, "name": "Ketchup" }],
    "removable_items": ["Onions"],
    "removable_item_options": [{ "id": 11, "name": "Onions" }],
    "ingredients": ["Chicken breast", "Breadcrumbs", "Flour", "Coleslaw"],
    "category": {
        "id": 1,
        "name": "Burger",
        "slug": "burger"
    },
    "allergens": [
        { "id": 1, "name": "Gluten", "icon": "🌾" },
        { "id": 3, "name": "Eggs", "icon": "🥚" }
    ],
    "tags": [
        { "id": 1, "label": "Popular", "icon": "🔥" },
        { "id": 4, "label": "Spicy", "icon": "🌶️" }
    ],
    "modifier_groups": [
        {
            "id": 1,
            "name": "Choose your side",
            "type": "single",
            "is_required": true,
            "min_selection": 1,
            "max_selection": 1,
            "vat_rate": 10,
            "options": [
                { "id": 1, "name": "Fries", "price_adjustment": 0.0 },
                { "id": 2, "name": "Onion Rings", "price_adjustment": 1.65 },
                {
                    "id": 3,
                    "name": "Sweet Potato Fries",
                    "price_adjustment": 2.2
                }
            ]
        }
    ]
}
```

**Notes:**

- **All prices are VAT-inclusive (gross).** See §3.5 notes for details.
- Only active and currently available items are returned by this endpoint; unavailable items return 404.
- `ingredients` is a list of ingredient names (from the JSON column).
- `fat`, `carbs`, `protein` are in grams; `null` if not set.
- `dietary_preference` is the localized dietary preference name (for example, `Vegetarian` or `Vegetarisch`), or `null`.
- `allergens` and `tags` include `icon` for UI display. Names/labels are locale-aware (resolved from translation tables).
- `modifier_groups` shows customisation options — `type` is `single` or `multiple`, with `min_selection`/`max_selection` constraints. Group-level `vat_rate` applies to all options.
- `discount_percent` and `discounted_price` are only present when `has_discount` is `true`.
- Only active modifier groups and options are returned.
- The response includes `Content-Language` with the resolved locale.
- The response includes `Vary: Accept-Language` for language-aware caching.

---

### 3.7 Get Restaurant Reviews

**GET** `/api/customer/restaurants/{vendorPublicId}/reviews`

Returns all public (non-flagged) reviews for a restaurant, with reviewer info and the menu item being reviewed (if any).

**Query Parameters:**

| Param         | Type   | Description                             |
| ------------- | ------ | --------------------------------------- |
| `rating`      | int    | Filter by star rating (1–5)             |
| `with_images` | bool   | Only return reviews that include images |
| `sort_by`     | string | `recent` (default), `highest`, `lowest` |
| `per_page`    | int    | Items per page (default: 20)            |
| `page`        | int    | Page number                             |

**Response (200):**

```json
{
    "data": [
        {
            "review_public_id": "rev_abc123...",
            "rating": 5,
            "text": "Loved the burger, juicy and fresh!",
            "images": [
                "http://localhost:8000/media/reviews/15/photos/img1.jpg",
                "http://localhost:8000/media/reviews/15/photos/img2.jpg"
            ],
            "created_at": "18.04.2026 14:32",
            "reviewer": {
                "name": "John D.",
                "profile_picture": "http://localhost:8000/media/customers/1/avatar/abc123.jpg"
            },
            "menu_items": [
                {
                    "id": 42,
                    "name": "4 Piece chicken Box",
                    "slug": "4-piece-chicken-box",
                    "image_url": "http://localhost:8000/media/menu-items/42/photo/abc123.jpg",
                    "quantity": 1
                },
                {
                    "id": 57,
                    "name": "Caesar Salad",
                    "slug": "caesar-salad",
                    "image_url": "http://localhost:8000/media/menu-items/57/photo/def456.jpg",
                    "quantity": 2
                }
            ],
            "item_reviews": [
                {
                    "menu_item_id": 42,
                    "menu_item_name": "4 Piece chicken Box",
                    "menu_item_image": "http://localhost:8000/media/menu-items/42/photo/abc123.jpg",
                    "rating": 5,
                    "text": "Perfectly crispy!"
                },
                {
                    "menu_item_id": 57,
                    "menu_item_name": "Caesar Salad",
                    "menu_item_image": "http://localhost:8000/media/menu-items/57/photo/def456.jpg",
                    "rating": 4,
                    "text": null
                }
            ],
            "vendor_reply": "Thank you for the kind words!",
            "vendor_replied_at": "19.04.2026 08:10"
        }
    ],
    "current_page": 1,
    "last_page": 3,
    "per_page": 20,
    "total": 48,
    "review_summary": {
        "average_rating": 4.7,
        "total_reviews": 128,
        "rating_breakdown": [
            { "star": 5, "count": 80, "percent": 62.5 },
            { "star": 4, "count": 30, "percent": 23.4 },
            { "star": 3, "count": 10, "percent": 7.8 },
            { "star": 2, "count": 5, "percent": 3.9 },
            { "star": 1, "count": 3, "percent": 2.4 }
        ]
    }
}
```

**Notes:**

- Only non-flagged reviews are returned.
- `menu_items` is derived from `cart_items` linked to the review's paid order: owned rows where `cart_items.order_id` matches the review order, plus shared rows whose `shared_order_ids` include that order ID. Each entry includes the item `name`, `quantity` ordered, and — when the vendor still has a matching menu item — its `id`, `slug`, and `image_url`. `id`, `name`, `slug`, and `image_url` are `null` if the menu item can no longer be resolved.
- `item_reviews` contains per-item ratings and review text submitted by the customer. Each entry includes the `menu_item_id`, `menu_item_name`, `menu_item_image`, `rating` (1–5), and optional `text`.
- `images` is an array of image URLs uploaded by the reviewer (may be empty).
- `reviewer.name` falls back to `"Anonymous"` if the customer has no name set.
- `created_at` and `vendor_replied_at` use the vendor's saved date and time formats.
- `review_summary` is computed across **all** non-flagged reviews for this restaurant — it is independent of the `rating`, `with_images`, and pagination filters, so the breakdown stays stable while the user filters the list.
    - `average_rating` is rounded to 1 decimal (0 if there are no reviews).
    - `rating_breakdown` always includes all 5 buckets (5★ → 1★), with `count` and `percent` (rounded to 1 decimal).

---

### 3.8 Get Restaurant About

**GET** `/api/customer/restaurants/{vendorPublicId}/about`

Returns the public "About" profile for a restaurant — vanity stats, features, payment methods, legal/location info, working hours, and contact details (only the contact fields the vendor has marked as public).

**Response (200):**

```json
{
    "vendor_public_id": "V-ABC123",
    "restaurant_name": "Buffalo Burger",
    "description": "Authentic smash burgers since 2010.",
    "years_of_experience": 15,
    "signature_recipes_count": 12,
    "happy_customers_count": 25400,
    "restaurant_features": [
        {
            "title": "Free Wi-Fi",
            "description": "High-speed wireless throughout the venue."
        },
        {
            "title": "Outdoor seating",
            "description": "Heated terrace open year-round."
        },
        {
            "title": "Parking",
            "description": "Free customer parking next door."
        },
        { "title": "Wheelchair accessible", "description": null },
        {
            "title": "Vegan options",
            "description": "Dedicated vegan menu section."
        }
    ],
    "payment_methods": {
        "on-site": true,
        "stripe": true
    },
    "vat_number": "ATU12345678",
    "address": "Herrengasse 14",
    "city": "Vienna",
    "country": "Austria",
    "latitude": 48.2092,
    "longitude": 16.3666,
    "business_hours": {
        "monday": { "open": "10:00", "close": "22:00", "closed": false },
        "tuesday": { "open": "10:00", "close": "22:00", "closed": false },
        "wednesday": { "open": "10:00", "close": "22:00", "closed": false },
        "thursday": { "open": "10:00", "close": "22:00", "closed": false },
        "friday": { "open": "10:00", "close": "23:00", "closed": false },
        "saturday": { "open": "11:00", "close": "23:00", "closed": false },
        "sunday": { "closed": true }
    },
    "service_types": ["dine_in", "takeaway", "reservation"],
    "contact": {
        "phone": "+43 1 234 5678",
        "website": "https://example.com"
    }
}
```

**Notes:**

- `restaurant_features` is a list of structured feature objects chosen by the vendor. Each entry has:
    - `title` — short label (string, required, max 100)
    - `description` — optional longer explanation (string, max 500, may be `null`)
- `restaurant_features` titles and descriptions are translatable. When an `Accept-Language` header is present and the vendor has stored translations for that language, the response resolves to the translated values. Missing translations fall back to the English base values.
- `payment_methods["on-site"]` reflects whether customers can pay staff at the restaurant. `payment_methods.stripe` is `true` only when Stripe is enabled and the vendor's Stripe Connect account is onboarded.
- `contact` is a partial object — each field is only included when the vendor has marked it as publicly visible:
    - `phone` requires `show_phone_public = true` (default `true`)
    - `email` requires `show_email_public = true` (default `false`)
    - `website` requires `show_website_public = true` (default `true`)
- Numeric stats (`years_of_experience`, `signature_recipes_count`, `happy_customers_count`) are `null` when the vendor hasn't set them.
- `is_favorite` is included in the response (`true`/`false`). Always `false` for unauthenticated requests.

---

### 3.9 Get Restaurant Tables

**GET** `/api/customer/restaurants/{vendorPublicId}/tables`

**Response (200):**

```json
[{ "id": 1, "number": 1, "name": "Table 1" }]
```

---

### 3.9.1 Get Restaurant Languages

**GET** `/api/customer/restaurants/{vendorPublicId}/languages`

Returns the fixed English fallback, customer-facing languages, date format, and time format enabled by the vendor in settings.

**Authentication:** public route; no Bearer token required.

**Request body:** none.

**Response (200):**

```json
{
    "vendor": { "id": "VID-8492", "name": "Bella Italia" },
    "default_language": "en",
    "available_languages": ["en", "de", "it"],
    "date_format": "DD.MM.YYYY",
    "time_format": "24h",
    "languages": [
        { "code": "en", "name": "English", "is_default": true },
        { "code": "de", "name": "Deutsch (German)", "is_default": false },
        { "code": "it", "name": "Italiano (Italian)", "is_default": false }
    ]
}
```

**Notes:**

- `default_language` is always `en`; it is no longer stored per vendor.
- `available_languages` comes from `vendor_settings.supported_languages`.
- `date_format` comes from `vendor_settings.date_format`.
- `time_format` comes from `vendor_settings.time_format`.
- Vendor-linked date/time fields in customer responses are already formatted using these values.
- English is always included first.
- Supported codes are `en`, `de`, `it`, `fr`, `ar`, `tr`, `zh`, `ja`, `sr`, `cs`, `es`, and `nl`.

### Customer Locale Resolution

The restaurant categories, menu list, and menu-item detail endpoints resolve one locale for the complete response:

1. Use `?lang={code}` when present and it requests an enabled language. This remains available as an explicit override.
2. When `lang` is absent, select the best enabled language from the `Accept-Language` header, including regional codes such as `de-DE` and quality weights.
3. If no requested language is enabled, use English.
4. Missing translated fields fall back to English/base content.

The resolved locale is returned in the `Content-Language` response header. Responses also include `Vary: Accept-Language`.

**Response (404):**

```json
{ "message": "No query results for model [App\\Models\\Vendor]." }
```

---

### 3.10 Get Table Status

**GET** `/api/customer/table/status?token=d5938525-f2a5-4849-803e-d579582af11f`

Checks the current table state before calling the table scan endpoint.

**Authentication:** public route; no Bearer token required.

**Request body:** none.

**Query parameters:**

- `token` — QR token from the scanned table link (string, required).

**Response (200):**

```json
{
    "table": { "id": "5", "number": 3, "name": "T3" },
    "vendor": { "id": "VID-8492", "name": "Bella Italia" },
    "status": "active"
}
```

**Status values:**

- `available` — the table has no active scan session.
- `draft` — the table has active scan session(s), but no cart items have been added.
- `active` — the table has active scan session(s) and at least one cart item.

**Response (410) — invalid / inactive QR token:**

```json
{ "message": "This QR code is no longer valid" }
```

**Response (422) — missing token:**

```json
{
    "message": "The token field is required.",
    "errors": {
        "token": ["The token field is required."]
    }
}
```

---

### 3.11 Scan Table QR (Create Session) 🔒

**POST** `/api/customer/table/scan`

Customer scans a printed table QR code. If the table has no active session, this creates the owner session with a unique 4-digit PIN that can be shared with joining customers.

**Authentication:** required (Bearer token).

**Body (recommended — plain text):**

Send the QR token as the raw request body.

Header:
`Content-Type: text/plain`

Body:

```
d5938525-f2a5-4849-803e-d579582af11f
```

**Body (also accepted — JSON, backwards compatible):**

```json
{ "token": "d5938525-f2a5-4849-803e-d579582af11f" }
```

**Response (201):**

```json
{
    "message": "Table session started",
    "status": "active",
    "requiresPin": false,
    "pin": "0473",
    "session": {
        "id": "12",
        "status": "active",
        "scannedAt": "23.04.2026 10:15"
    },
    "table": { "id": "5", "number": 3, "name": "T3" },
    "vendor": { "id": "VID-8492", "name": "Bella Italia", "currency": "EUR" }
}
```

**Response (201) — same customer scans the active table again:**

```json
{
    "message": "Table session was already started",
    "status": "active",
    "requiresPin": false,
    "pin": "0473",
    "session": {
        "id": "12",
        "status": "active",
        "scannedAt": "23.04.2026 10:15"
    },
    "table": { "id": "5", "number": 3, "name": "T3" },
    "vendor": { "id": "VID-8492", "name": "Bella Italia", "currency": "EUR" }
}
```

**Response (409) — table already has an active session:**

```json
{
    "message": "This table already has an active session",
    "status": "active",
    "requiresPin": true,
    "pin": null,
    "session": {
        "id": "12",
        "status": "active",
        "scannedAt": "23.04.2026 10:15"
    },
    "table": { "id": "5", "number": 3, "name": "T3" },
    "vendor": { "id": "VID-8492", "name": "Bella Italia", "currency": "EUR" }
}
```

> **Note:** The `pin` field is always `null` in 409 responses. The joining customer must receive the PIN through an out-of-band channel (verbally, or via a shared screen) and submit it to `POST /api/customer/table/pin`.

**Flow note:**

- If scan returns `201`, `requiresPin = false` because the scanning customer is already in the newly-created table session. The returned `pin` is for sharing with other customers.
- If the same customer scans again while their table session is still active, scan returns `201` with the existing session data and `requiresPin = false`; no new session is created.
- If scan returns `409` with `status = "active"`, the UI should show a PIN entry form.
- The customer then submits that PIN to `POST /api/customer/table/pin` to join the already-active table session.

**Response (410) — invalid / inactive QR token:**

```json
{ "message": "This QR code is no longer valid" }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 3.12 Join Active Table With PIN 🔒

**POST** `/api/customer/table/pin`

When a table is already active, another customer can join that same table flow by entering the PIN shown by the first customer.

**Authentication:** required (Bearer token).

**Body:**

```json
{
    "token": "d5938525-f2a5-4849-803e-d579582af11f",
    "pin": "0473"
}
```

**Response (201):**

```json
{
    "message": "Joined table session",
    "status": "active",
    "requiresPin": false,
    "pin": "0473",
    "session": {
        "id": "13",
        "status": "active",
        "scannedAt": "23.04.2026 10:17"
    },
    "table": { "id": "5", "number": 3, "name": "T3" },
    "vendor": { "id": "VID-8492", "name": "Bella Italia", "currency": "EUR" }
}
```

**Behavior:**

- A new `table_scan_sessions` row is created for the joining customer.
- The entered `pin` is stored on the joining customer's `table_scan_sessions` row.
- The joining customer receives the same PIN they entered in the response.
- Repeating the same request for a customer who is already joined returns the existing active session instead of creating a duplicate row.

**Response (200) — customer already joined:**

```json
{
    "message": "Already joined this table session",
    "status": "active",
    "requiresPin": false,
    "pin": "0473",
    "session": {
        "id": "13",
        "status": "active",
        "scannedAt": "23.04.2026 10:17"
    },
    "table": { "id": "5", "number": 3, "name": "T3" },
    "vendor": { "id": "VID-8492", "name": "Bella Italia", "currency": "EUR" }
}
```

**Response (422) — invalid PIN:**

```json
{ "message": "The provided PIN is invalid for this table" }
```

**Response (410) — invalid / inactive QR token:**

```json
{ "message": "This QR code is no longer valid" }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 3.12.1 Check My Table Session Status 🔒

**GET** `/api/customer/table/session/status`

Checks whether the authenticated customer still has an active table session. No QR token, table ID, or request body is required.

**Authentication:** required (Bearer token).

**Request body:** none.

**Response (200) — active session:**

```json
{
    "active": true,
    "session": {
        "id": "13",
        "status": "active",
        "scannedAt": "23.04.2026 10:17"
    },
    "table": {
        "id": "5",
        "number": 3,
        "name": "T3"
    },
    "vendor": {
        "id": "V-ABC123",
        "name": "Bella Italia",
        "currency": "EUR"
    }
}
```

**Response (200) — no active session:**

```json
{
    "active": false,
    "session": null,
    "table": null,
    "vendor": null
}
```

Only sessions belonging to the authenticated customer with `status = "active"` are considered. Active sessions belonging to other customers are ignored.

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 3.13 Close Table Session 🔒

**POST** `/api/customer/table/close`

Closes **all** active table scan sessions for the given table. Any authenticated customer can call this endpoint.

**Authentication:** required (Bearer token).

**Body:**

```json
{
    "table_id": 5
}
```

**Validation:**

- `table_id`: required, integer — the `restaurant_tables.id` for the table to close.
- `vendor_public_id`: optional, string — still accepted for backward compatibility. When supplied, the active sessions must also belong to that restaurant.

**Close rules:**

- If any session user has an unpaid, non-draft, non-cancelled order → blocked.
- If any session user has paid orders with unserved cart items → blocked.
- Otherwise → all active sessions for this table are closed.

**Response (200):**

```json
{
    "message": "Table session closed",
    "status": "closed",
    "session": {
        "id": "13",
        "status": "closed",
        "scannedAt": "23.04.2026 10:17",
        "closedAt": "23.04.2026 11:42"
    },
    "table": { "id": "5", "number": 3, "name": "T3" },
    "vendor": { "id": "V-ABC123", "name": "Bella Italia", "currency": "EUR" }
}
```

**Notes:**

- Closing affects all active sessions at the table, not just the caller's.
- Draft orders do not block closing the session.
- Paid orders with all items served do not block closing.

**Response (422) — active order exists on the table:**

```json
{ "message": "There is an active order on this table." }
```

**Response (422) — paid order has unserved items:**

```json
{ "message": "All the items on table are not served." }
```

**Response (422) — no active table session:**

```json
{ "message": "No active table session found." }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 3.14 Call Waiter

**POST** `/api/customer/table/call`

Sends a notification to all active waiters at the restaurant that owns the supplied table.

**Authentication:** not required.

**Request body:**

```json
{
    "table_id": 42,
    "note": "Please bring the bill."
}
```

**Rules:**

- `table_id` must be the ID of an existing restaurant table.
- `note` is optional, must be a string when provided, and may contain up to 500 characters. It is included in the waiter notification.
- The table determines which restaurant's waiters are notified; no table scan session is required.
- Only team members with role `waiter` and status `active` are notified.
- If no active waiters exist at the restaurant → blocked.

**Response (200):**

```json
{ "message": "Waiters have been notified." }
```

**Response (422) — missing or invalid table ID:**

```json
{
    "message": "The table id field is required.",
    "errors": {
        "table_id": ["The table id field is required."]
    }
}
```

**Response (422) — no waiters available:**

```json
{ "message": "No waiters available at this restaurant." }
```

---

### 3.15 Table Cart 🔒

A **table cart** is the live cart of every customer currently sitting at the same `restaurant_table`. It is automatically scoped to the authenticated customer's currently-active `table_scan_session`.

**Authentication:** required (Bearer token).

**Scope rule:** the customer must have a row in `table_scan_sessions` with `status = active`. The cart includes every active session at the same `restaurant_table_id`.

If the customer has no active session, every cart endpoint returns:

```json
{ "message": "No active table session found." }
```

with HTTP `422`.

#### 3.14.1 Get Table Cart

**GET** `/api/customer/cart`

Returns all visible open cart items per person at the same table. Draft orders do **not** bind or hide cart rows: items remain visible while the customer is still reviewing or sharing a draft. When an order is confirmed, the customer's open cart rows are bound to that order through `cart_items.order_id` and disappear from the open cart. Newly added items after confirmation stay open (`order_id = null`) until the customer confirms the open order again or starts/confirms a new order after payment.

**Response (200):**

```json
{
    "table": {
        "id": 5,
        "number": 3,
        "name": "T3"
    },
    "vendor": {
        "vendor_public_id": "V-ABC123",
        "restaurant_name": "Buffalo Burger"
    },
    "people": [
        {
            "session_id": 12,
            "customer_id": 7,
            "is_me": true,
            "name": "Alice Smith",
            "personal_items": [
                {
                    "id": 1,
                    "quantity": 2,
                    "notes": "No salt",
                    "price": 5.5,
                    "paid_addons": [
                        {
                            "name": "Cheese sauce",
                            "price": 1.65,
                            "vat_rate": 10
                        }
                    ],
                    "free_addons": ["Ketchup"],
                    "removed_items": ["Salt"],
                    "selected_modifiers": [
                        {
                            "modifier_group_id": 1,
                            "name": "Choose your side",
                            "type": "single",
                            "is_required": true,
                            "min_selection": 1,
                            "max_selection": 1,
                            "vat_rate": 10,
                            "options": [
                                {
                                    "id": 2,
                                    "name": "Onion Rings",
                                    "price_adjustment": 1.65
                                }
                            ]
                        }
                    ],
                    "vat_rate": 10,
                    "vat_amount": 1.0,
                    "line_total": 11.0,
                    "menu_item": {
                        "id": 42,
                        "name": "Fries",
                        "price": 3.85,
                        "vat_rate": 10,
                        "tax_category": "food",
                        "image_url": null
                    }
                }
            ],
            "tax_groups": [
                {
                    "code": "A",
                    "label": "Food (10%)",
                    "tax_category": "food",
                    "vat_rate": 10,
                    "vat_amount": 1.0,
                    "gross_amount": 11.0,
                    "net_amount": 10.0
                }
            ],
            "totals": {
                "net_total": 10.0,
                "vat_total": 1.0,
                "service_fee": 0.0,
                "grand_total": 11.0
            }
        },
        {
            "session_id": 13,
            "customer_id": 8,
            "is_me": false,
            "name": "Bob Jones",
            "personal_items": [],
            "tax_groups": [],
            "totals": {
                "net_total": 0.0,
                "vat_total": 0.0,
                "service_fee": 0.0,
                "grand_total": 0.0
            }
        }
    ]
}
```

**Notes:**

- `tax_groups` and `totals` are computed **per person** from that person's open cart items.

#### 3.14.2 Add Item

**POST** `/api/customer/cart/items`

When `CUSTOMER_ASYNC_COMMANDS_ENABLED=true`, cart add/update/remove and table-order
share/unshare writes are accepted into the ordered Redis command stream. The
request is authenticated and structurally validated before this response. The
frontend keeps its optimistic state until the matching realtime event contains
`command_status: "completed"`.

```json
{
    "message": "Change accepted.",
    "command_id": "0190f26e-7c87-7def-8e46-111111111111",
    "sequence": 12,
    "operation": "cart.add",
    "status": "accepted"
}
```

The legacy synchronous `200`/`201` response remains active whenever the feature
is disabled or Redis enqueueing is unavailable.

Adds an item to the authenticated customer's cart. If the same `menu_item_id` already exists in the customer's current session, the quantity is incremented instead of creating a duplicate entry.

**Body:**

```json
{
    "menu_item_id": 42,
    "quantity": 2,
    "notes": "No salt",
    "paid_addons": [{ "id": 5 }],
    "free_addons": [{ "id": 8 }],
    "removed_items": [{ "id": 11 }],
    "selected_modifiers": [
        {
            "modifier_group_id": 1,
            "option_ids": [2]
        }
    ]
}
```

**Validation:**

- `menu_item_id`: required, must exist in `menu_items`
- `quantity`: optional, integer, 1–99 (default `1`)
- `notes`: optional, string, max 500
- `paid_addons`: optional array of selected paid add-on objects. Prefer `{ "id": 5 }`; legacy `{ "name": "Cheese sauce" }` is still accepted. Submitted prices are ignored and replaced with the vendor-configured price for that menu item.
- `free_addons`: optional array of selected free add-ons. Prefer objects with `id`; legacy name strings are still accepted.
- `removed_items`: optional array of selected removed items. Prefer objects with `id`; legacy name strings are still accepted. `removable_items` is still accepted for backward compatibility.
- `selected_modifiers`: optional array of selected modifier groups. Each entry accepts `modifier_group_id` and `option_ids`. `modifiers` is also accepted as an alias. Selected groups must be attached to the menu item, options must belong to the group, and required/min/max group rules are enforced.

**Behavior:**

- If a cart item with the same `menu_item_id` already exists for the customer's active `table_scan_session`, the existing item's quantity is incremented by the requested amount (default `1`). The existing item is returned.
- A cart item is only merged with an existing row when `menu_item_id`, `paid_addons`, `free_addons`, `removed_items`, and `selected_modifiers` all match. Different customization choices create separate cart rows.
- Selected customization options must exist on the menu item. Invalid selections return `422`.
- Returned customization names use `Accept-Language` and the restaurant's enabled languages. Missing translations fall back to English/base content.
- Paid add-on prices and selected modifier `price_adjustment` values are included in `price`, `line_total`, order draft totals, and payment totals.
- If no matching cart item exists, a new one is created.

**Response (201):**

```json
{
    "id": 1,
    "quantity": 2,
    "notes": "No salt",
    "price": 5.5,
    "paid_addons": [
        { "id": 5, "name": "Cheese sauce", "price": 1.65, "vat_rate": 10 }
    ],
    "free_addons": ["Ketchup"],
    "removed_items": ["Salt"],
    "selected_modifiers": [
        {
            "modifier_group_id": 1,
            "name": "Choose your side",
            "type": "single",
            "is_required": true,
            "min_selection": 1,
            "max_selection": 1,
            "vat_rate": 10,
            "options": [
                { "id": 2, "name": "Onion Rings", "price_adjustment": 1.65 }
            ]
        }
    ],
    "vat_rate": 10,
    "vat_amount": 1.0,
    "line_total": 11.0,
    "menu_item": {
        "id": 42,
        "name": "Fries",
        "price": 3.85,
        "vat_rate": 10,
        "tax_category": "food",
        "image_url": null
    }
}
```

#### 3.14.3 Update Item

**PATCH** `/api/customer/cart/items/{id}`

Update quantity or notes on an item the customer owns.

**Body (all optional):**

```json
{
    "quantity": 3,
    "notes": "Extra crispy"
}
```

**Response (200):** updated item payload.

**Response (404):** if the item does not belong to the customer's current session.

#### 3.14.4 Remove Item

**DELETE** `/api/customer/cart/items/{id}`

Removes an item owned by the current session.

**Response (204):** empty.

**Response (404):** if the item does not belong to the customer's current session.

---

### 3.16 Table Payment Summary 🔒

**GET** `/api/customer/table/order/start`

Returns a payment-ready snapshot of the authenticated customer's current table:

- one node per active session at the same table (name, items, total)
- the table-wide grand total

**Authentication:** required (Bearer token).

**Scope rule:** the customer must have an `active` row in `table_scan_sessions`. The summary is computed across every active session at the same `restaurant_table_id`.

**Response (200):**

```json
{
    "table": {
        "id": 5,
        "number": 3,
        "name": "T3"
    },
    "people": [
        {
            "session_id": 12,
            "customer_id": 7,
            "is_me": true,
            "name": "Alice Smith",
            "item_count": 3,
            "total_price": 14.3,
            "items": [
                {
                    "cart_item_id": 1,
                    "menu_item_id": 42,
                    "name": "Fries",
                    "image_url": null,
                    "quantity": 2,
                    "unit_price": 3.85,
                    "total_price": 7.7,
                    "is_mine": true
                },
                {
                    "cart_item_id": 2,
                    "menu_item_id": 51,
                    "name": "Coke",
                    "image_url": null,
                    "quantity": 1,
                    "unit_price": 6.6,
                    "total_price": 6.6,
                    "is_mine": true
                }
            ],
            "tax_groups": [
                {
                    "code": "A",
                    "label": "Food (10%)",
                    "tax_category": "food",
                    "vat_rate": 10,
                    "vat_amount": 1.3,
                    "gross_amount": 14.3,
                    "net_amount": 13.0
                }
            ],
            "totals": {
                "net_total": 13.0,
                "vat_total": 1.3,
                "service_fee": 0.0,
                "grand_total": 14.3
            }
        },
        {
            "session_id": 13,
            "customer_id": 8,
            "is_me": false,
            "name": "Bob Jones",
            "item_count": 1,
            "total_price": 20.89,
            "items": [
                {
                    "cart_item_id": 3,
                    "menu_item_id": 42,
                    "name": "4 Piece chicken Box",
                    "image_url": "http://localhost:8000/media/menu-items/42/photo/abc123.jpg",
                    "quantity": 1,
                    "unit_price": 20.89,
                    "total_price": 20.89,
                    "is_mine": false
                }
            ],
            "tax_groups": [
                {
                    "code": "A",
                    "label": "Food (10%)",
                    "tax_category": "food",
                    "vat_rate": 10,
                    "vat_amount": 1.9,
                    "gross_amount": 20.89,
                    "net_amount": 18.99
                }
            ],
            "totals": {
                "net_total": 18.99,
                "vat_total": 1.9,
                "service_fee": 0.0,
                "grand_total": 20.89
            }
        }
    ],
    "summary": {
        "item_count": 4,
        "total_price": 35.19
    }
}
```

**Notes:**

- The current table is derived from the customer's active `table_scan_sessions` row — no `table_id` needs to be passed in the URL or body.
- The authenticated customer's own line is included in `people[]` and identified by `is_me: true`.
- `is_mine` on an item is `true` when the item belongs to the authenticated customer's session.
- `unit_price` and `total_price` are VAT-inclusive (gross) floats in the restaurant's currency. `total_price` per item = `unit_price × quantity`.
- `summary.total_price` equals the sum of `people[].total_price`.
- `name` falls back to `"Guest"` if the joined customer has no name set.

**Response (422) — no active table session:**

```json
{ "message": "No active table session found." }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 3.17 Create Order Draft 🔒

**POST** `/api/customer/table/order/draft`

Creates a `draft` order for the authenticated customer's active table session. The amount is computed from the customer's currently open owned `cart_items` (`order_id = null`) plus any selected shared items. If the customer already has an unpaid draft for this active session, the draft amount is refreshed instead of creating another draft. If the customer already has an unpaid, non-cancelled submitted order for this active session, this endpoint returns the table view without creating a new draft; added open items are bound to that existing order only when confirm is called again.

**Authentication:** required (Bearer token).

**Body:**

```json
{}
```

No request body is required.

**Response (201):** unified table-view payload — see [§4.1 Get Current Table History](#41-get-current-table-history) for the full shape.

**Response (422) — no active table session:**

```json
{ "message": "No active table session found." }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 3.18 Update Order 🔒

**PUT** `/api/customer/table/order/update/{order_id}`

Share or unshare a `cart_item` for the caller's draft order. At least one of `shared_item` or `unshared_item` must be provided; both may be sent together.

**Authentication:** required (Bearer token).

**Path parameters:**

- `order_id`: integer. ID of the order to update. Must belong to the authenticated customer.

**Body:**

```json
{
    "shared_item": 3,
    "unshared_item": 5
}
```

**Validation:**

- `shared_item`: nullable integer. ID of a `cart_item` belonging to **another** customer at the same table. The caller's `order_id` is appended to that cart_item's `shared_order_ids` array. Returns `422` if the item is already shared with the caller's order, or if the item belongs to an order already assigned to the caller for payment.
- `unshared_item`: nullable integer. ID of a `cart_item` at the same table. The caller's `order_id` is removed from that cart_item's `shared_order_ids` array. If the caller was not sharing this item, the operation is a silent no-op.
- At least one of `shared_item` or `unshared_item` must be present.

**Side-effect — order amount recalculation:** after sharing or unsharing, the `amount` of every affected unpaid order is recalculated. Affected orders include the caller's order, the cart item's owner order, and all other orders in the item's `shared_order_ids`. This keeps order amounts in sync with the current sharing state even when sharing changes after order confirmation.

**Response behavior:** the endpoint returns a compact response centered on an authoritative `state_patch`, together with compatibility `order_snapshots` and `removed_order_ids`; it does not rebuild or return the complete table-history payload. The frontend applies `state_patch` directly to its existing table-history cache, so a successful share/unshare does not require another `GET /api/customer/table/history` request.

The exact same patch is included at `metadata.state_patch` in the `order_updated` realtime notification sent to every active customer at the table. The initiating customer can therefore use the mutation response immediately, while tablemates apply the realtime copy. Both copies have the same `id` and are safe to deduplicate.

**Response (200):**

```json
{
    "message": "Item sharing updated.",
    "order_snapshots": [
        {
            "order_id": 42,
            "order_public_id": "ord-aB3xK9pQrS12",
            "table_scan_session_id": 12,
            "status": "confirmed",
            "amount": 10.45,
            "tip_amount": 0,
            "service_fee": 0,
            "currency": "EUR",
            "payment_method": null,
            "payment_pending": false,
            "payment_received": false,
            "paid_by": null
        }
    ],
    "removed_order_ids": [],
    "state_patch": {
        "id": "019b10e7-9924-7371-9008-11979e306858",
        "version": 1784123456796000,
        "operation": "order.sharing_updated",
        "orders": {
            "upsert": [
                {
                    "id": 42,
                    "order_public_id": "ord-aB3xK9pQrS12",
                    "parent_order_id": null,
                    "customer_id": 7,
                    "table_scan_session_id": 12,
                    "status": "confirmed",
                    "amount": 10.45,
                    "service_fee": 0,
                    "tip_amount": 0,
                    "currency": "EUR",
                    "payment_pending": false,
                    "payment_received": false,
                    "paid_by": null,
                    "visible_item_ids": [3]
                }
            ],
            "remove_ids": []
        },
        "items": {
            "upsert": [
                {
                    "cart_item_id": 3,
                    "menu_item_id": 51,
                    "owner_order_id": 87,
                    "owner_table_scan_session_id": 18,
                    "name": "Pizza",
                    "quantity": 1,
                    "shared_order_ids": [42],
                    "shared_between": 2,
                    "shared_with": [
                        {
                            "order_id": 42,
                            "customer_id": 7,
                            "customer_name": "Alice Smith"
                        }
                    ],
                    "unit_price": 20.89,
                    "line_total": 20.89,
                    "share_price": 10.45,
                    "vat_rate": 10,
                    "vat_amount": 1.9,
                    "tax_category": "food"
                }
            ],
            "remove_ids": []
        }
    }
}
```

`orders.upsert` contains only affected orders, each with an authoritative `visible_item_ids` list. `items.upsert` contains only affected item rows with current ownership, pricing, tax, status, and sharing fields. An empty side order deleted by unsharing appears in `orders.remove_ids`. The arrays are absolute state, not increments; clients should deduplicate by patch `id` and ignore an older `version` after applying a newer one.

**Response (404) — order not found:**

```json
{ "message": "Order not found." }
```

**Response (422) — neither field provided:**

```json
{ "message": "Provide shared_item or unshared_item." }
```

**Response (422) — invalid shared item:**

```json
{ "message": "This item is already shared with your order." }
```

```json
{ "message": "You are already paying for this item." }
```

**Response (422) — shared cart item not at this table:**

```json
{ "message": "Shared cart item does not belong to this table." }
```

**Response (422) — caller tried to share their own cart item:**

```json
{ "message": "You cannot share your own cart item with yourself." }
```

**Response (422) — unshared cart item not at this table:**

```json
{ "message": "Unshared cart item does not belong to this table." }
```

**Response (422) — no active table session:**

```json
{ "message": "No active table session found." }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 3.18.1 Get Customer Command Status 🔒

**GET** `/api/customer/commands/{command_id}`

Returns the short-lived status of an ordered cart/share command. This is a
reconnect/recovery endpoint; normal clients reconcile from Pusher metadata and
do not poll it.

**Authentication:** required (Bearer token). A customer can read only their own
command IDs.

**Request body:** none.

**Response (200):**

```json
{
    "command_id": "0190f26e-7c87-7def-8e46-111111111111",
    "sequence": 12,
    "status": "completed",
    "http_status": 201,
    "response": { "id": 92 }
}
```

**Response (404):**

```json
{ "message": "Command not found or expired." }
```

---

### 3.19 Create Order Confirmed 🔒

**POST** `/api/customer/table/order/confirmed`

Confirms the authenticated customer's open order for the active table session. **No request body is accepted.** The endpoint recomputes the final `amount`, updates a draft order to `status = "confirmed"`, and binds currently open owned cart rows by setting `cart_items.order_id`.

If no draft or submitted order exists, the endpoint **auto-creates a draft order** from the customer's open cart items and immediately confirms it — callers do not need to call the draft endpoint first.

If the same customer already has another unpaid, non-cancelled submitted order for the same active table session, no new submitted order is created. The endpoint merges the currently open owned cart rows into that existing unpaid order, migrates any draft sharing references to that order, removes the now-empty draft, and recalculates the existing order amount.

**Authentication:** required (Bearer token).

**Body:**

```json
{}
```

**Total computation (from `cart_items`):**

1. **Owned items** — every already-bound row where `cart_items.order_id = O.id`, plus currently open rows for the caller's active session during confirmation, contributes `(unit_price × quantity) / (1 + count(shared_order_ids))`.
2. **Shared-into items** — every `cart_item` whose `shared_order_ids` JSON array contains this order's id (and whose session is _not_ the caller's) contributes the same per-share amount.

The final amount is rounded to 2 decimals.

**Response (200):** unified table-view payload — see [§4.1 Get Current Table History](#41-get-current-table-history) for the full shape.

**Response (422) — empty cart:**

```json
{ "message": "Your cart is empty. Add items before confirming your order." }
```

**Response (422) — no active table session:**

```json
{ "message": "No active table session found." }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

## 4. Table History 🔒

### 4.1 Get Current Table History

**GET** `/api/customer/table/history`

Returns the unified table-view payload — table + vendor + session metadata, every active session at the same table, and every order each person has placed in their active table session. Draft orders show the customer's currently open cart rows; confirmed-or-later orders show rows bound through `cart_items.order_id` plus shared rows from `shared_order_ids`. Each person returns `orders` as an array containing all matching orders for that same `session_id`; it is an empty array when no order exists.

This is the **canonical "table view" response**. Order draft and confirmation flows may return this complete shape; [§3.18 Update Order](#318-update-order-) returns only a compact `state_patch` and relies on this endpoint solely for initial loading or recovery.

**Authentication:** required (Bearer token).

**Scope rule:** the customer must have an `active` row in `table_scan_sessions`. `people[]` includes every active session at the same restaurant table.

**Response (200):**

```json
{
    "table": {
        "id": 5,
        "number": 3,
        "name": "T3"
    },
    "vendor": {
        "vendor_public_id": "V-ABC123",
        "restaurant_name": "Buffalo Burger"
    },
    "session": {
        "id": 12,
        "status": "active",
        "scanned_at": "27.04.2026 10:15"
    },
    "people": [
        {
            "session_id": 12,
            "customer_id": 7,
            "is_me": true,
            "name": "Alice Smith",
            "scanned_at": "27.04.2026 10:15",
            "status": "active",
            "orders_count": 1,
            "total_amount": 16.49,
            "orders": [
                {
                    "id": 42,
                    "order_public_id": "ord-aB3xK9pQrS12",
                    "customer_id": 7,
                    "vendor_id": 1,
                    "table_scan_session_id": 12,
                    "status": "confirmed",
                    "amount": 16.49,
                    "tip_amount": 0.0,
                    "currency": "EUR",
                    "order_number": null,
                    "order_type": "dine-in",
                    "table_number": "3",
                    "service_fee": 0.0,
                    "vat_amount": 0.0,
                    "course": null,
                    "payment_method": null,
                    "payment_pending": true,
                    "payment_received": false,
                    "payment_confirmed_at": null,
                    "payment_note": null,
                    "transaction_id": null,
                    "served_at": null,
                    "cancelled_at": null,
                    "cancelled_reason": null,
                    "waiter_confirmed": false,
                    "waiter_confirmed_at": null,
                    "created_at": "27.04.2026 10:30",
                    "updated_at": "27.04.2026 10:31",
                    "items": [
                        {
                            "cart_item_id": 1,
                            "menu_item_id": 42,
                            "name": "Fries",
                            "image_url": null,
                            "quantity": 2,
                            "unit_price": 5.5,
                            "paid_addons": [
                                {
                                    "name": "Cheese sauce",
                                    "price": 1.65,
                                    "vat_rate": 10
                                }
                            ],
                            "free_addons": ["Ketchup"],
                            "removed_items": ["Salt"],
                            "selected_modifiers": [
                                {
                                    "modifier_group_id": 1,
                                    "name": "Choose your side",
                                    "type": "single",
                                    "is_required": true,
                                    "min_selection": 1,
                                    "max_selection": 1,
                                    "vat_rate": 10,
                                    "options": [
                                        {
                                            "id": 2,
                                            "name": "Onion Rings",
                                            "price_adjustment": 1.65
                                        }
                                    ]
                                }
                            ],
                            "vat_rate": 10,
                            "tax_category": "food",
                            "vat_amount": 1.0,
                            "line_total": 11.0,
                            "is_mine": true,
                            "shared_between": 1,
                            "shared_with": [],
                            "my_share": 11.0,
                            "status": "Preparing",
                            "received_at": "27.04.2026 10:30",
                            "preparing_start_at": "27.04.2026 10:31",
                            "ready_at": null,
                            "served_at": null
                        },
                        {
                            "cart_item_id": 3,
                            "menu_item_id": 51,
                            "name": "Pizza",
                            "image_url": null,
                            "quantity": 1,
                            "unit_price": 20.89,
                            "line_total": 20.89,
                            "is_mine": false,
                            "shared_between": 2,
                            "shared_with": [
                                {
                                    "order_id": 101,
                                    "customer_id": 9,
                                    "customer_name": "Bob Jones"
                                }
                            ],
                            "my_share": 10.45,
                            "status": "Ready",
                            "received_at": "27.04.2026 10:30",
                            "preparing_start_at": "27.04.2026 10:32",
                            "ready_at": "27.04.2026 10:42",
                            "served_at": null
                        }
                    ]
                }
            ],
            "tax_groups": [
                {
                    "code": "A",
                    "label": "Food (10%)",
                    "tax_category": "food",
                    "vat_rate": 10,
                    "vat_amount": 2.9,
                    "gross_amount": 31.89,
                    "net_amount": 28.99
                }
            ],
            "totals": {
                "net_total": 28.99,
                "vat_total": 2.9,
                "service_fee": 0.0,
                "total_tips": 0.0,
                "grand_total": 31.89
            }
        }
    ],
    "summary": {
        "orders_count": 1,
        "total_amount": 16.49
    }
}
```

**Per-item field reference:**

| Field                                                               | Type         | Description                                                                                                                                                                                                     |
| ------------------------------------------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cart_item_id`                                                      | int          | Source`cart_items.id`.                                                                                                                                                                                          |
| `menu_item_id`                                                      | int          | The menu item this cart entry references.                                                                                                                                                                       |
| `name`, `image_url`                                                 | string\|null | Cached from`menu_items`.                                                                                                                                                                                        |
| `quantity`                                                          | int          | Cart quantity.                                                                                                                                                                                                  |
| `unit_price`                                                        | float        | Per-unit VAT-inclusive (gross) price, including selected paid add-ons and selected modifier price adjustments.                                                                                                  |
| `paid_addons`, `free_addons`, `removed_items`, `selected_modifiers` | array        | Selected customization options for this cart item. Names are resolved from stored IDs using`Accept-Language`; paid add-ons and modifier options include `vat_rate` and gross prices.                            |
| `vat_rate`                                                          | float        | VAT rate resolved from`tax_categories` table for the vendor's country and item's `tax_category`.                                                                                                                |
| `tax_category`                                                      | string       | Tax category slug (e.g.`food`, `beverage_non_alcoholic`, `beverage_alcoholic`).                                                                                                                                 |
| `vat_amount`                                                        | float        | VAT portion of`line_total`, computed as `line_total - (line_total / (1 + vat_rate/100))`.                                                                                                                       |
| `line_total`                                                        | float        | `unit_price × quantity`.                                                                                                                                                                                        |
| `is_mine`                                                           | bool         | `true` if the cart_item belongs to the caller's own session.                                                                                                                                                    |
| `shared_between`                                                    | int          | `1 + count(shared_order_ids)`. The number of orders splitting this item (owner + sharers).                                                                                                                      |
| `shared_with`                                                       | object[]     | The orders that share this item with the owner. Each entry:`order_id`, `customer_id`, `customer_name`. Empty array if unshared.                                                                                 |
| `my_share`                                                          | float        | `line_total / shared_between` — what each participating order contributes.                                                                                                                                      |
| `status`                                                            | string\|null | Per-item status derived from timestamps:`Served` when `served_at` is set, `Ready` when `ready_at` is set, `Preparing` when `preparing_start_at` is set, `Received` when `received_at` is set, otherwise `null`. |
| `received_at`                                                       | string\|null | Vendor-formatted date and time set when the order is confirmed and the cart item is bound to the order.                                                                                                         |
| `preparing_start_at`, `ready_at`, `served_at`                       | string\|null | Vendor-formatted preparation/service timestamps set by the vendor flow.                                                                                                                                         |

**Item-set rule for an order:**
For a given order `O` in `people[].orders[]`, an item appears in its `items[]` if either:

- (a) `cart_items.order_id = O.id` for confirmed-or-later orders, or `cart_items.order_id IS NULL` for the session's current draft order, or
- (b) the cart_item's `shared_order_ids` contains `O.id` (shared-into).

**Notes:**

- `people[].orders` is ordered oldest-to-newest by `created_at`, then `id`, and is always an array.
- The columns `items_count`, `items`, `shared_items`, `ready_at`, `picked_up_at`, and `guest_count` no longer exist on `orders`. Per-item state lives on `cart_items` (`order_id`, `shared_order_ids`, `preparing_start_at`, `ready_at`, `served_at`); per-order amount is recomputed on confirm.
- `name` falls back to `"Guest"` if the customer has no name set.

**Response (422) — no active table session:**

```json
{ "message": "No active table session found." }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 4.2 Customer Account Order History

**GET** `/api/customer/orders/history`

Returns the authenticated customer's account-level order history grouped by restaurant. Unlike [§4.1](#41-get-current-table-history), this endpoint is not scoped to the currently active table session; it is for the customer's full past order history.

Every customer order object returned by table history, account history, restaurant-specific history, order detail, tracking, and receipts includes the payer identity below. It is `null` unless another customer claimed the order:

```json
{
    "paid_by": {
        "id": 7,
        "name": "Ali Khan"
    }
}
```

**Authentication:** required (Bearer token).

**Query Parameters:**

| Param      | Type | Description                                            |
| ---------- | ---- | ------------------------------------------------------ |
| `page`     | int  | Orders page number (default:`1`).                      |
| `per_page` | int  | Orders per restaurant group (default:`10`, max: `50`). |

**Pagination:** `history[].orders` is paginated per restaurant group. The top-level summary always reflects the full matching history, not only the current page.

**Response (200):**

```json
{
    "history": [
        {
            "restaurant_public_id": "REST-101",
            "restaurant_name": "Bella Italia",
            "restaurant_logo_url": "https://example.com/media/vendors/1/logo.png",
            "currency": "USD",
            "orders_count": 10,
            "total_spent": 240.0,
            "last_ordered_at": "10.10.2024 10:30",
            "orders": [
                {
                    "order_id": "ORD-8801",
                    "order_public_id": "ord-aB3xK9pQrS12",
                    "created_at": "10.10.2024 10:30",
                    "order_type": "dine-in",
                    "payment_status": "paid",
                    "payment_method": "card",
                    "items_count": 2,
                    "total_amount": 24.24,
                    "tip_amount": 0.0,
                    "items": [
                        {
                            "cart_item_id": 42,
                            "menu_item_id": 42,
                            "name": "Tonkotsu Ramen",
                            "image_url": "https://example.com/media/menu-items/42/photo.png",
                            "quantity": 1,
                            "unit_price": 17.86,
                            "paid_addons": [],
                            "free_addons": [],
                            "removed_items": [],
                            "selected_modifiers": [],
                            "vat_rate": 10,
                            "tax_category": "food",
                            "vat_amount": 1.62,
                            "line_total": 17.86,
                            "is_mine": true,
                            "shared_between": 1,
                            "shared_with": [],
                            "my_share": 17.86,
                            "status": "Served",
                            "received_at": "10.10.2024 10:30",
                            "preparing_start_at": "10.10.2024 10:32",
                            "ready_at": "10.10.2024 10:45",
                            "served_at": "10.10.2024 10:47"
                        },
                        {
                            "cart_item_id": 57,
                            "menu_item_id": 57,
                            "name": "Matcha Latte",
                            "image_url": null,
                            "quantity": 1,
                            "unit_price": 9.6,
                            "paid_addons": [],
                            "free_addons": [],
                            "removed_items": [],
                            "selected_modifiers": [],
                            "vat_rate": 20,
                            "tax_category": "beverage_non_alcoholic",
                            "vat_amount": 1.6,
                            "line_total": 9.6,
                            "is_mine": true,
                            "shared_between": 1,
                            "shared_with": [],
                            "my_share": 9.6,
                            "status": "Served",
                            "received_at": "10.10.2024 10:30",
                            "preparing_start_at": null,
                            "ready_at": "10.10.2024 10:35",
                            "served_at": "10.10.2024 10:36"
                        }
                    ],
                    "tax_groups": [
                        {
                            "code": "A",
                            "label": "Food (10%)",
                            "tax_category": "food",
                            "vat_rate": 10,
                            "vat_amount": 1.62,
                            "gross_amount": 17.86,
                            "net_amount": 16.24
                        },
                        {
                            "code": "B",
                            "label": "Beverage (20%)",
                            "tax_category": "beverage_non_alcoholic",
                            "vat_rate": 20,
                            "vat_amount": 1.6,
                            "gross_amount": 9.6,
                            "net_amount": 8.0
                        }
                    ],
                    "totals": {
                        "net_total": 24.24,
                        "vat_total": 3.22,
                        "service_fee": 0.0,
                        "total_tips": 0.0,
                        "grand_total": 27.46
                    }
                }
            ],
            "pagination": {
                "current_page": 1,
                "per_page": 10,
                "total": 10,
                "last_page": 1,
                "has_more": false
            }
        }
    ],
    "summary": {
        "restaurants_count": 1,
        "orders_count": 10,
        "total_spent": 240.0
    }
}
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 4.3 Customer Order Detail

**GET** `/api/customer/orders/{orderPublicId}`

Returns one order detail for the authenticated customer. Access is allowed when the customer placed the order (directly or through their table session) or is assigned as/persisted as the payer. This allows a customer who pays for a tablemate's order to open the post-payment tracking screen without changing account order-history ownership.

**Authentication:** required (Bearer token).

**Payment methods:** `card`, `stripe`, `cash`.

`can_view_receipt` is `true` only when the order is paid and the authenticated customer owns the completed payment. Frontends must hide receipt actions and skip receipt requests when it is `false`.

**Response (200):**

```json
{
    "order_id": "ORD-8842",
    "order_public_id": "ord-aB3xK9pQrS12",
    "restaurant": {
        "restaurant_public_id": "REST-101",
        "restaurant_name": "Bella Italia",
        "logo_url": "https://example.com/media/vendors/1/logo.png"
    },
    "created_at": "24.02.2024 10:30",
    "status": "delivered",
    "order_type": "dine-in",
    "payment_status": "paid",
    "payment_method": "card",
    "can_view_receipt": true,
    "tip_amount": 0.0,
    "items": [
        {
            "cart_item_id": 42,
            "menu_item_id": 42,
            "name": "Tonkotsu Ramen",
            "image_url": null,
            "quantity": 1,
            "unit_price": 17.86,
            "paid_addons": [],
            "free_addons": [],
            "removed_items": [],
            "selected_modifiers": [],
            "vat_rate": 10,
            "tax_category": "food",
            "vat_amount": 1.62,
            "line_total": 17.86,
            "is_mine": true,
            "shared_between": 1,
            "shared_with": [],
            "my_share": 17.86,
            "status": "Served",
            "received_at": "24.02.2024 10:30",
            "preparing_start_at": "24.02.2024 10:32",
            "ready_at": "24.02.2024 10:45",
            "served_at": "24.02.2024 10:47"
        },
        {
            "cart_item_id": 57,
            "menu_item_id": 57,
            "name": "Matcha Latte",
            "image_url": null,
            "quantity": 1,
            "unit_price": 9.6,
            "paid_addons": [],
            "free_addons": [],
            "removed_items": [],
            "selected_modifiers": [],
            "vat_rate": 20,
            "tax_category": "beverage_non_alcoholic",
            "vat_amount": 1.6,
            "line_total": 9.6,
            "is_mine": true,
            "shared_between": 1,
            "shared_with": [],
            "my_share": 9.6,
            "status": "Served",
            "received_at": "24.02.2024 10:30",
            "preparing_start_at": null,
            "ready_at": "24.02.2024 10:35",
            "served_at": "24.02.2024 10:36"
        }
    ],
    "tax_groups": [
        {
            "code": "A",
            "label": "Food (10%)",
            "tax_category": "food",
            "vat_rate": 10,
            "vat_amount": 1.62,
            "gross_amount": 17.86,
            "net_amount": 16.24
        },
        {
            "code": "B",
            "label": "Beverage (20%)",
            "tax_category": "beverage_non_alcoholic",
            "vat_rate": 20,
            "vat_amount": 1.6,
            "gross_amount": 9.6,
            "net_amount": 8.0
        }
    ],
    "totals": {
        "net_total": 24.24,
        "vat_total": 3.22,
        "service_fee": 0.0,
        "total_tips": 0.0,
        "grand_total": 27.46,
        "currency": "USD"
    }
}
```

**Response (404):**

```json
{ "message": "No query results for model [App\\Models\\Order]." }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 4.3.0a List Receipts

**GET** `/api/customer/receipts`

Returns every completed payment the authenticated customer made (a "receipt" is one payment — Stripe intent or confirmed cash — that may cover several orders, e.g. their own plus tablemates' orders claimed via pay-for). Only completed payments appear: status `succeeded` or a non-null `paid_at`. Newest first. Receipt ownership follows the payer: an order placed by the customer but paid by another participant remains in order history but is not included in this customer's receipt list.

**Authentication:** required (Bearer token).

**Response (200):**

```json
{
    "receipts": [
        {
            "receipt_id": 12,
            "payment_intent_id": "pi_3Nxxx",
            "restaurant": {
                "restaurant_public_id": "ven-Xy12",
                "restaurant_name": "Trattoria Roma",
                "logo_url": "https://cdn.example.com/logo.png"
            },
            "amount": 21.63,
            "currency": "EUR",
            "payment_method": "stripe",
            "status": "succeeded",
            "paid_at": "13.07.2026 14:05",
            "orders_count": 2,
            "orders": [
                {
                    "id": 42,
                    "order_public_id": "ord-aB3xK9pQrS12",
                    "amount": 14.42
                },
                {
                    "id": 43,
                    "order_public_id": "ord-Zt5rM2wNqA34",
                    "amount": 7.21
                }
            ]
        }
    ],
    "receipts_count": 1
}
```

`payment_method` is `stripe` when the payment has a PaymentIntent, otherwise `cash`. `orders[].amount` is each order's share of the payment.

---

### 4.3.0b Receipt Detail

**GET** `/api/customer/receipts/{receiptId}`

Returns a structured receipt for one completed payment in the same format as the single-order receipt below, except the single `"order": {}` block is replaced by an `"orders": []` array containing **every order paid by that intent** — the customer's own orders plus any tablemates' orders they paid for. `receiptId` is the numeric `receipt_id` from the list endpoint.

**Authentication:** required (Bearer token). Only the paying customer can access the receipt; another customer's receipt returns 404.

**Language header:** `Accept-Language` behaves exactly as in the single-order receipt.

**Rules:**

- The payment must be completed (status `succeeded` or non-null `paid_at`) — otherwise 422.
- On first access, an invoice number is atomically generated and persisted on the anchor order (shared with the single-order receipt of that order).

**Response (200):**

```json
{
    "data": {
        "restaurant": {
            "name": "Trattoria Roma",
            "logo_url": "https://cdn.example.com/logo.png",
            "address": "Main Street 1, Vienna, AT",
            "vat_id": "ATU12345678",
            "phone": "+43 1 234 5678",
            "email": "office@trattoria.example",
            "company_register_number": "FN 123456a"
        },
        "receipt": {
            "receipt_id": 12,
            "invoice_number": "INV-0001001",
            "date": "13.07.2026",
            "time": "14:05",
            "table": "Table 1",
            "order_ids": ["ord-aB3xK9pQrS12", "ord-Zt5rM2wNqA34"],
            "currency": "EUR",
            "locale": "en-AT"
        },
        "orders": [
            {
                "order_id": "ord-aB3xK9pQrS12",
                "paid_by": null,
                "tip_amount": 2.0,
                "items": []
            },
            {
                "order_id": "ord-Zt5rM2wNqA34",
                "paid_by": { "id": 7, "name": "Ali Khan" },
                "tip_amount": 0,
                "items": []
            }
        ],
        "tax_groups": [],
        "totals": {
            "service_fee": 0.63,
            "total_tips": 2.0,
            "grand_total": 23.63,
            "amount_charged": 23.63
        },
        "payment": {
            "method": "stripe",
            "status": "CONFIRMED",
            "transaction_id": "pi_3Nxxx",
            "paid_at": "13.07.2026 14:05"
        },
        "legal": {}
    },
    "meta": {
        "generated_at": "13.07.2026 14:06",
        "template": "tavlo-receipt-template",
        "version": "1.0"
    }
}
```

Each entry in `orders[].items` uses the same item shape as the single-order receipt (`unit_price_gross`, `line_gross`, VAT fields, shared-item shares, add-ons, modifiers). `tax_groups` and `totals` are computed across all covered orders (shared items counted once); `totals.amount_charged` is the actual amount captured by the payment.

**Response (422):**

```json
{ "message": "Receipt is only available for completed payments." }
```

**Response (404):** unknown `receiptId` or a receipt belonging to another customer.

---

### 4.3.1 Order Receipt

**GET** `/api/customer/orders/{orderPublicId}/receipt`

Returns a structured receipt payload for a paid order, including restaurant legal details, itemised order with VAT breakdown, tax groups, totals, payment confirmation, and legal notes. On first access, an invoice number is atomically generated and persisted on the order.

**Authentication:** required (Bearer token).

**Language header:** `Accept-Language: ar` (or another restaurant-enabled language). Menu item, add-on, removal, and selected modifier names use this language and fall back to English when unavailable. The legal receipt locale remains English plus the vendor country code.

**Rules:**

- The authenticated customer must be the payer of the order. A customer who placed the order but did not pay for it receives 404.
- Completed `order_payments` ownership is authoritative. Legacy self-paid orders without payment attribution remain accessible to their owner for backward compatibility.
- The order must have `payment_received = true` — unpaid orders return 422.

**Response (200):**

```json
{
    "data": {
        "restaurant": {
            "name": "La Bella Cucina",
            "logo_url": "http://localhost:8000/media/vendors/1/logo/abc.png",
            "address": "Mariahilfer Straße 45, Vienna, AT",
            "vat_id": "ATU12345678",
            "phone": "+43 1 234 5678",
            "email": "info@labellacucina.at",
            "company_register_number": "FN 123456 a"
        },
        "receipt": {
            "invoice_number": "INV-0001001",
            "date": "05.06.2026",
            "time": "19:43",
            "table": "Table 4",
            "order_id": "ORD-UHO2XBI1",
            "currency": "EUR",
            "locale": "en-AT"
        },
        "order": {
            "items": [
                {
                    "id": 1,
                    "name": "Bruschetta al Pomodoro",
                    "quantity": 2,
                    "unit_price_gross": 8.9,
                    "line_gross": 17.8,
                    "paid_addons": [
                        {
                            "id": 5,
                            "name": "Cheese sauce",
                            "price": 1.65,
                            "vat_rate": 10
                        }
                    ],
                    "free_addons": ["Ketchup"],
                    "removed_items": ["Salt"],
                    "selected_modifiers": [
                        {
                            "modifier_group_id": 1,
                            "name": "Choose your side",
                            "options": [
                                {
                                    "id": 2,
                                    "name": "Onion Rings",
                                    "price_adjustment": 1.65
                                }
                            ]
                        }
                    ],
                    "tax_category": "FOOD",
                    "vat_rate": 10
                },
                {
                    "id": 2,
                    "name": "Aperol Spritz",
                    "quantity": 1,
                    "unit_price_gross": 9.9,
                    "line_gross": 9.9,
                    "paid_addons": [],
                    "free_addons": [],
                    "removed_items": [],
                    "selected_modifiers": [],
                    "tax_category": "BEVERAGE_ALCOHOLIC",
                    "vat_rate": 20
                }
            ]
        },
        "tax_groups": [
            {
                "code": "A",
                "label": "Food (10%)",
                "tax_category": "FOOD",
                "vat_rate": 10,
                "gross_amount": 17.8,
                "net_amount": 16.18,
                "vat_amount": 1.62
            },
            {
                "code": "B",
                "label": "Beverage Alcoholic (20%)",
                "tax_category": "BEVERAGE_ALCOHOLIC",
                "vat_rate": 20,
                "gross_amount": 9.9,
                "net_amount": 8.25,
                "vat_amount": 1.65
            }
        ],
        "totals": {
            "net_total": 24.43,
            "vat_total": 3.27,
            "service_fee": 2.77,
            "total_tips": 0.0,
            "grand_total": 30.47
        },
        "payment": {
            "method": "stripe",
            "status": "CONFIRMED",
            "transaction_id": "pi_123456789",
            "paid_at": "05.06.2026 19:43"
        },
        "legal": {
            "invoice_note": "This invoice was issued in accordance with § 11 UStG (Austria).",
            "tax_note": "All prices include statutory VAT. The service date corresponds to the invoice date.",
            "company_register_note": "Registration number: FN 123456 a",
            "rksv_required_check": true
        }
    },
    "meta": {
        "generated_at": "05.06.2026 19:43",
        "template": "tavlo-receipt-template",
        "version": "1.0"
    }
}
```

**Field reference:**

| Section                                                                           | Field                                                                          | Source                                                 |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------ |
| `restaurant.name`                                                                 | `vendors.restaurant_name`                                                      |                                                        |
| `restaurant.logo_url`                                                             | `vendor_settings.logo_url`                                                     | Absolute URL                                           |
| `restaurant.address`                                                              | `vendors.address, city, country`                                               | Joined with comma                                      |
| `restaurant.vat_id`                                                               | `vendors.vat_number`                                                           |                                                        |
| `restaurant.company_register_number`                                              | `vendors.business_registration_number`                                         |                                                        |
| `receipt.invoice_number`                                                          | Auto-generated from`vendor_settings.invoice_prefix` + `next_invoice_number`    | Persisted on`orders.invoice_number` after first access |
| `receipt.locale`                                                                  | English + vendor country code                                                  | e.g.`en-AT`, `en-DE`                                   |
| `receipt.table`                                                                   | From`table_scan_session` → `restaurant_tables.name` or `Table {number}`        | `null` if no table session                             |
| `order.items[].name`                                                              | Live menu item name resolved from`menu_item_id`                                | Localized using`Accept-Language`                       |
| `order.items[].paid_addons`, `free_addons`, `removed_items`, `selected_modifiers` | Selected customization IDs or legacy names matched to current menu definitions | Names localized using`Accept-Language`                 |
| `order.items[].tax_category`                                                      | Uppercased`menu_items.tax_category`                                            | e.g.`FOOD`, `BEVERAGE_ALCOHOLIC`                       |
| `payment.status`                                                                  | `CONFIRMED` when `payment_received = true`                                     |                                                        |
| `payment.transaction_id`                                                          | `orders.transaction_id` or `order_payments.stripe_payment_intent_id`           |                                                        |
| `payment.paid_at`                                                                 | `orders.payment_confirmed_at` or `order_payments.paid_at`                      |                                                        |
| `legal.invoice_note`                                                              | Hardcoded per vendor country                                                   | AT: § 11 UStG, DE: § 14 UStG, GB: UK VAT               |
| `legal.rksv_required_check`                                                       | `true` for Austrian vendors                                                    | Hardcoded                                              |

**Invoice number generation:**

- Format: `{invoice_prefix}-{zero-padded 7-digit number}` (e.g. `INV-0001001`).
- Generated atomically on first receipt access — `vendor_settings.next_invoice_number` is incremented with a row lock.
- Subsequent requests for the same order return the persisted `orders.invoice_number` without incrementing.
- `receipt.date`, `receipt.time`, `payment.paid_at`, and `meta.generated_at` use the vendor's saved date/time formats.

**Response (422) — order not paid:**

```json
{ "message": "Receipt is only available for paid orders." }
```

**Response (404):**

```json
{ "message": "No query results for model [App\\Models\\Order]." }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 4.4 Get Restaurant Payment Methods

**GET** `/api/customer/payment-methods?restaurant_id={restaurant_id}`

Returns the customer-facing payment methods currently available for a restaurant.

**Authentication:** public route; no Bearer token required.

**Query Parameters:**

| Parameter          | Type   | Required | Description                                                                       |
| ------------------ | ------ | -------- | --------------------------------------------------------------------------------- |
| `restaurant_id`    | string | yes      | Vendor numeric ID,`vendor_public_id`, or restaurant slug                          |
| `vendor_public_id` | string | no       | Alias for`restaurant_id`; useful when the caller already has the vendor public ID |

**Response (200):**

```json
{
    "method": {
        "on-site": true,
        "stripe": true
    }
}
```

**Rules:**

- `method["on-site"]` mirrors `vendor_settings.accept_on_site`.
- `method.stripe` is `true` only when `stripe_enabled = true`, `stripe_account_id` is present, and `stripe_onboarding_complete = true`.

**Response (422):**

```json
{
    "message": "The restaurant id field is required.",
    "errors": {
        "restaurant_id": ["The restaurant id field is required."]
    }
}
```

**Response (404):**

```json
{ "message": "Restaurant not found." }
```

---

#### Payment Mutation `state_patch`

Successful table-scoped payment mutations include an additive `state_patch`. The frontend applies this patch directly to its existing table-history cache; it does not need to request `GET /api/customer/table/history` after the mutation. The exact same patch is included at `metadata.state_patch` in the realtime notification delivered to the other active customers at the table.

```json
{
    "state_patch": {
        "id": "019b10d7-68d6-7a35-88bd-617293b6b44e",
        "version": 1784123456789000,
        "operation": "payment.assigned",
        "orders": {
            "upsert": [
                {
                    "id": 42,
                    "order_public_id": "ord-aB3xK9pQrS12",
                    "parent_order_id": null,
                    "customer_id": 8,
                    "table_scan_session_id": 15,
                    "status": "confirmed",
                    "amount": 12.65,
                    "service_fee": 0,
                    "tip_amount": 0,
                    "currency": "EUR",
                    "payment_pending": false,
                    "payment_received": false,
                    "paid_by": { "id": 7, "name": "Ali Khan" },
                    "visible_item_ids": [501]
                }
            ],
            "remove_ids": []
        },
        "items": {
            "upsert": [
                {
                    "cart_item_id": 501,
                    "menu_item_id": 9,
                    "owner_order_id": 42,
                    "owner_table_scan_session_id": 15,
                    "name": "Caprese Salad",
                    "quantity": 1,
                    "shared_order_ids": [],
                    "shared_between": 1,
                    "shared_with": [],
                    "unit_price": 12.65,
                    "line_total": 12.65,
                    "share_price": 12.65,
                    "vat_rate": 10,
                    "vat_amount": 1.15,
                    "tax_category": "food"
                }
            ],
            "remove_ids": []
        }
    }
}
```

`orders.upsert` contains complete authoritative rows for only the affected orders, including identity, totals, lifecycle/payment fields, `paid_by`, and `visible_item_ids`. `items.upsert` contains complete display, customization, price, tax, sharing, and preparation-state fields for only the affected items. IDs in either `remove_ids` array must be removed from the local cache. Patches are absolute and idempotent: deduplicate them by `id` and ignore a lower `version` after a newer patch has already been applied.

For readability, the route-specific examples below abbreviate the repeated affected rows as empty `upsert` arrays. Successful mutations populate those arrays whenever orders or items changed, as shown in the populated example above.

| Route                                        | `operation`                                                                         |
| -------------------------------------------- | ----------------------------------------------------------------------------------- |
| `POST /payments/pay-for`                     | `payment.assigned`                                                                  |
| `DELETE /payments/pay-for/{orderId}`         | `payment.assignment_released`                                                       |
| `POST /payments/request-cash`                | `payment.cash_requested`                                                            |
| `POST /payments/create-intent`               | `payment.initiated`                                                                 |
| `POST /payments/update-intent`               | `payment.updated`                                                                   |
| `DELETE /payments/intent`                    | `payment.canceled`                                                                  |
| `GET /payments/verify`                       | `payment.verified`, or `payment.completed` when verification first observes success |
| successful Stripe webhook notification       | `payment.completed`                                                                 |
| waiter/vendor cash-confirmation notification | `payment.cash_confirmed`                                                            |

---

### 4.4.1 Assign a Tablemate's Order for Payment

**POST** `/api/customer/payments/pay-for`

Assigns a single eligible order belonging to another customer at the authenticated customer's current table to the authenticated customer for payment.

**Authentication:** required (Bearer token).

**Body:**

```json
{
    "order_id": "ord-aB3xK9pQrS12"
}
```

`order_id` is required. It accepts either the order's numeric ID or its `order_public_id` and must reference an eligible tablemate order in the current active table session — confirmed or later, unpaid, and not cancelled — otherwise a 422 validation error is returned on `order_id`. Only the referenced order is assigned; the customer's other orders are unaffected. Repeating the same request is idempotent. To pay for several orders, call the endpoint once per order.

If the payer previously shared any item owned by the selected order, those individual share references are removed atomically before full-order coverage is assigned. The returned `state_patch` contains every recalculated order/item and any empty side-order ID that was removed.

On success, every customer with an active session at the table receives an `order_updated` notification identifying the payer and containing the same `state_patch`.

**Response (200):**

```json
{
    "message": "Orders assigned for payment.",
    "paid_by": {
        "id": 7,
        "name": "Ali Khan"
    },
    "orders_count": 1,
    "orders": [
        {
            "id": 42,
            "order_public_id": "ord-aB3xK9pQrS12"
        }
    ],
    "state_patch": {
        "id": "019b10d7-68d6-7a35-88bd-617293b6b44e",
        "version": 1784123456789000,
        "operation": "payment.assigned",
        "orders": { "upsert": [], "remove_ids": [] },
        "items": { "upsert": [], "remove_ids": [] }
    }
}
```

**Response (409):**

```json
{ "message": "One or more orders are already assigned to another payer." }
```

**Response (422):** returned for a missing or ineligible `order_id`, self-selection, no active table session, or a customer outside the current table.

---

### 4.4.2 Release a Tablemate's Payment Assignment

**DELETE** `/api/customer/payments/pay-for/{orderId}`

Releases the authenticated customer's assignment for the selected unpaid order. `orderId` accepts the numeric ID or `order_public_id`. Successfully paid orders retain their payer identity for history and receipts. The operation is idempotent when no releasable order remains. Any unpaid side order merged back into the main order appears in `state_patch.orders.remove_ids`.

When at least one assignment is released, every customer with an active session at the table receives an `order_updated` notification containing the same `state_patch`.

**Authentication:** required (Bearer token).

**Request body:** none.

**Response (200):**

```json
{
    "message": "Payment assignment released.",
    "released_orders_count": 1,
    "state_patch": {
        "id": "019b10d8-45bf-7c9a-a541-c5881b87ee98",
        "version": 1784123456790000,
        "operation": "payment.assignment_released",
        "orders": { "upsert": [], "remove_ids": [] },
        "items": { "upsert": [], "remove_ids": [] }
    }
}
```

**Response (409):**

```json
{ "message": "Orders with an active payment cannot be released." }
```

**Response (422):** returned when the authenticated customer has no active table session.

---

### 4.5 Request Cash Payment

**POST** `/api/customer/payments/request-cash`

Requests a cash payment for every order payable by the authenticated customer in their active table visit: their own eligible unpaid orders plus any tablemate orders explicitly assigned to them through the pay-for flow. Creates an `order_payments` record with status `cash_requested` and notifies all customers at the table plus the waiter staff. The waiter manually confirms cash receipt via `PATCH /api/vendor/orders/{orderId}/confirm-cash`.

**Authentication:** required (Bearer token).

**Body:**

```json
{
    "notes": "I have a 50€ note, will need change"
}
```

| Field   | Type          | Required | Description                                         |
| ------- | ------------- | -------- | --------------------------------------------------- |
| `notes` | string\| null | No       | Free-text note shown to the waiter (max 500 chars). |

**Backend behavior:**

- Resolves the payer from the authenticated token; no `customer_id` is accepted or required.
- Resolves payable orders from the payer's active table session; no `order_id` is accepted or required.
- Rejects an owner attempting to pay an order assigned to someone else with HTTP `409`.
- Includes all confirmed-or-later, unpaid orders assigned to the payer in the same active table visit.
- Rejects if any session has unsubmitted cart items (HTTP `422`).
- Rejects if total amount is zero or negative (HTTP `422`).
- Creates one `order_payments` row with `status: 'cash_requested'`, `payment_method: 'cash'`, and `notes` in metadata.
- Updates each covered order: `payment_method = 'cash'`, `payment_pending = true`.
- Sends a `payment_updated` notification to all customers at the table and to waiter/vendor staff.
- Returns the same `payment.cash_requested` state patch sent to table customers so the initiator can lock the affected orders without reloading history.

**Response (200):**

```json
{
    "message": "Cash payment requested. A waiter will come to your table.",
    "amount": 42.5,
    "currency": "EUR",
    "state_patch": {
        "id": "019b10d9-58be-7f9d-858d-0db5e086aa72",
        "version": 1784123456791000,
        "operation": "payment.cash_requested",
        "orders": { "upsert": [], "remove_ids": [] },
        "items": { "upsert": [], "remove_ids": [] }
    }
}
```

**Response (409) — assigned to another payer or already paid:**

```json
{ "message": "This order is assigned to another payer." }
```

**Response (422) — already paid, unsubmitted items, or zero amount:**

```json
{ "message": "Order is already paid." }
```

---

### 4.6 Create Stripe Payment Intent

**POST** `/api/customer/payments/create-intent`

Creates one Stripe PaymentIntent covering every payable order in the authenticated customer's current table visit: the customer's own unpaid orders in their active table session, plus every unpaid order at the same table whose `paid_by` is the authenticated customer. This endpoint is for Stripe Elements / PaymentElement; the frontend stays in the app and uses the returned `clientSecret`.

**Authentication:** required (Bearer token).

**Body:** none. All covered orders are derived from the authenticated customer's active table session.

**Backend behavior:**

- Requires an active table scan session for the authenticated customer (HTTP `422` otherwise).
- Includes the customer's own confirmed-or-later, unpaid orders in that session, excluding orders another customer has claimed via pay-for.
- Includes all confirmed-or-later, unpaid orders at the same table assigned to the payer via pay-for.
- Returns HTTP `409` when the customer's only unpaid orders are assigned to another payer, and HTTP `422` when there is nothing to pay.
- Derives `table_session_id` from the covered orders; the frontend does not send it.
- Recalculates each covered order from bound and shared cart rows and charges their combined total.
- Requires the restaurant's `vendor_settings` to have Stripe enabled, a `stripe_account_id`, and completed onboarding.
- Creates a platform PaymentIntent with `transfer_data.destination` set to the vendor Stripe account ID.
- Stores one `order_payments` row and links every covered order through `order_payment_orders` for audit and webhook reconciliation.
- Returns the existing active PaymentIntent when the request is retried.
- Returns a `payment.initiated` state patch whose order rows have `payment_pending: true`; the same patch is sent in the table notification.

**PaymentIntent metadata:**

```json
{
    "order_id": "42",
    "order_public_id": "ord-aB3xK9pQrS12",
    "vendor_id": "1",
    "customer_id": "7",
    "table_session_id": "12",
    "payment_for": "dine_in",
    "covered_order_count": "3"
}
```

**Response (200):**

```json
{
    "clientSecret": "pi_123_secret_abc",
    "paymentIntentId": "pi_123",
    "state_patch": {
        "id": "019b10da-d2d5-7aed-ac19-11414c55c7b7",
        "version": 1784123456792000,
        "operation": "payment.initiated",
        "orders": { "upsert": [], "remove_ids": [] },
        "items": { "upsert": [], "remove_ids": [] }
    }
}
```

**Response (422) — already paid or Stripe unavailable:**

```json
{ "message": "Stripe payments are not enabled for this restaurant." }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 4.7 Update Stripe Payment Intent With Tip

**POST** `/api/customer/payments/update-intent`

Updates an existing Stripe PaymentIntent after the customer chooses a tip. For a grouped payment, the backend recalculates every linked order and stores the full tip on the payer-owned anchor order. A positive tip is rejected when the payment contains only other customers' orders.

**Authentication:** required (Bearer token).

**Body:**

```json
{
    "payment_intent_id": "pi_123_secret_abc",
    "tip_amount": 5.0
}
```

`payment_intent_id` may be either the PaymentIntent ID (`pi_123`) or the client secret (`pi_123_secret_abc`). Both fields are required; no other data is accepted.

**Backend behavior:**

- Resolves the payment by `payment_intent_id` scoped to the authenticated customer through `order_payments` (HTTP `404` when the intent belongs to another customer).
- Validates that the Stripe PaymentIntent metadata matches the stored payment.
- Stores `tip_amount` on the payer-owned anchor order.
- Keeps `orders.amount` as the order subtotal/payable amount before tip.
- Updates the Stripe PaymentIntent amount to the sum of all covered orders plus the tip.
- Updates `order_payments.amount` to the final charged amount including tip.
- Returns a `payment.updated` state patch for every covered order and includes the same patch in realtime notification metadata.

**Response (200):**

```json
{
    "clientSecret": "pi_123_secret_abc",
    "paymentIntentId": "pi_123",
    "state_patch": {
        "id": "019b10dc-17df-7971-ad78-f1a862fd3923",
        "version": 1784123456793000,
        "operation": "payment.updated",
        "orders": { "upsert": [], "remove_ids": [] },
        "items": { "upsert": [], "remove_ids": [] }
    }
}
```

**Response (422):**

```json
{ "message": "PaymentIntent cannot be updated in its current status." }
```

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

---

### 4.8 Verify Stripe Payment Intent

**GET** `/api/customer/payments/verify?payment_intent=pi_123`

Retrieves the PaymentIntent from Stripe, verifies it matches the authenticated customer's internal payment row, synchronizes every covered order, and returns the frontend-safe payment state.

**Authentication:** required (Bearer token).

**Language:** notification messages are rendered from admin-managed notification templates using `Accept-Language`. Missing templates fall back to English, then to the stored fallback message.

**Response (200):**

```json
{
    "status": "succeeded",
    "orderStatus": "paid",
    "state_patch": {
        "id": "019b10dd-808f-7de5-a068-1f8fbba8011c",
        "version": 1784123456794000,
        "operation": "payment.completed",
        "orders": { "upsert": [], "remove_ids": [] },
        "items": { "upsert": [], "remove_ids": [] }
    }
}
```

**Status values:**

| Field         | Values                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------ |
| `status`      | `requires_payment_method`, `requires_confirmation`, `requires_action`, `processing`, `succeeded`, `canceled` |
| `orderStatus` | `pending`, `paid`, `failed`                                                                                  |

**Mapping:**

- `succeeded` → `orderStatus: "paid"` and `orders.payment_received = true`.
- `processing`, `requires_action`, `requires_confirmation`, `requires_payment_method` → `orderStatus: "pending"`.
- `canceled` or failed webhook events → `orderStatus: "failed"`.
- The patch operation is `payment.completed` when this verification first observes a successful payment; otherwise it is `payment.verified`.

**Response (404):**

```json
{ "message": "No query results for model [App\\Models\\OrderPayment]." }
```

---

### 4.8.1 Cancel Active Stripe Payment Intent

**DELETE** `/api/customer/payments/intent`

Cancels the authenticated customer's non-processing Stripe PaymentIntent for the current active table, resets `payment_pending` on every covered unpaid order, and returns the authoritative unlock patch. A Stripe intent already in `processing` cannot be canceled.

**Authentication:** required (Bearer token).

**Request body:** none.

**Response (200):**

```json
{
    "canceled": true,
    "state_patch": {
        "id": "019b10de-35a8-7540-9258-42530b931761",
        "version": 1784123456795000,
        "operation": "payment.canceled",
        "orders": { "upsert": [], "remove_ids": [] },
        "items": { "upsert": [], "remove_ids": [] }
    }
}
```

When no active intent remains, the idempotent response is `canceled: false`; `state_patch` is still present with empty upsert/removal arrays.

**Response (409):**

```json
{ "message": "A payment is currently processing and cannot be canceled." }
```

---

### 4.9 Stripe Payment Webhook

**POST** `/api/customer/payments/webhook`

Receives Stripe PaymentIntent events and keeps `order_payments` and every order linked through `order_payment_orders` in sync.

**Authentication:** public route; Stripe signature required via `Stripe-Signature` header and `STRIPE_WEBHOOK_SECRET`.

**Handled events:**

- `payment_intent.succeeded`
- `payment_intent.payment_failed`
- `payment_intent.canceled`
- `payment_intent.processing`

**Response (200):**

```json
{ "received": true }
```

When the webhook changes customer-visible order state, the `payment_updated` realtime notification contains `metadata.state_patch` with operation `payment.completed`. The Stripe webhook response itself remains the acknowledgement shown above.

**Response (400) — invalid signature:**

```json
{ "message": "Invalid Stripe webhook signature." }
```

**Logging:** every webhook delivery attempt is logged to `stripe_webhook_logs` with event type, payment intent ID, HTTP status, outcome (`processed`, `signature_invalid`, `payment_not_found`, `ignored_event_type`), and any error message.

**Recovery:** a scheduled command `payments:reconcile-stale` runs every 5 minutes, polling Stripe directly for any `order_payments` rows stuck in a non-succeeded state for >10 minutes where the order is still `payment_pending = true`.

**Tables used:**

- `orders` stores current customer-facing payment flags and Stripe transaction ID.
- `order_payments` stores PaymentIntent audit/reconciliation data.
- `order_payment_orders` stores every order and pre-tip amount covered by a PaymentIntent.
- `stripe_webhook_logs` stores every webhook delivery attempt and its outcome.
- `table_scan_sessions` links dine-in orders to the customer session.
- `cart_items.order_id` links owned item rows to the order; `cart_items.shared_order_ids` links shared item rows to participant orders.
- `vendor_settings` provides the vendor Stripe Connect account ID.

---

### 4.10 Track Participant Order 🔒

**GET** `/api/customer/orders/{orderPublicId}/tracking`

Returns the authenticated participant's order tracking payload. The order must either belong to the customer directly/through the customer's `table_scan_session`, or be assigned to or covered by a payment made by that customer. Therefore both the participant who placed an order and the participant who paid for it may track it.

`can_view_receipt` is `true` only when the order is paid and the authenticated customer owns the completed payment (including legacy self-paid orders). Clients must not request the order receipt or show receipt actions when it is `false`.

**Authentication:** Bearer token with `auth:customer`.

**Request Body:** none

**Response (200):**

```json
{
    "id": 101,
    "order_public_id": "ORD-ABC123",
    "order_number": "1001",
    "status": "draft",
    "estimated_delivery_time": "27.11.2025 10:31",
    "total_amount": 25.99,
    "currency": "EUR",
    "order_type": "dine-in",
    "payment_method": null,
    "payment_pending": true,
    "payment_received": false,
    "can_view_receipt": false,
    "items": [
        {
            "cart_item_id": 1,
            "menu_item_id": 42,
            "name": "Fries",
            "image_url": null,
            "quantity": 2,
            "unit_price": 3.85,
            "line_total": 7.7,
            "vat_rate": 10,
            "tax_category": "food",
            "status": "Preparing",
            "notes": null,
            "paid_addons": [],
            "free_addons": [],
            "removed_items": [],
            "selected_modifiers": []
        },
        {
            "cart_item_id": 3,
            "menu_item_id": 51,
            "name": "Pizza",
            "image_url": null,
            "quantity": 1,
            "unit_price": 20.89,
            "line_total": 20.89,
            "vat_rate": 10,
            "tax_category": "food",
            "status": "Preparing",
            "notes": null,
            "paid_addons": [],
            "free_addons": [],
            "removed_items": [],
            "selected_modifiers": []
        }
    ],
    "shared_items": [
        {
            "cart_item_id": 1,
            "menu_item_id": 42,
            "name": "Fries",
            "image_url": null,
            "quantity": 2,
            "unit_price": 3.85,
            "line_total": 7.7,
            "vat_rate": 10,
            "tax_category": "food",
            "status": "Preparing",
            "notes": null,
            "paid_addons": [],
            "free_addons": [],
            "removed_items": [],
            "selected_modifiers": [],
            "shared_between": 2,
            "shared_with": [
                {
                    "order_id": 102,
                    "customer_id": 9,
                    "customer_name": "Bob Jones"
                }
            ],
            "my_share": 3.85
        },
        {
            "cart_item_id": 3,
            "menu_item_id": 51,
            "name": "Pizza",
            "image_url": null,
            "quantity": 1,
            "unit_price": 20.89,
            "line_total": 20.89,
            "vat_rate": 10,
            "tax_category": "food",
            "status": "Preparing",
            "notes": null,
            "paid_addons": [],
            "free_addons": [],
            "removed_items": [],
            "selected_modifiers": [],
            "shared_between": 2,
            "shared_with": [
                {
                    "order_id": 101,
                    "customer_id": 9,
                    "customer_name": "Bob Jones"
                }
            ],
            "my_share": 10.45
        }
    ],
    "tax_groups": [
        {
            "code": "A",
            "label": "Food (10%)",
            "tax_category": "food",
            "vat_rate": 10,
            "vat_amount": 2.6,
            "gross_amount": 28.59,
            "net_amount": 25.99
        }
    ],
    "totals": {
        "net_total": 25.99,
        "vat_total": 2.6,
        "service_fee": 0.0,
        "grand_total": 28.59
    }
}
```

**Item rules:**

- `items` contains only the cart items that the authenticated participant added to their own cart.
- `shared_items` is empty by default.
- `shared_items` contains an owned item only when that participant shares it with another order.
- `shared_items` contains another participant's item when that item is shared with the authenticated participant's order.
- `status` is derived from the cart item timestamps: `Served`, `Ready`, `Preparing`, or `null`.
- `estimated_delivery_time` is computed from the order creation time plus the vendor's `estimated_prep_time` setting.
- `estimated_delivery_time` uses the vendor's saved date/time formats.

**Response (401):**

```json
{ "message": "Unauthenticated." }
```

**Response (404):**

```json
{ "message": "No query results for model [App\\Models\\Order]." }
```

---

### 4.11 Notifications 🔒

#### 4.10.1 List Notifications

**GET** `/api/customer/notifications`

Returns the authenticated customer's most recent notifications (up to 50) and an unread count. Notifications are created automatically when cart items or order data changes — both from customer actions (adding/removing items, confirming orders) and vendor actions (marking orders ready, served, cancelled, confirming payments, updating item statuses).

**Authentication:** required (Bearer token).

**Response (200):**

```json
{
    "notifications": [
        {
            "id": 1,
            "event": "order_updated",
            "message": "Your order is ready!",
            "read": false,
            "created_at": "12.06.2026 14:30"
        },
        {
            "id": 2,
            "event": "cart_updated",
            "message": "Alice added Fries to the cart.",
            "read": true,
            "created_at": "12.06.2026 14:25"
        }
    ],
    "unread_count": 1
}
```

**Event types:**

| Event               | Trigger                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------- |
| `cart_updated`      | Cart item added, updated, or removed by any customer at the table                           |
| `cart_item_updated` | Individual cart item status changed by vendor (preparing, ready, served)                    |
| `order_updated`     | Customer/waiter order created or updated, confirmed, ready, served, picked up, or cancelled |
| `payment_updated`   | Payment assignment claimed/released, payment initiated/completed, or cash payment confirmed |
| `participant_added` | New customer joined the table session                                                       |
| `session_expire`    | Table session closed                                                                        |

`created_at` uses the related vendor's saved date/time formats. New restaurant notifications retain `vendor_id` so the correct format can be resolved.

#### 4.10.2 Mark Notification as Read

**PATCH** `/api/customer/notifications/{id}/read`

Marks a single notification as read.

**Authentication:** required (Bearer token).

**Response (200):**

```json
{ "message": "Notification marked as read." }
```

**Response (404):**

```json
{ "message": "Notification not found." }
```

#### 4.10.3 Mark All Notifications as Read

**POST** `/api/customer/notifications/read-all`

Marks all unread notifications for the authenticated customer as read.

**Authentication:** required (Bearer token).

**Response (200):**

```json
{ "message": "All notifications marked as read." }
```

---

## 5. Reservations 🔒

### 5.1 List Reservations

**GET** `/api/customer/reservations`

**Query Parameters:**

| Param      | Type   | Description                                                    |
| ---------- | ------ | -------------------------------------------------------------- |
| `tab`      | string | `upcoming`, `pending`, `past`, `cancelled` (default: upcoming) |
| `per_page` | int    | Items per page (default: 20)                                   |

---

### 5.2 Create Reservation

**POST** `/api/customer/reservations`

**Body:**

```json
{
    "vendor_public_id": "V-ABC123",
    "date": "2026-04-15",
    "time": "19:00",
    "party_size": 4,
    "customer_note": "Window seat please"
}
```

**Response (201):** Reservation with status `pending`.

The request uses canonical `YYYY-MM-DD` and `HH:mm` values. Returned reservation `date`, `time`, `created_at`, and `updated_at` fields use the selected vendor's saved formats.

---

### 5.3 Show Reservation

**GET** `/api/customer/reservations/{reservationPublicId}`

---

### 5.4 Cancel Reservation

**POST** `/api/customer/reservations/{reservationPublicId}/cancel`

Cancels a `pending` or `confirmed` reservation.

**Body:**

```json
{}
```

---

## 6. Loyalty Points 🔒

### 6.1 List Loyalty Wallets

**GET** `/api/customer/loyalty`

Returns one wallet per restaurant with points balance, progress info, and redeemability.

**Response (200):**

```json
[
    {
        "vendor": { "vendor_public_id": "...", "restaurant_name": "..." },
        "logo_url": "...",
        "points_balance": 150,
        "total_earned": 200,
        "total_redeemed": 50,
        "next_reward_at": 200,
        "points_to_next_reward": 50,
        "reward_value_eur": 2.0,
        "is_redeemable": false
    }
]
```

---

### 6.2 Loyalty Wallet Detail

**GET** `/api/customer/loyalty/{vendorPublicId}`

Returns wallet details and paginated transaction history.

Transaction `created_at` and `updated_at` fields use the selected vendor's saved date/time formats.

---

## 7. Favorites 🔒

Favorite restaurants are stored per customer in the `customer_favorites` pivot table. The `is_favorite` flag on restaurant browsing responses (3.2 List Restaurants, 3.3 Restaurant Profile, 3.8 Restaurant About) is derived from this list — it is `true` when the request carries a valid customer Bearer token and the restaurant is in the customer's favorites, and `false` otherwise (including unauthenticated requests).

### 7.1 List Favorites

**GET** `/api/customer/favorites`

Returns the customer's favorite restaurants.

**Response (200):**

```json
[
    {
        "id": 1,
        "vendor_public_id": "V-ABC123",
        "restaurant_name": "Buffalo Burger",
        "slug": "buffalo-burger",
        "city": "Vienna",
        "address": "Herrengasse 14",
        "logo_url": "http://localhost:8000/media/vendors/1/logo/abc.png",
        "cover_photo_url": "http://localhost:8000/media/vendors/1/cover/def.png",
        "avg_rating": 4.2,
        "review_count": 890,
        "is_open": true,
        "status": "Open",
        "business_hours": {
            "monday": { "open": "11:00", "close": "22:00", "closed": false },
            "tuesday": { "open": "11:00", "close": "22:00", "closed": false }
        },
        "cuisines": ["Burgers", "Fast food"]
    }
]
```

**Notes:**

- `avg_rating` is rounded to 1 decimal (0 if no reviews). `review_count` is the total number of reviews.
- `is_open` is computed from the vendor's `business_hours` for the current day/time, evaluated in the vendor's timezone. Windows that span midnight (e.g. `18:00`–`02:00`) are handled correctly, including in the early-morning hours after midnight.
- `status` is the human-readable string `"Open"` or `"Closed"`, mirroring `is_open`.
- `business_hours` is the vendor's configured weekly business-hours map, or `null` if unavailable.
- Opening and closing values use the vendor's saved time format.
- `cuisines` is derived from the restaurant's active menu categories.

---

### 7.2 Add Favorite

**POST** `/api/customer/favorites/{vendorPublicId}/add` 🔒

Adds the given restaurant to the authenticated customer's favorites. The restaurant is identified by the `vendorPublicId` URL segment — no request body is required.

**Path Parameters:**

- `vendorPublicId` — the restaurant's `vendor_public_id` (e.g. `V-ABC123`).

**Body:**

```json
{}
```

**Response (201):**

```json
{ "message": "Restaurant added to favorites." }
```

**Response (404):**

```json
{ "message": "Not found." }
```

Idempotent — re-adding an existing favorite returns `201` without creating a duplicate row.

---

### 7.3 Remove Favorite

**DELETE** `/api/customer/favorites/{vendorPublicId}/delete` 🔒

Removes the given restaurant from the authenticated customer's favorites.

**Path Parameters:**

- `vendorPublicId` — the restaurant's `vendor_public_id`.

**Body:**

```json
{}
```

**Response (200):**

```json
{ "message": "Restaurant removed from favorites." }
```

**Response (404):**

```json
{ "message": "Not found." }
```

---

## 8. Reviews 🔒

### 8.1 List Reviews

**GET** `/api/customer/reviews`

**Query Parameters:** `per_page`, `page`

Review `created_at`, `updated_at`, and `vendor_replied_at` fields use each review vendor's saved date/time formats. Each review includes its per-item ratings in the `items` array.

**Response (200):**

```json
{
    "data": [
        {
            "review_public_id": "rev_abc123...",
            "session_scan_table_id": 81,
            "rating": 5,
            "text": "Amazing food and service!",
            "photos": [
                "http://localhost:8000/media/reviews/rev_abc123/photos/table.jpg"
            ],
            "vendor": {
                "vendor_public_id": "V-ABC123",
                "restaurant_name": "Bella Italia"
            },
            "items": [
                {
                    "cart_item_id": 1,
                    "menu_item_id": 42,
                    "menu_item_name": "Margherita Pizza",
                    "menu_item_image": "http://localhost:8000/media/menu-items/42/photo.jpg",
                    "rating": 5,
                    "text": "Best pizza I've ever had",
                    "photos": [
                        "http://localhost:8000/media/reviews/rev_abc123/items/1/pizza.jpg"
                    ]
                }
            ],
            "vendor_reply": null,
            "vendor_replied_at": null,
            "flagged": false,
            "created_at": "06/21/2026 2:30 PM",
            "updated_at": "06/21/2026 2:30 PM"
        }
    ]
}
```

---

### 8.2 List Reviews for a Menu Item

**GET** `/api/customer/reviews/item/{menuItemId}`

**Authentication:** required (`auth:customer`).

Returns only the item-level review entries for the requested menu item. The parent session's overall rating, text, photos, and reviews for other items are not returned. Flagged session reviews are excluded.

**Query Parameters:** `per_page` (default `20`, maximum `100`), `page`

**Request body:** none.

**Response (200):**

```json
{
    "data": [
        {
            "review_public_id": "rev_abc123",
            "session_scan_table_id": 81,
            "cart_item_id": 301,
            "rating": 5,
            "text": "Best pizza I've ever had",
            "photos": [
                "http://localhost:8000/media/reviews/rev_abc123/items/301/pizza.jpg"
            ],
            "reviewer": {
                "name": "Jane Customer",
                "profile_picture": null
            },
            "created_at": "06/21/2026 2:30 PM",
            "updated_at": "06/21/2026 2:30 PM"
        }
    ],
    "menu_item": {
        "id": 42,
        "name": "Margherita Pizza",
        "image_url": "http://localhost:8000/media/menu-items/42/photo.jpg"
    },
    "review_summary": {
        "average_rating": 5,
        "total_reviews": 1,
        "rating_breakdown": [
            { "star": 5, "count": 1, "percent": 100 },
            { "star": 4, "count": 0, "percent": 0 },
            { "star": 3, "count": 0, "percent": 0 },
            { "star": 2, "count": 0, "percent": 0 },
            { "star": 1, "count": 0, "percent": 0 }
        ]
    }
}
```

**Response (404):** The menu item does not exist, is inactive, or belongs to a restaurant that is not live and discoverable.

---

### 8.3 List All Sessions & Review Status

**GET** `/api/customer/reviews/sessions`

**Authentication:** required (`auth:customer`).

**Request body:** none.

Returns all table scan sessions for the authenticated customer that have at least one order, along with review eligibility flags and full review details when a review exists. Sessions with no orders are excluded.

**Response (200):**

```json
{
    "data": [
        {
            "session_scan_table_id": 81,
            "reviewed": true,
            "reviewable": false,
            "all_paid": true,
            "all_served": true,
            "orders": [
                {
                    "order_id": "ord-aB3xK9pQrS12",
                    "total_amount_paid": 27.5,
                    "items": [{ "cart_item_id": 1 }, { "cart_item_id": 2 }]
                }
            ],
            "review": {
                "review_public_id": "rev-xYz123",
                "session_scan_table_id": 81,
                "rating": 5,
                "text": "Amazing food and service!",
                "photos": ["https://.../photo1.jpg"],
                "vendor": {
                    "vendor_public_id": "vnd-aBcDeF",
                    "restaurant_name": "Tavlo Kitchen"
                },
                "items": [
                    {
                        "cart_item_id": 1,
                        "menu_item_id": 10,
                        "menu_item_name": "Margherita Pizza",
                        "menu_item_image": "https://.../pizza.jpg",
                        "rating": 5,
                        "text": "Best pizza I've ever had",
                        "photos": ["https://.../item_photo.jpg"]
                    }
                ],
                "vendor_reply": null,
                "vendor_replied_at": null,
                "flagged": false,
                "created_at": "2026-07-04 19:30:00",
                "updated_at": "2026-07-04 19:30:00"
            }
        },
        {
            "session_scan_table_id": 72,
            "reviewed": false,
            "reviewable": true,
            "all_paid": true,
            "all_served": true,
            "orders": [
                {
                    "order_id": "ord-kL9mN2pQrT45",
                    "total_amount_paid": 15.0,
                    "items": [{ "cart_item_id": 5 }]
                }
            ]
        }
    ]
}
```

The `review` object is only present when `reviewed` is `true`.

---

### 8.4 Get Session Orders & Review Status

**GET** `/api/customer/reviews/session/{sessionScanTableId}`

**Authentication:** required (`auth:customer`).

**Request body:** none.

Returns every order and cart item for the table scan session, along with review eligibility flags. If a review has already been submitted, the full review details are included so the client can prefill an edit form.

**Status flags:**

| Field        | Type    | Description                                                                                                                   |
| ------------ | ------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `reviewed`   | boolean | `true` if a review has already been submitted for this session.                                                               |
| `reviewable` | boolean | `true` only when all orders are paid, all items are served, **and** no review exists yet. `false` if a review already exists. |
| `all_paid`   | boolean | `true` if every order in the session has `payment_received = true`.                                                           |
| `all_served` | boolean | `true` if every cart item (including shared items) has `served_at` set.                                                       |

**Response (200) — no existing review:**

```json
{
    "session_scan_table_id": 81,
    "reviewed": false,
    "reviewable": true,
    "all_paid": true,
    "all_served": true,
    "orders": [
        {
            "order_id": "ord-aB3xK9pQrS12",
            "total_amount_paid": 27.5,
            "items": [{ "cart_item_id": 1 }, { "cart_item_id": 2 }]
        }
    ]
}
```

**Response (200) — review already submitted:**

```json
{
    "session_scan_table_id": 81,
    "reviewed": true,
    "reviewable": false,
    "all_paid": true,
    "all_served": true,
    "orders": [
        {
            "order_id": "ord-aB3xK9pQrS12",
            "total_amount_paid": 27.5,
            "items": [{ "cart_item_id": 1 }, { "cart_item_id": 2 }]
        }
    ],
    "review": {
        "review_public_id": "rev-xYz123",
        "session_scan_table_id": 81,
        "rating": 5,
        "text": "Amazing food and service!",
        "photos": ["https://.../photo1.jpg"],
        "vendor": {
            "vendor_public_id": "vnd-aBcDeF",
            "restaurant_name": "Tavlo Kitchen"
        },
        "items": [
            {
                "cart_item_id": 1,
                "menu_item_id": 10,
                "menu_item_name": "Margherita Pizza",
                "menu_item_image": "https://.../pizza.jpg",
                "rating": 5,
                "text": "Best pizza I've ever had",
                "photos": ["https://.../item_photo.jpg"]
            }
        ],
        "vendor_reply": null,
        "vendor_replied_at": null,
        "flagged": false,
        "created_at": "2026-07-04 19:30:00",
        "updated_at": "2026-07-04 19:30:00"
    }
}
```

`total_amount_paid` includes the order amount and its tip.

**Response (422):**

The response uses `errors.session_scan_table_id` with one of these messages:

- `This session has no orders.`

---

### 8.5 Get Reviewable Sessions for a Vendor

**GET** `/api/customer/reviews/vendor/{vendorPublicId}/reviewable`

**Authentication:** required (`auth:customer`).

**Request body:** none.

Returns the list of table scan sessions at the given vendor that the authenticated customer can still review. A session is included only when:

- The customer has at least one non-cancelled/non-draft order in the session.
- All active orders are paid (`payment_received = true`).
- All active orders are served or picked up.
- No review has been submitted for the session yet.

Each session includes the served items so the client can show what the customer ordered.

**Response (200):**

```json
{
    "vendor": {
        "vendor_public_id": "vnd-aBcDeF",
        "restaurant_name": "Tavlo Kitchen"
    },
    "data": [
        {
            "session_scan_table_id": 81,
            "scanned_at": "2026-07-04 14:30:00",
            "items": [
                {
                    "cart_item_id": 1,
                    "menu_item_id": 10,
                    "menu_item_name": "Margherita Pizza",
                    "menu_item_image": "https://.../pizza.jpg",
                    "quantity": 2
                },
                {
                    "cart_item_id": 2,
                    "menu_item_id": 15,
                    "menu_item_name": "Caesar Salad",
                    "menu_item_image": "https://.../salad.jpg",
                    "quantity": 1
                }
            ]
        }
    ]
}
```

Returns an empty `data` array when the customer has no reviewable sessions at the vendor.

**Response (404):** Vendor not found.

---

### 8.6 Get Review Eligibility for a Vendor

**GET** `/api/customer/reviews/vendor/{vendorPublicId}/eligibility`

**Authentication:** required (`auth:customer`).

**Request body:** none.

Reports which kinds of reviews the vendor currently allows, based on the vendor's **Reviews** settings. The client uses these flags to decide whether to surface the restaurant-review, per-item-review, and anonymous-review options.

| Field | Type | Description |
|-------|------|-------------|
| `enableReviews` | boolean | Whether customers may rate and review the restaurant. |
| `enableMenuReviews` | boolean | Whether customers may rate and review each ordered item. |
| `allowAnonymousReviews` | boolean | Whether unlogged guests may rate and review. |

**Response (200):**

```json
{
    "enableReviews": true,
    "enableMenuReviews": false,
    "allowAnonymousReviews": false
}
```

**Response (404):** Vendor not found.

---

### 8.7 Get Session Items by Order

**GET** `/api/customer/reviews/order/{orderPublicId}/session`

**Authentication:** required (`auth:customer`).

**Request body:** none.

Given an order ID, returns the parent table scan session and **all** items the customer was served across every order in that session — not just the items from the given order. This lets the client show the full session context for review submission when the entry point is a single order.

Includes review eligibility flags identical to the session detail endpoint.

**Response (200):**

```json
{
    "session_scan_table_id": 81,
    "reviewable": true,
    "all_paid": true,
    "all_served": true,
    "reviewed": false,
    "items": [
        {
            "cart_item_id": 1,
            "menu_item_id": 10,
            "menu_item_name": "Margherita Pizza",
            "menu_item_image": "https://.../pizza.jpg",
            "quantity": 2
        },
        {
            "cart_item_id": 2,
            "menu_item_id": 15,
            "menu_item_name": "Caesar Salad",
            "menu_item_image": "https://.../salad.jpg",
            "quantity": 1
        }
    ]
}
```

Items from all of the customer's orders in that session are included (e.g. if the customer placed two separate orders during the same table session, items from both appear).

**Response (404):** Order not found or does not belong to the authenticated customer.

**Response (422):**

The response uses `errors.order` with one of these messages:

- `This order is not linked to a table session.`

---

### 8.8 Create Review

**POST** `/api/customer/reviews`

**Authentication:** required (`auth:customer`).

**Content-Type:** `multipart/form-data`.

Reviews are per table scan session. The customer provides one overall session review and can rate cart items from any paid order in that session.

**Conceptual request:**

```json
{
    "session_scan_table_id": 81,
    "rating": 5,
    "review": "Amazing food and service!",
    "photos": ["<image file>"],
    "items": [
        {
            "cart_item_id": 1,
            "rating": 5,
            "review": "Best pizza I've ever had",
            "photos": ["<image file>"]
        }
    ]
}
```

For multipart clients, send nested fields such as `photos[]`, `items[0][cart_item_id]`, `items[0][rating]`, `items[0][review]`, and `items[0][photos][]`.

**Validation:**

- `session_scan_table_id`: required, integer, must identify the authenticated customer's session
- `rating`: required, integer, 1–5 (overall session rating)
- `review`: nullable, string, max 2000
- `photos`: nullable, maximum 5 JPG, JPEG, PNG, or WebP files; maximum 5 MB each
- `items`: required, array, min 1 item
- `items.*.cart_item_id`: required, distinct integer, must belong to an order in the session
- `items.*.rating`: required, integer, 1–5
- `items.*.review`: nullable, string, max 1000
- `items.*.photos`: nullable, maximum 5 JPG, JPEG, PNG, or WebP files per item; maximum 5 MB each

**Eligibility:**

- The same session-wide checks described in §8.2 are performed again when the review is submitted.
- Only one review can be created per table scan session.

**Side effects:**

- Each reviewed menu item's `rating` and `review_count` are recalculated from all `review_items` for that menu item.

**Response (201):**

```json
{
    "message": "Review submitted.",
    "review": {
        "review_public_id": "rev_abc123...",
        "session_scan_table_id": 81,
        "rating": 5,
        "text": "Amazing food and service!",
        "photos": [
            "http://localhost:8000/media/reviews/rev_abc123/photos/table.jpg"
        ],
        "items": [
            {
                "cart_item_id": 1,
                "menu_item_id": 42,
                "menu_item_name": "Margherita Pizza",
                "menu_item_image": "http://localhost:8000/media/menu-items/42/photo.jpg",
                "rating": 5,
                "text": "Best pizza I've ever had",
                "photos": [
                    "http://localhost:8000/media/reviews/rev_abc123/items/1/pizza.jpg"
                ]
            }
        ]
    }
}
```

---

### 8.9 Update Review

**PATCH** `/api/customer/reviews/{reviewPublicId}`

**Authentication:** required (`auth:customer`).

**Content-Type:** `multipart/form-data`.

Can update the overall rating/text/photos and optionally update per-item ratings/photos. New photos are appended to existing ones; use `remove_photos` / `items[N][remove_photos]` to delete specific photos first (pass the stored path returned in the review's `photos` array).

Each review and each item may have at most **5 photos**. If the combined count (existing − removed + new) exceeds 5, a `422` is returned.

**Conceptual request:**

```json
{
    "rating": 4,
    "text": "Updated review text",
    "photos": ["<image file>"],
    "remove_photos": ["reviews/rev_abc123/photos/old.jpg"],
    "items": [
        {
            "cart_item_id": 1,
            "rating": 4,
            "review": "Updated item review",
            "photos": ["<image file>"],
            "remove_photos": ["reviews/rev_abc123/items/1/old.jpg"]
        }
    ]
}
```

**Multipart encoding** — identical to the create endpoint (section 8.7):

| Field pattern | Type | Description |
|---|---|---|
| `rating` | integer | Overall rating 1-5. |
| `text` | string | Overall review text (max 2000 chars). |
| `photos[]` | file | New overall review photos (jpg/jpeg/png/webp, max 5 MB each). |
| `remove_photos[]` | string | Stored paths of overall photos to delete. |
| `items[0][cart_item_id]` | integer | Cart item ID. |
| `items[0][rating]` | integer | Item rating 1-5. |
| `items[0][review]` | string | Item review text (max 1000 chars). |
| `items[0][photos][]` | file | New item photos. |
| `items[0][remove_photos][]` | string | Stored paths of item photos to delete. |

All fields are optional. If `items` is provided, each item must belong to an order in the review's session and is upserted by `cart_item_id`. Menu item ratings are recalculated.

**Response (200):**

```json
{
    "message": "Review updated.",
    "review": {
        "review_public_id": "rev_xYz123",
        "session_scan_table_id": 81,
        "rating": 4,
        "text": "Updated review text",
        "photos": ["https://.../photo_new.jpg"],
        "vendor": {
            "vendor_public_id": "vnd-aBcDeF",
            "restaurant_name": "Tavlo Kitchen"
        },
        "items": [
            {
                "cart_item_id": 1,
                "menu_item_id": 10,
                "menu_item_name": "Margherita Pizza",
                "menu_item_image": "https://.../pizza.jpg",
                "rating": 4,
                "text": "Updated item review",
                "photos": ["https://.../item_photo.jpg"]
            }
        ],
        "vendor_reply": null,
        "vendor_replied_at": null,
        "flagged": false,
        "created_at": "2026-07-04 19:30:00",
        "updated_at": "2026-07-04 20:15:00"
    }
}
```

**Response (422):**

- `photos` → `A review may have at most 5 photos.`
- `items` → `Item {id} may have at most 5 photos.`
- `items` → `One or more item IDs do not belong to this session.`

---

### 8.10 Delete Review

**DELETE** `/api/customer/reviews/{reviewPublicId}`

Deletes the review, uploaded review photos, associated item reviews, and item photos. Menu item ratings are recalculated.

---

## 9. Privacy & Data 🔒

### 9.1 Request Data Export

**POST** `/api/customer/privacy/export`

Submits a GDPR data export request. Customer will receive a download link via email.

**Body:**

```json
{}
```

**Response (201):**

```json
{
    "message": "Data export request submitted. You will receive a download link via email."
}
```

---

### 9.2 Request Account Deletion

**POST** `/api/customer/privacy/delete`

**Body:**

```json
{
    "password": "current-password",
    "confirmation": true
}
```

**Response (201):**

```json
{
    "message": "Account deletion request submitted. Your account will be permanently deleted after processing."
}
```

---

### 9.3 List GDPR Requests

**GET** `/api/customer/privacy/requests`

Returns all data export and deletion requests for the customer.

---

## Error Responses

All endpoints return standard JSON error responses:

**401 Unauthorized:**

```json
{
    "message": "Unauthenticated."
}
```

**404 Not Found:**

```json
{
    "message": "Not found."
}
```

**422 Validation Error:**

```json
{
    "message": "The given data was invalid.",
    "errors": {
        "field": ["Error message"]
    }
}
```
