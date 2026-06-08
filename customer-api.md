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

Order-level responses include a `tax_groups` array that groups all items by tax category:

| Field | Type | Description |
|-------|------|-------------|
| `code` | string | Letter code (A, B, C...) for receipt display |
| `label` | string | Human-readable label, e.g. "Food (10%)" |
| `tax_category` | string | Tax category slug |
| `vat_rate` | float | VAT percentage |
| `vat_amount` | float | Total VAT for this group |
| `gross_amount` | float | Total gross (VAT-inclusive) for this group |
| `net_amount` | float | Total net (VAT-exclusive) for this group |

### `totals` Object

| Field | Type | Description |
|-------|------|-------------|
| `net_total` | float | Sum of all `net_amount` across tax groups |
| `vat_total` | float | Sum of all `vat_amount` across tax groups |
| `service_fee` | float | `gross_total × service_fee_rate%` (from vendor settings) |
| `grand_total` | float | `gross_total + service_fee` |
| `currency` | string | Only present in order detail responses |

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
    "registration_source": "email"
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
- If email exists but no social link → links social account to existing customer
- If neither exists → creates new customer
- Returns `422` with `access_token` error if the token is invalid or expired

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

| Field | Type | Notes |
|---|---|---|
| `first_name` | string | Optional |
| `last_name` | string | Optional |
| `gender` | string | Optional: `male`, `female`, `other`, `prefer_not_to_say` |
| `date_of_birth` | date | Optional, before today |
| `address` | string | Optional |
| `profile_picture` | file | Optional image file. Backend stores it and returns the generated public URL. |

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
  { "id": 1, "name": "Burger", "slug": "burger", "icon": "http://localhost:8000/media/categories/burger.png" },
  { "id": 2, "name": "Pizza", "slug": "pizza", "icon": "http://localhost:8000/media/categories/pizza.png" },
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
| Param | Type | Description |
|-------|------|-------------|
| `search` | string | Search by restaurant name or city |
| `city` | string | Filter by city |
| `cuisine` | int | Filter by menu category ID |
| `price_range` | int | Price bracket: `1` = €0–10, `2` = €10–25, `3` = €25–50, `4` = €50+ |
| `service_type` | string | `dine_in`, `takeaway`, or `reservation` |
| `rating` | float | Minimum average rating (e.g. `4`) |
| `distance` | float | Max distance in km (requires `latitude` and `longitude`) |
| `latitude` | float | Customer's current latitude |
| `longitude` | float | Customer's current longitude |
| `sort_by` | string | `name` (default), `distance` (requires `latitude`/`longitude`), `rating` |
| `per_page` | int | Items per page (default: 20) |
| `page` | int | Page number |

**Response (200):**
```json
{
  "data": [
    {
      "vendor_public_id": "V-ABC123",
      "slug": "buffalo-burger",
      "restaurant_name": "Buffalo Burger",
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
        "monday":    { "open": "10:45", "close": "20:45", "closed": false },
        "tuesday":   { "open": "10:45", "close": "20:45", "closed": false },
        "wednesday": { "open": "10:45", "close": "20:45", "closed": false },
        "thursday":  { "open": "10:45", "close": "20:45", "closed": false },
        "friday":    { "open": "10:45", "close": "22:00", "closed": false },
        "saturday":  { "open": "11:00", "close": "22:00", "closed": false },
        "sunday":    { "closed": true }
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
- `cuisines` is derived from the restaurant's active menu categories.
- `price_label` is computed from the average menu item price: `Budget-friendly` (≤€10), `Mid-range` (€10–25), `Fine dining` (€25–50), `Premium` (€50+). `null` if no menu items.
- `latitude` / `longitude` are the restaurant's coordinates (may be `null` if not set by the vendor).
- `is_open` is computed from `business_hours` for the current day/time. `today_hours` shows today's open–close range, or `null` if closed today.
- `business_hours` is a per-day map. Each day has either `{ "open": "HH:MM", "close": "HH:MM", "closed": false }` or `{ "closed": true }`.
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
| Param | Type | Description |
|-------|------|-------------|
| `latitude` | float | Customer's current latitude (for distance) |
| `longitude` | float | Customer's current longitude (for distance) |

**Response (200):**
```json
{
  "vendor_public_id": "V-ABC123",
  "slug": "buffalo-burger",
  "restaurant_name": "Buffalo Burger",
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
    "monday":    { "open": "10:00", "close": "22:00", "closed": false },
    "tuesday":   { "open": "10:00", "close": "22:00", "closed": false },
    "wednesday": { "open": "10:00", "close": "22:00", "closed": false },
    "thursday":  { "open": "10:00", "close": "22:00", "closed": false },
    "friday":    { "open": "10:00", "close": "23:00", "closed": false },
    "saturday":  { "open": "11:00", "close": "23:00", "closed": false },
    "sunday":    { "open": "11:00", "close": "21:00", "closed": false }
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
- `is_open` is computed from the vendor's `business_hours` for the current day/time.
- `today_hours` shows today's open–close range, or `null` if closed today.
- `distance_km` is only returned when `latitude` and `longitude` are provided.
- `cuisines` is derived from the restaurant's active menu categories.
- `is_favorite` is `true` when the request is authenticated and the customer has favorited this restaurant; `false` otherwise (including for unauthenticated requests). Not shown above — also returned in the response payload.

---

### 3.4 Get Restaurant Categories

**GET** `/api/customer/restaurants/{vendorPublicId}/categories`

**Response (200):**
```json
[
  { "id": 1, "name": "Starters", "slug": "starters", "icon": "http://localhost:8000/media/categories/starters.png", "sort_order": 1 },
  { "id": 2, "name": "Mains", "slug": "mains", "icon": null, "sort_order": 2 }
]
```

**Notes:**
- `icon` is the absolute URL of the category icon image, or `null` if no icon is set.

---

### 3.5 Get Restaurant Menu

**GET** `/api/customer/restaurants/{vendorPublicId}/menu`

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `category_id` | int | Filter by category (optional) |
| `search` | string | Search by item name (optional) |

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
    "fat": 32.50,
    "carbs": 45.00,
    "protein": 38.00,
    "dietary_preference": null,
    "paid_addons": [
      { "name": "Extra cheese", "price": 1.65, "vat_rate": 10 }
    ],
    "free_addons": ["Ketchup"],
    "removable_items": ["Onions"],
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
          { "id": 1, "name": "Fries", "price_adjustment": 0.00 },
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
- `paid_addons` include `vat_rate` per add-on. Prices are gross.
- `modifier_groups` include `vat_rate` at the group level. Option `price_adjustment` values are gross.
- `free_addons` and `removable_items` are legacy menu-item customization options configured directly on the item.
- `modifier_groups` are reusable vendor-defined option groups attached to the item. Only active groups and options are returned.

---

### 3.6 Get Menu Item Detail

**GET** `/api/customer/restaurants/{vendorPublicId}/menu/{itemId}`

**Response (200):**
```json
{
  "id": 42,
  "name": "4 Piece chicken Box",
  "description": "4 Piece of hand-breaded original chicken with our special sauce and coleslaw.",
  "image_url": "http://localhost:8000/media/menu-items/42/photo/abc123.jpg",
  "price": 20.89,
  "has_discount": true,
  "discount_percent": 15.00,
  "discounted_price": 17.75,
  "vat_rate": 10,
  "tax_category": "food",
  "available": true,
  "rating": 4.4,
  "review_count": 252,
  "ordered_count": 1200,
  "calories": 680,
  "fat": 32.50,
  "carbs": 45.00,
  "protein": 38.00,
  "dietary_preference": null,
  "paid_addons": [
    { "name": "Extra cheese", "price": 1.65, "vat_rate": 10 }
  ],
  "free_addons": ["Ketchup"],
  "removable_items": ["Onions"],
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
        { "id": 1, "name": "Fries", "price_adjustment": 0.00 },
        { "id": 2, "name": "Onion Rings", "price_adjustment": 1.65 },
        { "id": 3, "name": "Sweet Potato Fries", "price_adjustment": 2.20 }
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
- `dietary_preference` can be `vegetarian`, `vegan`, `gluten_free`, etc. or `null`.
- `allergens` and `tags` include `icon` for UI display.
- `modifier_groups` shows customisation options — `type` is `single` or `multiple`, with `min_selection`/`max_selection` constraints. Group-level `vat_rate` applies to all options.
- `discount_percent` and `discounted_price` are only present when `has_discount` is `true`.
- Only active modifier groups and options are returned.

---

### 3.7 Get Restaurant Reviews

**GET** `/api/customer/restaurants/{vendorPublicId}/reviews`

Returns all public (non-flagged) reviews for a restaurant, with reviewer info and the menu item being reviewed (if any).

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `rating` | int | Filter by star rating (1–5) |
| `with_images` | bool | Only return reviews that include images |
| `sort_by` | string | `recent` (default), `highest`, `lowest` |
| `per_page` | int | Items per page (default: 20) |
| `page` | int | Page number |

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
      "created_at": "2026-04-18T14:32:10+00:00",
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
      "vendor_reply": "Thank you for the kind words!",
      "vendor_replied_at": "2026-04-19T08:10:00+00:00"
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
- `images` is an array of image URLs uploaded by the reviewer (may be empty).
- `reviewer.name` falls back to `"Anonymous"` if the customer has no name set.
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
    { "title": "Free Wi-Fi", "description": "High-speed wireless throughout the venue." },
    { "title": "Outdoor seating", "description": "Heated terrace open year-round." },
    { "title": "Parking", "description": "Free customer parking next door." },
    { "title": "Wheelchair accessible", "description": null },
    { "title": "Vegan options", "description": "Dedicated vegan menu section." }
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
    "monday":    { "open": "10:00", "close": "22:00", "closed": false },
    "tuesday":   { "open": "10:00", "close": "22:00", "closed": false },
    "wednesday": { "open": "10:00", "close": "22:00", "closed": false },
    "thursday":  { "open": "10:00", "close": "22:00", "closed": false },
    "friday":    { "open": "10:00", "close": "23:00", "closed": false },
    "saturday":  { "open": "11:00", "close": "23:00", "closed": false },
    "sunday":    { "closed": true }
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
[
  { "id": 1, "number": 1, "name": "Table 1" }
]
```

---

### 3.9.1 Get Restaurant Languages

**GET** `/api/customer/restaurants/{vendorPublicId}/languages`

Returns the restaurant's default menu language, customer-facing languages, date format, and time format enabled by the vendor in settings.

**Authentication:** public route; no Bearer token required.

**Request body:** none.

**Response (200):**
```json
{
  "vendor": { "id": "VID-8492", "name": "Bella Italia" },
  "default_language": "de",
  "available_languages": ["de", "en", "it"],
  "date_format": "DD.MM.YYYY",
  "time_format": "24h",
  "languages": [
    { "code": "de", "name": "Deutsch (German)", "is_default": true },
    { "code": "en", "name": "English", "is_default": false },
    { "code": "it", "name": "Italiano (Italian)", "is_default": false }
  ]
}
```

**Notes:**
- `default_language` comes from `vendor_settings.default_language`.
- `available_languages` comes from `vendor_settings.supported_languages`.
- `date_format` comes from `vendor_settings.date_format`.
- `time_format` comes from `vendor_settings.time_format`.
- The default language is always included first in `available_languages`, even if it is missing from `supported_languages`.

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
    "scannedAt": "2026-04-23T10:15:00+00:00"
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
    "scannedAt": "2026-04-23T10:15:00+00:00"
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
    "scannedAt": "2026-04-23T10:15:00+00:00"
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
    "scannedAt": "2026-04-23T10:17:00+00:00"
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
    "scannedAt": "2026-04-23T10:17:00+00:00"
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
    "scannedAt": "2026-04-23T10:17:00+00:00",
    "closedAt": "2026-04-23T11:42:00+00:00"
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

### 3.14 Call Waiter 🔒

**POST** `/api/customer/table/call`

Sends a notification to all active waiters at the restaurant. The customer must have an active table session.

**Authentication:** required (Bearer token).

**Body:** none.

**Rules:**
- Customer must have an active table session — otherwise blocked.
- Only team members with role `waiter` and status `active` are notified.
- If no active waiters exist at the restaurant → blocked.

**Response (200):**
```json
{ "message": "Waiters have been notified." }
```

**Response (422) — no active session:**
```json
{ "message": "You do not have an active table session." }
```

**Response (422) — no waiters available:**
```json
{ "message": "No waiters available at this restaurant." }
```

**Response (401):**
```json
{ "message": "Unauthenticated." }
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
          "price": 5.50,
          "paid_addons": [
            { "name": "Cheese sauce", "price": 1.65, "vat_rate": 10 }
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
          "vat_amount": 1.00,
          "line_total": 11.00,
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
          "vat_amount": 1.00,
          "gross_amount": 11.00,
          "net_amount": 10.00
        }
      ],
      "totals": {
        "net_total": 10.00,
        "vat_total": 1.00,
        "service_fee": 0.00,
        "grand_total": 11.00
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
        "net_total": 0.00,
        "vat_total": 0.00,
        "service_fee": 0.00,
        "grand_total": 0.00
      }
    }
  ]
}
```

**Notes:**
- `tax_groups` and `totals` are computed **per person** from that person's open cart items.

#### 3.14.2 Add Item

**POST** `/api/customer/cart/items`

Adds an item to the authenticated customer's cart. If the same `menu_item_id` already exists in the customer's current session, the quantity is incremented instead of creating a duplicate entry.

**Body:**
```json
{
  "menu_item_id": 42,
  "quantity": 2,
  "notes": "No salt",
  "paid_addons": [
    { "name": "Cheese sauce" }
  ],
  "free_addons": ["Ketchup"],
  "removed_items": ["Salt"],
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
- `paid_addons`: optional array of selected paid add-on objects. Each object must include `name`; submitted prices are ignored and replaced with the vendor-configured price for that menu item.
- `free_addons`: optional array of selected free add-on names.
- `removed_items`: optional array of selected removed item names. `removable_items` is still accepted for backward compatibility.
- `selected_modifiers`: optional array of selected modifier groups. Each entry accepts `modifier_group_id` and `option_ids`. `modifiers` is also accepted as an alias. Selected groups must be attached to the menu item, options must belong to the group, and required/min/max group rules are enforced.

**Behavior:**
- If a cart item with the same `menu_item_id` already exists for the customer's active `table_scan_session`, the existing item's quantity is incremented by the requested amount (default `1`). The existing item is returned.
- A cart item is only merged with an existing row when `menu_item_id`, `paid_addons`, `free_addons`, `removed_items`, and `selected_modifiers` all match. Different customization choices create separate cart rows.
- Selected customization options must exist on the menu item. Invalid selections return `422`.
- Paid add-on prices and selected modifier `price_adjustment` values are included in `price`, `line_total`, order draft totals, and payment totals.
- If no matching cart item exists, a new one is created.

**Response (201):**
```json
{
  "id": 1,
  "quantity": 2,
  "notes": "No salt",
  "price": 5.50,
  "paid_addons": [
    { "name": "Cheese sauce", "price": 1.65, "vat_rate": 10 }
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
  "vat_amount": 1.00,
  "line_total": 11.00,
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
      "total_price": 14.30,
      "items": [
        {
          "cart_item_id": 1,
          "menu_item_id": 42,
          "name": "Fries",
          "image_url": null,
          "quantity": 2,
          "unit_price": 3.85,
          "total_price": 7.70,
          "is_mine": true
        },
        {
          "cart_item_id": 2,
          "menu_item_id": 51,
          "name": "Coke",
          "image_url": null,
          "quantity": 1,
          "unit_price": 6.60,
          "total_price": 6.60,
          "is_mine": true
        }
      ],
      "tax_groups": [
        {
          "code": "A",
          "label": "Food (10%)",
          "tax_category": "food",
          "vat_rate": 10,
          "vat_amount": 1.30,
          "gross_amount": 14.30,
          "net_amount": 13.00
        }
      ],
      "totals": {
        "net_total": 13.00,
        "vat_total": 1.30,
        "service_fee": 0.00,
        "grand_total": 14.30
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
          "vat_amount": 1.90,
          "gross_amount": 20.89,
          "net_amount": 18.99
        }
      ],
      "totals": {
        "net_total": 18.99,
        "vat_total": 1.90,
        "service_fee": 0.00,
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

Creates a `draft` order for the authenticated customer's active table session. The amount is computed from the customer's currently open owned `cart_items` (`order_id = null`) plus any selected shared items. If the customer already has an unpaid draft for this active session, the draft amount is refreshed instead of creating another draft. If the latest unpaid order for this active session is already confirmed, this endpoint returns the table view without creating a new order; added open items are bound only when confirm is called again.

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
- `shared_item`: nullable integer. ID of a `cart_item` belonging to **another** customer at the same table. The caller's `order_id` is appended to that cart_item's `shared_order_ids` array (deduplicated). When the order is confirmed, this item contributes a share to the total amount.
- `unshared_item`: nullable integer. ID of a `cart_item` at the same table. The caller's `order_id` is removed from that cart_item's `shared_order_ids` array. If the caller was not sharing this item, the operation is a silent no-op.
- At least one of `shared_item` or `unshared_item` must be present.

**Response (200):** unified table-view payload — see [§4.1 Get Current Table History](#41-get-current-table-history) for the full shape.

**Response (404) — order not found:**
```json
{ "message": "Order not found." }
```

**Response (422) — neither field provided:**
```json
{ "message": "Provide shared_item or unshared_item." }
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

### 3.19 Create Order Confirmed 🔒

**POST** `/api/customer/table/order/confirmed`

Confirms the authenticated customer's open order for the active table session. **No request body is accepted.** The endpoint recomputes the final `amount`, updates the order to `status = "confirmed"`, and binds currently open owned cart rows to that order by setting `cart_items.order_id`.

**Authentication:** required (Bearer token).

**Body:**
```json
{}
```

**Total computation (from `cart_items`):**
1. **Owned items** — every already-bound row where `cart_items.order_id = O.id`, plus currently open rows for the caller's active session during confirmation, contributes `(unit_price × quantity) / (1 + count(shared_order_ids))`.
2. **Shared-into items** — every `cart_item` whose `shared_order_ids` JSON array contains this order's id (and whose session is *not* the caller's) contributes the same per-share amount.

The final amount is rounded to 2 decimals.

**Response (200):** unified table-view payload — see [§4.1 Get Current Table History](#41-get-current-table-history) for the full shape.

**Response (404) — no draft order:**
```json
{ "message": "No draft order found." }
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

This is the **canonical "table view" response** — it is also returned (with the same shape) by [§3.15](#315-create-order-draft-), [§3.16](#316-update-order-), and [§3.17](#317-create-order-confirmed-).

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
    "scanned_at": "2026-04-27T10:15:00+00:00"
  },
  "people": [
    {
      "session_id": 12,
      "customer_id": 7,
      "is_me": true,
      "name": "Alice Smith",
      "scanned_at": "2026-04-27T10:15:00+00:00",
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
          "currency": "EUR",
          "order_number": null,
          "order_type": "dine-in",
          "table_number": "3",
          "service_fee": 0.00,
          "vat_amount": 0.00,
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
          "created_at": "2026-04-27T10:30:00+00:00",
          "updated_at": "2026-04-27T10:31:00+00:00",
          "items": [
            {
              "cart_item_id": 1,
              "menu_item_id": 42,
              "name": "Fries",
              "image_url": null,
              "quantity": 2,
              "unit_price": 5.50,
              "paid_addons": [
                { "name": "Cheese sauce", "price": 1.65, "vat_rate": 10 }
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
              "tax_category": "food",
              "vat_amount": 1.00,
              "line_total": 11.00,
              "is_mine": true,
              "shared_between": 1,
              "shared_with": [],
              "my_share": 11.00,
              "status": "Preparing",
              "preparing_start_at": "2026-04-27T10:31:00+00:00",
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
                { "order_id": 101, "customer_id": 9, "customer_name": "Bob Jones" }
              ],
              "my_share": 10.45,
              "status": "Ready",
              "preparing_start_at": "2026-04-27T10:32:00+00:00",
              "ready_at": "2026-04-27T10:42:00+00:00",
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
          "vat_amount": 2.90,
          "gross_amount": 31.89,
          "net_amount": 28.99
        }
      ],
      "totals": {
        "net_total": 28.99,
        "vat_total": 2.90,
        "service_fee": 0.00,
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

| Field | Type | Description |
|------|------|-------------|
| `cart_item_id` | int | Source `cart_items.id`. |
| `menu_item_id` | int | The menu item this cart entry references. |
| `name`, `image_url` | string\|null | Cached from `menu_items`. |
| `quantity` | int | Cart quantity. |
| `unit_price` | float | Per-unit VAT-inclusive (gross) price, including selected paid add-ons and selected modifier price adjustments. |
| `paid_addons`, `free_addons`, `removed_items`, `selected_modifiers` | array | Selected customization options for this cart item. Paid add-ons and modifier options include `vat_rate` and gross prices. |
| `vat_rate` | float | VAT rate resolved from `tax_categories` table for the vendor's country and item's `tax_category`. |
| `tax_category` | string | Tax category slug (e.g. `food`, `beverage_non_alcoholic`, `beverage_alcoholic`). |
| `vat_amount` | float | VAT portion of `line_total`, computed as `line_total - (line_total / (1 + vat_rate/100))`. |
| `line_total` | float | `unit_price × quantity`. |
| `is_mine` | bool | `true` if the cart_item belongs to the caller's own session. |
| `shared_between` | int | `1 + count(shared_order_ids)`. The number of orders splitting this item (owner + sharers). |
| `shared_with` | object[] | The orders that share this item with the owner. Each entry: `order_id`, `customer_id`, `customer_name`. Empty array if unshared. |
| `my_share` | float | `line_total / shared_between` — what each participating order contributes. |
| `status` | string\|null | Per-item status derived from timestamps: `Served` when `served_at` is set, `Ready` when `ready_at` is set, `Preparing` when `preparing_start_at` is set, otherwise `null`. |
| `preparing_start_at`, `ready_at`, `served_at` | ISO8601\|null | Per-item preparation/service timestamps (set by the vendor flow). |

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

**Authentication:** required (Bearer token).

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `page` | int | Orders page number (default: `1`). |
| `per_page` | int | Orders per restaurant group (default: `10`, max: `50`). |

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
      "last_ordered_at": "2024-10-10T10:30:00Z",
      "orders": [
        {
          "order_id": "ORD-8801",
          "order_public_id": "ord-aB3xK9pQrS12",
          "created_at": "2024-10-10T10:30:00Z",
          "order_type": "dine-in",
          "payment_status": "paid",
          "payment_method": "card",
          "items_count": 2,
          "total_amount": 24.24,
          "items": [
            {
              "id": 42,
              "menu_item_id": 42,
              "name": "Tonkotsu Ramen",
              "quantity": 1,
              "unit_price": 17.86,
              "line_total": 17.86,
              "vat_rate": 10,
              "tax_category": "food",
              "image_url": "https://example.com/media/menu-items/42/photo.png",
              "notes": null,
              "paid_addons": [],
              "free_addons": [],
              "removed_items": [],
              "selected_modifiers": []
            },
            {
              "id": 57,
              "menu_item_id": 57,
              "name": "Matcha Latte",
              "quantity": 1,
              "unit_price": 9.60,
              "line_total": 9.60,
              "vat_rate": 20,
              "tax_category": "beverage_non_alcoholic",
              "image_url": null,
              "notes": null,
              "paid_addons": [],
              "free_addons": [],
              "removed_items": [],
              "selected_modifiers": []
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
              "vat_amount": 1.60,
              "gross_amount": 9.60,
              "net_amount": 8.00
            }
          ],
          "totals": {
            "net_total": 24.24,
            "vat_total": 3.22,
            "service_fee": 0.00,
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

Returns one order detail for the authenticated customer.

**Authentication:** required (Bearer token).

**Payment methods:** `card`, `stripe`, `cash`.

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
  "created_at": "2024-02-24T10:30:00Z",
  "status": "delivered",
  "order_type": "dine-in",
  "payment_status": "paid",
  "payment_method": "card",
  "items": [
    {
      "menu_item_id": 42,
      "name": "Tonkotsu Ramen",
      "quantity": 1,
      "unit_price": 17.86,
      "line_total": 17.86,
      "vat_rate": 10,
      "tax_category": "food",
      "image_url": null,
      "notes": null,
      "paid_addons": [],
      "free_addons": [],
      "removed_items": [],
      "selected_modifiers": []
    },
    {
      "menu_item_id": 57,
      "name": "Matcha Latte",
      "quantity": 1,
      "unit_price": 9.60,
      "line_total": 9.60,
      "vat_rate": 20,
      "tax_category": "beverage_non_alcoholic",
      "image_url": null,
      "notes": null,
      "paid_addons": [],
      "free_addons": [],
      "removed_items": [],
      "selected_modifiers": []
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
      "vat_amount": 1.60,
      "gross_amount": 9.60,
      "net_amount": 8.00
    }
  ],
  "totals": {
    "net_total": 24.24,
    "vat_total": 3.22,
    "service_fee": 0.00,
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

### 4.3.1 Order Receipt

**GET** `/api/customer/orders/{orderPublicId}/receipt`

Returns a structured receipt payload for a paid order, including restaurant legal details, itemised order with VAT breakdown, tax groups, totals, payment confirmation, and legal notes. On first access, an invoice number is atomically generated and persisted on the order.

**Authentication:** required (Bearer token).

**Rules:**
- The order must belong to the authenticated customer.
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
      "date": "2026-06-05",
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
          "unit_price_gross": 8.90,
          "line_gross": 17.80,
          "tax_category": "FOOD",
          "vat_rate": 10
        },
        {
          "id": 2,
          "name": "Aperol Spritz",
          "quantity": 1,
          "unit_price_gross": 9.90,
          "line_gross": 9.90,
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
        "gross_amount": 17.80,
        "net_amount": 16.18,
        "vat_amount": 1.62
      },
      {
        "code": "B",
        "label": "Beverage Alcoholic (20%)",
        "tax_category": "BEVERAGE_ALCOHOLIC",
        "vat_rate": 20,
        "gross_amount": 9.90,
        "net_amount": 8.25,
        "vat_amount": 1.65
      }
    ],
    "totals": {
      "net_total": 24.43,
      "vat_total": 3.27,
      "service_fee": 2.77,
      "grand_total": 30.47
    },
    "payment": {
      "method": "stripe",
      "status": "CONFIRMED",
      "transaction_id": "pi_123456789",
      "paid_at": "2026-06-05T19:43:00+02:00"
    },
    "legal": {
      "invoice_note": "This invoice was issued in accordance with § 11 UStG (Austria).",
      "tax_note": "All prices include statutory VAT. The service date corresponds to the invoice date.",
      "company_register_note": "Registration number: FN 123456 a",
      "rksv_required_check": true
    }
  },
  "meta": {
    "generated_at": "2026-06-05T19:43:10+02:00",
    "template": "tavlo-receipt-template",
    "version": "1.0"
  }
}
```

**Field reference:**

| Section | Field | Source |
|---------|-------|--------|
| `restaurant.name` | `vendors.restaurant_name` | |
| `restaurant.logo_url` | `vendor_settings.logo_url` | Absolute URL |
| `restaurant.address` | `vendors.address, city, country` | Joined with comma |
| `restaurant.vat_id` | `vendors.vat_number` | |
| `restaurant.company_register_number` | `vendors.business_registration_number` | |
| `receipt.invoice_number` | Auto-generated from `vendor_settings.invoice_prefix` + `next_invoice_number` | Persisted on `orders.invoice_number` after first access |
| `receipt.locale` | Derived from `vendor_settings.default_language` + vendor country code | e.g. `en-AT`, `de-DE` |
| `receipt.table` | From `table_scan_session` → `restaurant_tables.name` or `Table {number}` | `null` if no table session |
| `order.items[].tax_category` | Uppercased `menu_items.tax_category` | e.g. `FOOD`, `BEVERAGE_ALCOHOLIC` |
| `payment.status` | `CONFIRMED` when `payment_received = true` | |
| `payment.transaction_id` | `orders.transaction_id` or `order_payments.stripe_payment_intent_id` | |
| `payment.paid_at` | `orders.payment_confirmed_at` or `order_payments.paid_at` | |
| `legal.invoice_note` | Hardcoded per vendor country | AT: § 11 UStG, DE: § 14 UStG, GB: UK VAT |
| `legal.rksv_required_check` | `true` for Austrian vendors | Hardcoded |

**Invoice number generation:**
- Format: `{invoice_prefix}-{zero-padded 7-digit number}` (e.g. `INV-0001001`).
- Generated atomically on first receipt access — `vendor_settings.next_invoice_number` is incremented with a row lock.
- Subsequent requests for the same order return the persisted `orders.invoice_number` without incrementing.

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
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `restaurant_id` | string | yes | Vendor numeric ID, `vendor_public_id`, or restaurant slug |
| `vendor_public_id` | string | no | Alias for `restaurant_id`; useful when the caller already has the vendor public ID |

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

### 4.5 Create Stripe Payment Intent

**POST** `/api/customer/payments/create-intent`

Creates a Stripe PaymentIntent for the authenticated customer's order. This endpoint is for Stripe Elements / PaymentElement; the frontend stays in the app and uses the returned `clientSecret`.

**Authentication:** required (Bearer token).

**Body:**
```json
{
  "order_id": "ord-aB3xK9pQrS12",
  "customer_id": 7
}
```

`order_id` may be either the numeric `orders.id` or `orders.order_public_id`. `customer_id` must be the numeric authenticated customer ID from the Bearer token.

**Backend behavior:**
- Resolves `order_id` by `orders.order_public_id` first, then numeric `orders.id`.
- Validates that `customer_id` matches the authenticated customer.
- Validates that the order belongs to the authenticated customer.
- Derives `table_session_id` from `orders.table_scan_session_id`; the frontend does not send it.
- Recalculates the final payable amount from cart rows already bound to the table order through `cart_items.order_id` plus shared rows in `shared_order_ids`, then updates `orders.amount`.
- Requires the restaurant's `vendor_settings` to have Stripe enabled, a `stripe_account_id`, and completed onboarding.
- Creates a platform PaymentIntent with `transfer_data.destination` set to the vendor Stripe account ID.
- Stores an `order_payments` row for audit and webhook reconciliation.

**PaymentIntent metadata:**
```json
{
  "order_id": "42",
  "order_public_id": "ord-aB3xK9pQrS12",
  "vendor_id": "1",
  "customer_id": "7",
  "table_session_id": "12",
  "payment_for": "dine_in"
}
```

**Response (200):**
```json
{
  "clientSecret": "pi_123_secret_abc",
  "paymentIntentId": "pi_123"
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

### 4.6 Update Stripe Payment Intent With Tip

**POST** `/api/customer/payments/update-intent`

Updates an existing Stripe PaymentIntent after the customer chooses a tip. The backend stores the tip on the order and updates the Stripe payable amount to `order amount + tip`.

**Authentication:** required (Bearer token).

**Body:**
```json
{
  "payment_intent_id": "pi_123_secret_abc",
  "order_id": "ord-aB3xK9pQrS12",
  "customer_id": 123,
  "tip_amount": 5.00
}
```

`payment_intent_id` may be either the PaymentIntent ID (`pi_123`) or the client secret (`pi_123_secret_abc`). `order_id` may be either the numeric `orders.id` or `orders.order_public_id`. `customer_id` must be the numeric authenticated customer ID from the Bearer token.

**Backend behavior:**
- Resolves the order and validates that it belongs to the authenticated customer.
- Validates that the PaymentIntent belongs to the same order/customer through `order_payments`.
- Stores `tip_amount` in `orders.tip_amount`.
- Keeps `orders.amount` as the order subtotal/payable amount before tip.
- Updates the Stripe PaymentIntent amount to `orders.amount + orders.tip_amount`.
- Updates `order_payments.amount` to the final charged amount including tip.

**Response (200):**
```json
{
  "clientSecret": "pi_123_secret_abc",
  "paymentIntentId": "pi_123"
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

### 4.7 Verify Stripe Payment Intent

**GET** `/api/customer/payments/verify?payment_intent=pi_123`

Retrieves the PaymentIntent from Stripe, verifies it matches the authenticated customer's internal payment row, syncs the order payment fields, and returns the frontend-safe payment state.

**Authentication:** required (Bearer token).

**Response (200):**
```json
{
  "status": "succeeded",
  "orderStatus": "paid"
}
```

**Status values:**
| Field | Values |
|-------|--------|
| `status` | `requires_payment_method`, `requires_confirmation`, `requires_action`, `processing`, `succeeded`, `canceled` |
| `orderStatus` | `pending`, `paid`, `failed` |

**Mapping:**
- `succeeded` → `orderStatus: "paid"` and `orders.payment_received = true`.
- `processing`, `requires_action`, `requires_confirmation`, `requires_payment_method` → `orderStatus: "pending"`.
- `canceled` or failed webhook events → `orderStatus: "failed"`.

**Response (404):**
```json
{ "message": "No query results for model [App\\Models\\OrderPayment]." }
```

---

### 4.8 Stripe Payment Webhook

**POST** `/api/customer/payments/webhook`

Receives Stripe PaymentIntent events and keeps `order_payments` and `orders` in sync.

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

**Response (400) — invalid signature:**
```json
{ "message": "Invalid Stripe webhook signature." }
```

**Logging:** every webhook delivery attempt is logged to `stripe_webhook_logs` with event type, payment intent ID, HTTP status, outcome (`processed`, `signature_invalid`, `payment_not_found`, `ignored_event_type`), and any error message.

**Recovery:** a scheduled command `payments:reconcile-stale` runs every 5 minutes, polling Stripe directly for any `order_payments` rows stuck in a non-succeeded state for >10 minutes where the order is still `payment_pending = true`.

**Tables used:**
- `orders` stores current customer-facing payment flags and Stripe transaction ID.
- `order_payments` stores PaymentIntent audit/reconciliation data.
- `stripe_webhook_logs` stores every webhook delivery attempt and its outcome.
- `table_scan_sessions` links dine-in orders to the customer session.
- `cart_items.order_id` links owned item rows to the order; `cart_items.shared_order_ids` links shared item rows to participant orders.
- `vendor_settings` provides the vendor Stripe Connect account ID.

---

### 4.9 Track Participant Order 🔒

**GET** `/api/customer/orders/{orderPublicId}/tracking`

Returns the authenticated participant's order tracking payload. This endpoint is scoped to the authenticated customer; the order must belong to the customer directly or through the customer's `table_scan_session`.

**Authentication:** Bearer token with `auth:customer`.

**Request Body:** none

**Response (200):**
```json
{
  "id": 101,
  "order_public_id": "ORD-ABC123",
  "order_number": "1001",
  "status": "draft",
  "estimated_delivery_time": "2025-11-27T10:31:00+00:00",
  "total_amount": 25.99,
  "currency": "EUR",
  "order_type": "dine-in",
  "payment_method": null,
  "payment_pending": true,
  "payment_received": false,
  "items": [
    {
      "cart_item_id": 1,
      "menu_item_id": 42,
      "name": "Fries",
      "image_url": null,
      "quantity": 2,
      "unit_price": 3.85,
      "line_total": 7.70,
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
      "line_total": 7.70,
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
      "vat_amount": 2.60,
      "gross_amount": 28.59,
      "net_amount": 25.99
    }
  ],
  "totals": {
    "net_total": 25.99,
    "vat_total": 2.60,
    "service_fee": 0.00,
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

**Response (401):**
```json
{ "message": "Unauthenticated." }
```

**Response (404):**
```json
{ "message": "No query results for model [App\\Models\\Order]." }
```

---

## 5. Reservations 🔒

### 5.1 List Reservations

**GET** `/api/customer/reservations`

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `tab` | string | `upcoming`, `pending`, `past`, `cancelled` (default: upcoming) |
| `per_page` | int | Items per page (default: 20) |

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
    "reward_value_eur": 2.00,
    "is_redeemable": false
  }
]
```

---

### 6.2 Loyalty Wallet Detail

**GET** `/api/customer/loyalty/{vendorPublicId}`

Returns wallet details and paginated transaction history.

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
- `is_open` is computed from the vendor's `business_hours` for the current day/time.
- `status` is the human-readable string `"Open"` or `"Closed"`, mirroring `is_open`.
- `business_hours` is the vendor's configured weekly business-hours map, or `null` if unavailable.
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

---

### 8.2 Create Review

**POST** `/api/customer/reviews`

**Body:**
```json
{
  "vendor_public_id": "V-ABC123",
  "rating": 5,
  "review": "Amazing food and service!"
}
```

**Validation:**
- `vendor_public_id`: required, must identify an existing vendor
- `rating`: required, integer, 1–5
- `review`: nullable, string, max 2000
- Customer must have at least one paid, non-draft order with the vendor
- One review per customer/vendor

---

### 8.3 Update Review

**PATCH** `/api/customer/reviews/{reviewPublicId}`

**Body:**
```json
{
  "rating": 4,
  "text": "Updated review text"
}
```

---

### 8.4 Delete Review

**DELETE** `/api/customer/reviews/{reviewPublicId}`

**Body:**
```json
{}
```

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
