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
      "logo_url": "https://example.com/logo.png",
      "cover_photo_url": "https://example.com/cover.jpg",
      "currency": "EUR",
      "cuisines": ["burger", "Fast food"],
      "price_label": "Budget-friendly",
      "avg_rating": 4.2,
      "review_count": 890,
      "payment_methods": {
        "card": true,
        "cash": true
      },
      "loyalty": {
        "enabled": true,
        "points_per_euro": 20
      },
      "enable_reservations": true,
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
- `distance_km` is only returned when `latitude` and `longitude` are provided.
- `cuisine` filter matches by menu category ID.
- `price_range` filter checks if the vendor has active menu items in the given price bracket.
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
  "logo_url": "https://example.com/logo.png",
  "cover_photo_url": "https://example.com/cover.jpg",
  "currency": "EUR",
  "cuisines": ["burger", "Fast food"],
  "avg_rating": 4.2,
  "review_count": 890,
  "is_open": true,
  "today_hours": "10:45 – 20:45",
  "distance_km": 1.8,
  "payment_methods": {
    "card": true,
    "cash": true
  },
  "loyalty": {
    "enabled": true,
    "points_per_euro": 20
  },
  "enable_reservations": true
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

### 3.7 Get Restaurant Tables

**GET** `/api/customer/restaurants/{vendorPublicId}/tables`

**Response (200):**
```json
[
  { "id": 1, "number": 1, "name": "Table 1" }
]
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
