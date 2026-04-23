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

### 1.2 Login (Email/Password)

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

### 1.3 Social Register (Google / Apple / Facebook)

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

### 1.4 Social Login (Google / Apple / Facebook)

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

### 1.5 Get Current User

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

### 1.6 Logout

**POST** `/api/customer/logout` 🔒

Revokes the current access token.

**Response (200):**
```json
{
  "message": "Logged out."
}
```

---

### 1.7 Logout All Devices

**POST** `/api/customer/logout-all` 🔒

Revokes all access tokens for the customer.

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
  "profile": { ... },
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

---

### 2.2 Update Profile

**PATCH** `/api/customer/profile` 🔒

**Body (all optional):**
```json
{
  "gender": "male",
  "date_of_birth": "1990-01-15",
  "address": "123 Main Street, Vienna",
  "profile_picture": "https://example.com/photo.jpg"
}
```

**Validation:**
- `gender`: nullable, in: `male`, `female`, `other`, `prefer_not_to_say`
- `date_of_birth`: nullable, date, before today
- `address`: nullable, string, max 500
- `profile_picture`: nullable, string, max 500

**Note:** `first_name`, `last_name`, `email`, and `phone` are not editable per requirements.

---

### 2.3 Change Password

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
  { "id": 1, "name": "Burger", "slug": "burger" },
  { "id": 2, "name": "Pizza", "slug": "pizza" },
  { "id": 3, "name": "Starters", "slug": "starters" }
]
```

**Notes:**
- Only categories from restaurants with `is_live_and_discoverable = true` are included.
- Categories are deduplicated by name and sorted alphabetically.
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
      "logo_url": "https://example.com/logo.png",
      "cover_photo_url": "https://example.com/cover.jpg",
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
        "card": true,
        "cash": true
      },
      "loyalty": {
        "enabled": true,
        "points_per_euro": 20
      },
      "service_types": ["dine_in", "takeaway", "reservation"],
      "distance_km": 1.8
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
  "logo_url": "https://example.com/logo.png",
  "cover_photo_url": "https://example.com/cover.jpg",
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
    "card": true,
    "cash": true
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

---

### 3.4 Get Restaurant Categories

**GET** `/api/customer/restaurants/{vendorPublicId}/categories`

**Response (200):**
```json
[
  { "id": 1, "name": "Starters", "slug": "starters", "sort_order": 1 },
  { "id": 2, "name": "Mains", "slug": "mains", "sort_order": 2 }
]
```

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
    "image_url": "https://example.com/items/chicken-box.jpg",
    "price": 18.99,
    "has_discount": false,
    "discount_percent": null,
    "discounted_price": null,
    "rating": 4.4,
    "review_count": 252,
    "ordered_count": 1200,
    "popularity_rank": 4,
    "calories": 680,
    "dietary_preference": null,
    "category": {
      "id": 1,
      "name": "Burger",
      "slug": "burger"
    },
    "allergens": ["Gluten", "Eggs"],
    "tags": ["Popular", "Spicy"],
    "modifier_groups": []
  }
]
```

**Notes:**
- Both `category_id` and `search` are optional. If omitted, all active menu items are returned.
- `popularity_rank` is computed from `ordered_count` (e.g. `4` = "#4 most liked").
- `rating` is a percentage-style approval score (e.g. `88%` with `252` reviews).
- `discount_percent` and `discounted_price` are only present when `has_discount` is `true`.
- Each item includes its `category` for grouping/display.

---

### 3.6 Get Menu Item Detail

**GET** `/api/customer/restaurants/{vendorPublicId}/menu/{itemId}`

**Response (200):**
```json
{
  "id": 42,
  "name": "4 Piece chicken Box",
  "description": "4 Piece of hand-breaded original chicken with our special sauce and coleslaw.",
  "image_url": "https://example.com/items/chicken-box.jpg",
  "price": 18.99,
  "has_discount": true,
  "discount_percent": 15.00,
  "discounted_price": 16.14,
  "available": true,
  "rating": 4.4,
  "review_count": 252,
  "ordered_count": 1200,
  "calories": 680,
  "fat": 32.50,
  "carbs": 45.00,
  "protein": 38.00,
  "dietary_preference": null,
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
      "options": [
        { "id": 1, "name": "Fries", "price_adjustment": 0.00 },
        { "id": 2, "name": "Onion Rings", "price_adjustment": 1.50 },
        { "id": 3, "name": "Sweet Potato Fries", "price_adjustment": 2.00 }
      ]
    }
  ]
}
```

**Notes:**
- `available` indicates if the item is currently in stock.
- `ingredients` is a list of ingredient names (from the JSON column).
- `fat`, `carbs`, `protein` are in grams; `null` if not set.
- `dietary_preference` can be `vegetarian`, `vegan`, `gluten_free`, etc. or `null`.
- `allergens` and `tags` include `icon` for UI display.
- `modifier_groups` shows customisation options — `type` is `single` or `multiple`, with `min_selection`/`max_selection` constraints.
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
        "https://example.com/reviews/img1.jpg",
        "https://example.com/reviews/img2.jpg"
      ],
      "created_at": "2026-04-18T14:32:10+00:00",
      "reviewer": {
        "name": "John D.",
        "profile_picture": "https://example.com/avatars/john.jpg"
      },
      "menu_items": [
        {
          "id": 42,
          "name": "4 Piece chicken Box",
          "slug": "4-piece-chicken-box",
          "image_url": "https://example.com/items/chicken-box.jpg",
          "quantity": 1
        },
        {
          "id": 57,
          "name": "Caesar Salad",
          "slug": "caesar-salad",
          "image_url": "https://example.com/items/caesar.jpg",
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
  "total": 48
}
```

**Notes:**
- Only non-flagged reviews are returned.
- `menu_items` is derived from the order attached to the review. Each entry includes the item `name`, `quantity` ordered, and — when the vendor still has a matching menu item — its `id`, `slug`, and `image_url`. `id` and `image_url` are `null` if the item is no longer on the menu.
- `images` is an array of image URLs uploaded by the reviewer (may be empty).
- `reviewer.name` falls back to `"Anonymous"` if the customer has no name set.

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
    "Free Wi-Fi",
    "Outdoor seating",
    "Parking",
    "Wheelchair accessible",
    "Vegan options"
  ],
  "payment_methods": {
    "cash": true,
    "card": true,
    "visa": true,
    "mastercard": true,
    "amex": false,
    "apple_pay": true,
    "google_pay": true,
    "bank_transfer": false
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
- `restaurant_features` is a free-form array of feature labels chosen by the vendor (e.g. `Free Wi-Fi`, `Outdoor seating`, `Parking`, `Vegan options`).
- `payment_methods` reflects every accepted method on the vendor settings.
- `contact` is a partial object — each field is only included when the vendor has marked it as publicly visible:
  - `phone` requires `show_phone_public = true` (default `true`)
  - `email` requires `show_email_public = true` (default `false`)
  - `website` requires `show_website_public = true` (default `true`)
- Numeric stats (`years_of_experience`, `signature_recipes_count`, `happy_customers_count`) are `null` when the vendor hasn't set them.

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

### 3.10 Scan Table QR (Create Session) 🔒

**POST** `/api/customer/table/scan`

Customer scans a printed table QR code and creates a new table scan session with a unique 4-digit PIN.

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
  "pin": "0473",
  "session": {
    "id": "12",
    "status": "active",
    "scannedAt": "2026-04-23T10:15:00+00:00"
  },
  "table": { "id": "5", "number": 3, "name": "T3" },
  "vendor": { "id": "VID-8492", "name": "Bella Italia" }
}
```

**Response (409) — table already has an active session:**
```json
{
  "message": "This table already has an active session",
  "status": "active",
  "requiresPin": true,
  "table": { "id": "5", "number": 3, "name": "T3" },
  "vendor": { "id": "VID-8492", "name": "Bella Italia" }
}
```

**Flow note:**
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

### 3.11 Join Active Table With PIN 🔒

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
  "pin": null,
  "session": {
    "id": "13",
    "status": "active",
    "scannedAt": "2026-04-23T10:17:00+00:00"
  },
  "table": { "id": "5", "number": 3, "name": "T3" },
  "vendor": { "id": "VID-8492", "name": "Bella Italia" }
}
```

**Behavior:**
- A new `table_scan_sessions` row is created for the joining customer.
- The joining customer does **not** receive a new PIN, so `pin` is returned as `null`.
- Repeating the same request for a customer who is already joined returns the existing active session instead of creating a duplicate row.

**Response (200) — customer already joined:**
```json
{
  "pin": null,
  "session": {
    "id": "13",
    "status": "active",
    "scannedAt": "2026-04-23T10:17:00+00:00"
  },
  "table": { "id": "5", "number": 3, "name": "T3" },
  "vendor": { "id": "VID-8492", "name": "Bella Italia" }
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

## 4. Order History 🔒

### 4.1 List Restaurants with Orders

**GET** `/api/customer/orders/restaurants`

Returns all restaurants where the customer has placed orders.

**Response (200):**
```json
[
  {
    "vendor_public_id": "V-ABC123",
    "restaurant_name": "Café Central",
    "orders_count": 5,
    "orders_sum_amount": "156.50",
    "orders_max_created_at": "2026-04-01T..."
  }
]
```

---

### 4.2 Restaurant Order History

**GET** `/api/customer/orders/restaurants/{vendorPublicId}`

**Query Parameters:** `per_page`, `page`

**Response (200):**
```json
{
  "restaurant": { "id": "V-ABC123", "name": "Café Central" },
  "summary": { "total_orders": 5, "total_spent": "156.50" },
  "orders": { "data": [ ... ], "current_page": 1, ... }
}
```

---

### 4.3 Order Detail

**GET** `/api/customer/orders/{orderPublicId}`

**Response (200):** Full order details including vendor, items, amounts, fees.

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

### 7.1 List Favorites

**GET** `/api/customer/favorites`

---

### 7.2 Add Favorite

**POST** `/api/customer/favorites`

**Body:**
```json
{
  "vendor_public_id": "V-ABC123"
}
```

---

### 7.3 Remove Favorite

**DELETE** `/api/customer/favorites/{vendorPublicId}`

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
  "order_public_id": "ord_abc123...",
  "rating": 5,
  "text": "Amazing food and service!"
}
```

**Validation:**
- `order_public_id`: required, must belong to the customer
- `rating`: required, integer, 1–5
- `text`: nullable, string, max 2000
- One review per order

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

---

## 9. Privacy & Data 🔒

### 9.1 Request Data Export

**POST** `/api/customer/privacy/export`

Submits a GDPR data export request. Customer will receive a download link via email.

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
