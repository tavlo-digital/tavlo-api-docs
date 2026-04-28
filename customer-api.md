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
  "profile_picture": "http://localhost:8000/media/customers/1/avatar/abc123.jpg"
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
        "card": true,
        "cash": true
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
- `is_favorite` is `true` when the request is authenticated and the customer has favorited this restaurant; `false` otherwise (including for unauthenticated requests). Not shown above — also returned in the response payload.

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
    "image_url": "http://localhost:8000/media/menu-items/42/photo/abc123.jpg",
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
  "image_url": "http://localhost:8000/media/menu-items/42/photo/abc123.jpg",
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
- `menu_items` is derived from the order attached to the review. Each entry includes the item `name`, `quantity` ordered, and — when the vendor still has a matching menu item — its `id`, `slug`, and `image_url`. `id` and `image_url` are `null` if the item is no longer on the menu.
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
- `restaurant_features` is a list of structured feature objects chosen by the vendor. Each entry has:
  - `title` — short label (string, required, max 100)
  - `description` — optional longer explanation (string, max 500, may be `null`)
- `payment_methods` reflects every accepted method on the vendor settings.
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
  "message": "Table session started",
  "status": "active",
  "requiresPin": true,
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

> **Note:** `pin` in the 409 payload is the existing owner PIN — only the original scanning customer (or an admin/debug context) should rely on it. Joining customers should still be sent through `POST /api/customer/table/pin` with the PIN they were given verbally / on screen.

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
  "message": "Joined table session",
  "status": "active",
  "requiresPin": false,
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
  "message": "Already joined this table session",
  "status": "active",
  "requiresPin": false,
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

### 3.12 Table Cart 🔒

A **table cart** is the live cart of every customer currently sitting at the same `restaurant_table`. It is automatically scoped to the authenticated customer's currently-active `table_scan_session`.

**Authentication:** required (Bearer token).

**Scope rule:** the customer must have a row in `table_scan_sessions` with `status = active`. The cart includes every active session at the same `restaurant_table_id`.

If the customer has no active session, every cart endpoint returns:

```json
{ "message": "No active table session found." }
```
with HTTP `422`.

#### 3.12.1 Get Table Cart

**GET** `/api/customer/cart`

Returns all items per person at the same table.

**Response (200):**
```json
{
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
          "menu_item": { "id": 42, "name": "Fries", "price": 3.50, "image_url": null }
        }
      ]
    },
    {
      "session_id": 13,
      "customer_id": 8,
      "is_me": false,
      "name": "Bob Jones",
      "personal_items": []
    }
  ]
}
```

#### 3.12.2 Add Item

**POST** `/api/customer/cart/items`

Adds an item to the authenticated customer's cart.

**Body:**
```json
{
  "menu_item_id": 42,
  "quantity": 2,
  "notes": "No salt"
}
```

**Validation:**
- `menu_item_id`: required, must exist in `menu_items`
- `quantity`: optional, integer, 1–99 (default `1`)
- `notes`: optional, string, max 500

**Response (201):**
```json
{
  "id": 1,
  "quantity": 2,
  "notes": "No salt",
  "menu_item": { "id": 42, "name": "Fries", "price": 3.50, "image_url": null }
}
```

#### 3.12.3 Update Item

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

#### 3.12.4 Remove Item

**DELETE** `/api/customer/cart/items/{id}`

Removes an item owned by the current session.

**Response (204):** empty.

**Response (404):** if the item does not belong to the customer's current session.

---

### 3.13 Table Payment Summary 🔒

**GET** `/api/customer/table/order`

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
      "total_price": 12.50,
      "items": [
        {
          "cart_item_id": 1,
          "menu_item_id": 42,
          "name": "Fries",
          "image_url": null,
          "quantity": 2,
          "unit_price": 3.50,
          "total_price": 7.00,
          "is_mine": true
        },
        {
          "cart_item_id": 2,
          "menu_item_id": 51,
          "name": "Coke",
          "image_url": null,
          "quantity": 1,
          "unit_price": 5.50,
          "total_price": 5.50,
          "is_mine": true
        }
      ]
    },
    {
      "session_id": 13,
      "customer_id": 8,
      "is_me": false,
      "name": "Bob Jones",
      "item_count": 1,
      "total_price": 18.99,
      "items": [
        {
          "cart_item_id": 3,
          "menu_item_id": 42,
          "name": "4 Piece chicken Box",
          "image_url": "http://localhost:8000/media/menu-items/42/photo/abc123.jpg",
          "quantity": 1,
          "unit_price": 18.99,
          "total_price": 18.99,
          "is_mine": false
        }
      ]
    }
  ],
  "summary": {
    "item_count": 4,
    "total_price": 31.49
  }
}
```

**Notes:**
- The current table is derived from the customer's active `table_scan_sessions` row — no `table_id` needs to be passed in the URL or body.
- The authenticated customer's own line is included in `people[]` and identified by `is_me: true`.
- `is_mine` on an item is `true` when the item belongs to the authenticated customer's session.
- `unit_price` and `total_price` are floats in the restaurant's currency. `total_price` per item = `unit_price × quantity`.
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

### 3.14 Pay Now (Create Pending Order) 🔒

**POST** `/api/customer/table/order`

Creates a `pending` order for the authenticated customer based on the current table cart, optionally splitting selected items across N people. The order is created with `payment_pending = true` (no money is captured here — payment confirmation happens via a separate flow).

**Authentication:** required (Bearer token).

**Body (all fields optional):**
```json
{
  "shared_items": [
    { "cart_item_id": 3, "shared_between": 3 },
    { "cart_item_id": 7, "shared_between": 2 }
  ]
}
```

**Validation:**
- `shared_items`: optional array.
- `shared_items.*.cart_item_id`: required integer. Must reference a cart item that belongs to **any** active session at the same table (own cart or someone else's).
- `shared_items.*.shared_between`: required integer, `2`–`99`.

**Total computation:**
For each cart item across the table:
- **Not shared** + mine → full `line_total` is added to my bill.
- **Not shared** + someone else's → not on my bill.
- **Shared** → `share = line_total / shared_between`:
  - If the item is in **my** cart, I am billed only `share` (so `(N-1)/N` is removed from what I would have paid).
  - If the item is in **someone else's** cart, I am billed `share` extra.

`line_total = unit_price × quantity` (rounded to 2 decimals). `share` is also rounded to 2 decimals.

**Response (201):**
```json
{
  "order": {
    "id": 101,
    "order_public_id": "ord-aB3xK9pQrS12",
    "status": "pending",
    "payment_pending": true,
    "amount": 16.49,
    "currency": "EUR",
    "items_count": 3,
    "table_scan_session_id": 12,
    "vendor_id": 1,
    "created_at": "2026-04-27T10:30:00+00:00"
  }
}
```

**Notes:**
- The order is linked to the customer via `table_scan_session_id` (the `orders` table no longer carries a direct `customer_id`).
- The full per-line item snapshot and the split definitions are persisted on the order (`orders.items` and `orders.shared_items`) but are **not** returned by this endpoint — the response only exposes the totals.
- `payment_pending = true` and `status = "pending"` until a separate payment-confirmation step updates them.
- An empty body or omitted `shared_items` produces a non-split order (full prices for own items only).

**Response (422) — invalid shared cart item:**
```json
{
  "message": "One or more shared cart items do not belong to this table.",
  "invalid_cart_item_ids": [99]
}
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

Returns the full history view of the customer's currently active table — table + vendor + active session metadata, and every order the authenticated customer has placed during the current table session (full per-order snapshot, including items and shared-item splits).

**Authentication:** required (Bearer token).

**Scope rule:** the customer must have an `active` row in `table_scan_sessions`. The response is scoped to **that single active session** only — orders from other people sitting at the same table are not included.

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
          "id": 101,
          "order_public_id": "ord-aB3xK9pQrS12",
          "status": "pending",
          "payment_pending": true,
          "payment_received": false,
          "amount": 16.49,
          "currency": "EUR",
          "items_count": 3,
          "items": [
            {
              "cart_item_id": 1,
              "menu_item_id": 42,
              "name": "Fries",
              "image_url": null,
              "quantity": 2,
              "unit_price": 3.50,
              "line_total": 7.00,
              "is_mine": true,
              "shared": false,
              "amount_billed": 7.00
            },
            {
              "cart_item_id": 3,
              "menu_item_id": 51,
              "name": "Pizza",
              "image_url": null,
              "quantity": 1,
              "unit_price": 18.99,
              "line_total": 18.99,
              "is_mine": false,
              "shared": true,
              "shared_between": 2,
              "my_share": 9.49,
              "amount_billed": 9.49
            }
          ],
          "shared_items": [
            {
              "cart_item_id": 3,
              "menu_item_id": 51,
              "name": "Pizza",
              "quantity": 1,
              "line_total": 18.99,
              "shared_between": 2,
              "my_share": 9.49,
              "is_mine": false
            }
          ],
          "order_type": "dine-in",
          "table_scan_session_id": 12,
          "created_at": "2026-04-27T10:30:00+00:00"
        }
      ]
    }
  ],
  "summary": {
    "orders_count": 1,
    "total_amount": 16.49
  }
}
```

**Notes:**
- The current table is derived from the customer's active `table_scan_sessions` row — no `table_id` needs to be passed.
- `people[]` always contains exactly **one** entry — the authenticated customer's own active session (`is_me: true`). Other people sitting at the same table are intentionally excluded.
- `orders[]` is the **full, unedited** snapshot persisted on the `orders` row at pay-now time, including the per-line `items` array and the `shared_items` split definitions.
- `total_amount` is the sum of `amount` across the customer's orders. `summary.total_amount` mirrors it.
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
    "status": "Open"
  }
]
```

**Notes:**
- `avg_rating` is rounded to 1 decimal (0 if no reviews). `review_count` is the total number of reviews.
- `is_open` is computed from the vendor's `business_hours` for the current day/time.
- `status` is the human-readable string `"Open"` or `"Closed"`, mirroring `is_open`.

---

### 7.2 Add Favorite

**POST** `/api/customer/favorites/{vendorPublicId}/add` 🔒

Adds the given restaurant to the authenticated customer's favorites. The restaurant is identified by the `vendorPublicId` URL segment — no request body is required.

**Path Parameters:**
- `vendorPublicId` — the restaurant's `vendor_public_id` (e.g. `V-ABC123`).

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
