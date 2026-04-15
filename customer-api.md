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

### 3.1 List Restaurants

**GET** `/api/customer/restaurants`

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `city` | string | Filter by city |
| `search` | string | Search by name or city |
| `per_page` | int | Items per page (default: 20) |
| `page` | int | Page number |

**Response (200):** Paginated list of discoverable restaurants with settings.

---

### 3.2 Show Restaurant

**GET** `/api/customer/restaurants/{vendorPublicId}`

**Response (200):**
```json
{
  "restaurant": {
    "vendor_public_id": "V-ABC123",
    "restaurant_name": "Café Central",
    "city": "Vienna",
    "address": "Herrengasse 14",
    "vendor_setting": { "description": "...", "logo_url": "...", ... },
    "reviews": [ ... ]
  },
  "avg_rating": 4.5,
  "review_count": 42
}
```

---

### 3.3 Get Restaurant Categories

**GET** `/api/customer/restaurants/{vendorPublicId}/categories`

**Response (200):**
```json
[
  { "id": 1, "name": "Starters", "slug": "starters", "sort_order": 1 },
  { "id": 2, "name": "Mains", "slug": "mains", "sort_order": 2 }
]
```

---

### 3.4 Get Restaurant Menu

**GET** `/api/customer/restaurants/{vendorPublicId}/menu`

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `category_id` | int | Filter by category |
| `search` | string | Search by item name |

**Response (200):** Array of menu items with category, allergens, tags, and modifier groups.

---

### 3.5 Get Menu Item Detail

**GET** `/api/customer/restaurants/{vendorPublicId}/menu/{itemId}`

**Response (200):** Single menu item with full details.

---

### 3.6 Get Restaurant Tables

**GET** `/api/customer/restaurants/{vendorPublicId}/tables`

**Response (200):**
```json
[
  { "id": 1, "number": 1, "name": "Table 1", "qr_token": "uuid-token" }
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
