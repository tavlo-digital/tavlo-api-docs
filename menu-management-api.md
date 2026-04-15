# Menu Management API — Documentation

**Base URL:** `/api/vendor`  
**Authentication:** `Authorization: Bearer {token}` (vendor token via Sanctum)  
**Content-Type:** `application/json`

All endpoints require a valid vendor token. Tokens are obtained via `POST /api/vendor/auth/login`.

---

## Table of Contents

1. [Tax Categories](#1-tax-categories)
2. [Menu Categories](#2-menu-categories)
3. [Menu Items](#3-menu-items)
4. [Modifier Groups](#4-modifier-groups)
5. [Reference Lookups](#5-reference-lookups)

---

## 1. Tax Categories

Tax rates are **system-controlled** and cannot be overridden by vendors. The system returns rates for the vendor's registered country automatically.

### GET `/api/vendor/menu/tax-categories`

Returns all available tax categories for the vendor's country.

**Request:**
```
GET /api/vendor/menu/tax-categories
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "data": [
    {
      "id": 1,
      "slug": "food",
      "name": "Food",
      "vatRate": 10
    },
    {
      "id": 2,
      "slug": "beverage_non_alcoholic",
      "name": "Beverage (Non-Alcoholic)",
      "vatRate": 20
    },
    {
      "id": 3,
      "slug": "beverage_alcoholic",
      "name": "Beverage (Alcoholic)",
      "vatRate": 20
    }
  ]
}
```

**Tax Rates by Country:**

| Country | Food | Non-Alcoholic Beverage | Alcoholic Beverage |
|---------|------|------------------------|---------------------|
| AT (Austria) | 10% | 20% | 20% |
| DE (Germany) | 7% | 19% | 19% |
| GB (UK) | 0% | 20% | 20% |

---

## 2. Menu Categories

### GET `/api/vendor/menu/categories`

Returns all menu categories for the authenticated vendor.

**Request:**
```
GET /api/vendor/menu/categories
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "data": [
    {
      "id": 26,
      "name": "Antipasti",
      "slug": "antipasti",
      "taxCategory": {
        "id": 1,
        "slug": "food",
        "name": "Food",
        "vatRate": 10
      },
      "sortOrder": 0,
      "isActive": true,
      "itemCount": 2
    },
    {
      "id": 32,
      "name": "Bevande",
      "slug": "bevande",
      "taxCategory": {
        "id": 2,
        "slug": "beverage_non_alcoholic",
        "name": "Beverage (Non-Alcoholic)",
        "vatRate": 20
      },
      "sortOrder": 6,
      "isActive": true,
      "itemCount": 2
    }
  ]
}
```

**Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `id` | integer | Category ID |
| `name` | string | Category name |
| `slug` | string | URL-safe slug |
| `taxCategory` | object\|null | Linked system tax category |
| `taxCategory.id` | integer | Tax category ID (use for create/update) |
| `taxCategory.slug` | string | Tax category slug |
| `taxCategory.name` | string | Tax category display name |
| `taxCategory.vatRate` | float | VAT percentage |
| `sortOrder` | integer | Display order |
| `isActive` | boolean | Whether category is visible |
| `itemCount` | integer | Count of active items in category |

---

### POST `/api/vendor/menu/categories`

Creates a new menu category.

**Request:**
```
POST /api/vendor/menu/categories
Authorization: Bearer {token}
Content-Type: application/json
```

**Body:**
```json
{
  "name": "Drinks",
  "taxCategoryId": 2
}
```

**Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | **Yes** | Category name (max 255 chars) |
| `taxCategoryId` | integer | No | Tax category FK. Defaults to food rate for vendor's country |

**Response `201`:**
```json
{
  "data": {
    "id": 35,
    "name": "Drinks",
    "slug": "drinks",
    "taxCategory": {
      "id": 2,
      "slug": "beverage_non_alcoholic",
      "name": "Beverage (Non-Alcoholic)",
      "vatRate": 20
    },
    "sortOrder": 9,
    "isActive": true,
    "itemCount": 0
  }
}
```

**Error `422` — Name already exists:**
```json
{
  "message": "A category with this name already exists."
}
```

---

### PATCH `/api/vendor/menu/categories/{categoryId}`

Updates an existing menu category.

**Request:**
```
PATCH /api/vendor/menu/categories/26
Authorization: Bearer {token}
Content-Type: application/json
```

**Body (all fields optional):**
```json
{
  "name": "Antipasti & Starters",
  "taxCategoryId": 1,
  "sortOrder": 0,
  "isActive": true
}
```

**Response `200`:** Same shape as POST response.

---

### DELETE `/api/vendor/menu/categories/{categoryId}`

Deletes a menu category. **Fails if category has active items.**

**Request:**
```
DELETE /api/vendor/menu/categories/34
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "message": "Category deleted"
}
```

**Error `422` — Has items:**
```json
{
  "message": "Category cannot be deleted because menu items are assigned to it."
}
```

---

## 3. Menu Items

### GET `/api/vendor/menu/items`

Returns all active menu items with stats. Supports filtering.

**Request:**
```
GET /api/vendor/menu/items?categoryId=26&search=pizza&available=true
Authorization: Bearer {token}
```

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `categoryId` | integer | Filter by category |
| `search` | string | Search by name (case-insensitive) |
| `available` | boolean | Filter by availability |

**Response `200`:**
```json
{
  "stats": {
    "totalItems": 13,
    "totalCategories": 9,
    "averagePrice": 11.87,
    "averageRating": 4.59
  },
  "data": [
    {
      "id": 40,
      "categoryId": 26,
      "categoryName": "Antipasti",
      "name": "Bruschetta al Pomodoro",
      "description": "Toasted bread topped with fresh tomatoes, basil, garlic, and extra virgin olive oil.",
      "price": 8.9,
      "imageUrl": null,
      "available": true,
      "isActive": true,
      "calories": 280,
      "fat": 12.0,
      "carbs": 35.0,
      "protein": 6.0,
      "vatRate": 10.0,
      "taxCategory": "food",
      "dietaryPreference": "vegan",
      "allergens": [
        {
          "id": 1,
          "name": "Gluten",
          "icon": "🌾"
        }
      ],
      "tags": [
        {
          "id": 8,
          "slug": "popular",
          "label": "Popular",
          "icon": null
        }
      ],
      "modifierGroups": [
        {
          "id": 1,
          "name": "Size",
          "type": "single",
          "minSelection": 1,
          "maxSelection": 1,
          "isRequired": true,
          "sortOrder": 0,
          "options": [
            { "id": 1, "name": "Small", "priceAdjustment": 0.0, "sortOrder": 0, "isActive": true },
            { "id": 2, "name": "Medium", "priceAdjustment": 2.0, "sortOrder": 1, "isActive": true },
            { "id": 3, "name": "Large", "priceAdjustment": 4.0, "sortOrder": 2, "isActive": true }
          ]
        }
      ],
      "translations": {
        "en": {
          "language": "en",
          "name": "Bruschetta al Pomodoro",
          "description": "Toasted bread topped with fresh tomatoes."
        },
        "de": {
          "language": "de",
          "name": "Bruschetta mit Tomaten",
          "description": "Geröstetes Brot mit frischen Tomaten."
        }
      },
      "ingredients": [
        {
          "id": 1,
          "inventoryItemId": 5,
          "quantity": 100.0,
          "unit": "g",
          "isCritical": false
        }
      ],
      "hasDiscount": false,
      "discountPercent": 0.0,
      "discountedPrice": null,
      "rating": 4.6,
      "reviewCount": 42,
      "orderedCount": 187,
      "sortOrder": 0
    }
  ]
}
```

**Item Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `id` | integer | Item ID |
| `categoryId` | integer | Parent category ID |
| `categoryName` | string | Parent category name |
| `name` | string | Item name |
| `description` | string\|null | Item description |
| `price` | float | Base price |
| `imageUrl` | string\|null | Image URL |
| `available` | boolean | Whether item is currently available (sold out toggle) |
| `isActive` | boolean | Whether item is visible (false = soft deleted) |
| `calories` | integer | Calorie count |
| `fat` | float | Fat in grams |
| `carbs` | float | Carbohydrates in grams |
| `protein` | float | Protein in grams |
| `vatRate` | float | Applied VAT percentage |
| `taxCategory` | string | Tax category slug |
| `dietaryPreference` | string\|null | e.g. "vegan", "vegetarian" |
| `allergens` | array | Relational allergen objects |
| `tags` | array | Relational tag objects |
| `modifierGroups` | array | Linked modifier groups with options |
| `translations` | object | Language-keyed translation objects |
| `ingredients` | array | Recipe ingredient links |
| `hasDiscount` | boolean | Whether a discount is active |
| `discountPercent` | float | Discount percentage |
| `discountedPrice` | float\|null | Computed discounted price |
| `rating` | float | Average customer rating |
| `reviewCount` | integer\|null | Number of reviews |
| `orderedCount` | integer\|null | Total orders for this item |
| `sortOrder` | integer | Display order within category |

---

### GET `/api/vendor/menu/items/{itemId}`

Returns a single menu item by ID.

**Request:**
```
GET /api/vendor/menu/items/40
Authorization: Bearer {token}
```

**Response `200`:** Same shape as a single item in the list.

**Error `404`:**
```json
{ "message": "No query results for model [App\\Models\\MenuItem] 999" }
```

---

### POST `/api/vendor/menu/items`

Creates a new menu item with optional allergens, tags, modifier groups, translations, and ingredients.

**Request:**
```
POST /api/vendor/menu/items
Authorization: Bearer {token}
Content-Type: application/json
```

**Body:**
```json
{
  "categoryId": 26,
  "name": "Caprese Salad",
  "description": "Fresh mozzarella, tomatoes, and basil with olive oil.",
  "price": 11.50,
  "imageUrl": "https://example.com/caprese.jpg",
  "available": true,
  "calories": 320,
  "fat": 22.0,
  "carbs": 12.0,
  "protein": 14.0,
  "taxCategoryId": 1,
  "dietaryPreference": "vegetarian",
  "allergenIds": [2],
  "tagIds": [1, 3],
  "modifierGroupIds": [1],
  "hasDiscount": true,
  "discountPercent": 15,
  "translations": [
    {
      "language": "de",
      "name": "Caprese Salat",
      "description": "Frische Mozzarella, Tomaten und Basilikum."
    },
    {
      "language": "en",
      "name": "Caprese Salad",
      "description": "Fresh mozzarella, tomatoes, and basil."
    }
  ],
  "ingredients": [
    {
      "inventoryItemId": 3,
      "quantity": 150,
      "unit": "g",
      "isCritical": true
    }
  ]
}
```

**Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `categoryId` | integer | **Yes** | Must belong to this vendor |
| `name` | string | **Yes** | Item name (max 255 chars) |
| `description` | string | No | Item description (max 2000 chars) |
| `price` | float | **Yes** | Base price ≥ 0 |
| `imageUrl` | string | No | Image URL |
| `available` | boolean | No | Default: `true` |
| `calories` | integer | No | Default: `0` |
| `fat` | float | No | Default: `0` |
| `carbs` | float | No | Default: `0` |
| `protein` | float | No | Default: `0` |
| `taxCategoryId` | integer | No | FK to tax_categories. Inherits from category if omitted |
| `dietaryPreference` | string | No | e.g. "vegan", "vegetarian", "pescetarian" |
| `allergenIds` | array[int] | No | Array of allergen IDs from `/api/vendor/allergens` |
| `tagIds` | array[int] | No | Array of special tag IDs from `/api/vendor/special-tags` |
| `modifierGroupIds` | array[int] | No | Array of modifier group IDs. Ordering in array = sortOrder |
| `hasDiscount` | boolean | No | Default: `false` |
| `discountPercent` | float | No | 0-100. Required when hasDiscount=true |
| `translations` | array | No | Array of translation objects |
| `translations[].language` | string | Yes (in array) | Language code: "en", "de", "tr", "it", "fr", "ar" |
| `translations[].name` | string | Yes (in array) | Translated name |
| `translations[].description` | string | No | Translated description |
| `ingredients` | array | No | Recipe ingredient links |
| `ingredients[].inventoryItemId` | integer | Yes (in array) | FK to inventory_items |
| `ingredients[].quantity` | float | Yes (in array) | Quantity per serving |
| `ingredients[].unit` | string | No | Unit: "g", "kg", "ml", "l", "piece" |
| `ingredients[].isCritical` | boolean | No | Default: `false`. Item auto-unavailable when stock = 0 |

**Response `201`:** Same shape as GET single item.

---

### PATCH `/api/vendor/menu/items/{itemId}`

Updates an existing menu item (all fields optional).

**Request:**
```
PATCH /api/vendor/menu/items/40
Authorization: Bearer {token}
Content-Type: application/json
```

**Body (any subset of POST fields):**
```json
{
  "price": 9.50,
  "hasDiscount": true,
  "discountPercent": 10,
  "allergenIds": [1, 2],
  "tagIds": [1],
  "translations": [
    {
      "language": "de",
      "name": "Bruschetta mit Tomaten",
      "description": "Geröstetes Brot mit frischen Tomaten, Basilikum und Knoblauch."
    }
  ]
}
```

**Notes:**
- `allergenIds`, `tagIds`, `modifierGroupIds` — providing these **replaces** the full set (sync behavior)
- `translations` — updates existing translations by language code, keeps others untouched
- `ingredients` — providing this **replaces** all ingredients
- VAT is automatically recalculated if `taxCategoryId` changes

**Response `200`:** Same shape as GET single item.

---

### DELETE `/api/vendor/menu/items/{itemId}`

Deletes a menu item. If the item exists in orders, it is **soft deleted** (`isActive = false`) instead.

**Request:**
```
DELETE /api/vendor/menu/items/40
Authorization: Bearer {token}
```

**Response `200` — Hard delete (item has no orders):**
```json
{
  "message": "Menu item deleted"
}
```

**Response `200` — Soft delete (item referenced in orders):**
```json
{
  "message": "Menu item hidden (soft deleted — referenced in orders)"
}
```

> Soft-deleted items are excluded from all GET endpoints. They remain in the database to preserve order history and invoices.

---

### PATCH `/api/vendor/menu/items/{itemId}/toggle`

Quickly toggles the `available` field (sold out / back in stock).

**Request:**
```
PATCH /api/vendor/menu/items/40/toggle
Authorization: Bearer {token}
```

**Response `200`:** Same shape as GET single item with updated `available` field.

---

## 4. Modifier Groups

Modifier groups replace the old paid_addons/free_addons/removable_items JSON structure. They allow rich customization options for complex dishes (pizza sizes, toppings, cooking levels, etc.).

### GET `/api/vendor/menu/modifier-groups`

Returns all modifier groups for the vendor.

**Request:**
```
GET /api/vendor/menu/modifier-groups
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Size",
      "type": "single",
      "minSelection": 1,
      "maxSelection": 1,
      "isRequired": true,
      "sortOrder": 0,
      "isActive": true,
      "options": [
        { "id": 1, "name": "Small", "priceAdjustment": 0.0, "sortOrder": 0, "isActive": true },
        { "id": 2, "name": "Medium", "priceAdjustment": 2.0, "sortOrder": 1, "isActive": true },
        { "id": 3, "name": "Large", "priceAdjustment": 4.0, "sortOrder": 2, "isActive": true }
      ]
    },
    {
      "id": 2,
      "name": "Toppings",
      "type": "multiple",
      "minSelection": 0,
      "maxSelection": 5,
      "isRequired": false,
      "sortOrder": 1,
      "isActive": true,
      "options": [
        { "id": 4, "name": "Mushrooms", "priceAdjustment": 1.5, "sortOrder": 0, "isActive": true },
        { "id": 5, "name": "Olives", "priceAdjustment": 1.0, "sortOrder": 1, "isActive": true },
        { "id": 6, "name": "Salami", "priceAdjustment": 2.0, "sortOrder": 2, "isActive": true }
      ]
    }
  ]
}
```

**Group Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `id` | integer | Group ID |
| `name` | string | Group name |
| `type` | string | `"single"`, `"multiple"`, or `"remove"` |
| `minSelection` | integer | Minimum required selections |
| `maxSelection` | integer\|null | Maximum allowed selections |
| `isRequired` | boolean | Whether customer must select at least minSelection |
| `sortOrder` | integer | Display order |
| `isActive` | boolean | Active state |
| `options` | array | List of modifier options |
| `options[].id` | integer | Option ID |
| `options[].name` | string | Option name |
| `options[].priceAdjustment` | float | Price change (0 = free, positive = surcharge) |
| `options[].sortOrder` | integer | Display order |
| `options[].isActive` | boolean | Active state |

---

### POST `/api/vendor/menu/modifier-groups`

Creates a new modifier group with options.

**Request:**
```
POST /api/vendor/menu/modifier-groups
Authorization: Bearer {token}
Content-Type: application/json
```

**Body:**
```json
{
  "name": "Cooking Level",
  "type": "single",
  "minSelection": 1,
  "maxSelection": 1,
  "isRequired": true,
  "options": [
    { "name": "Rare", "priceAdjustment": 0 },
    { "name": "Medium Rare", "priceAdjustment": 0 },
    { "name": "Well Done", "priceAdjustment": 0 }
  ]
}
```

**Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | **Yes** | Group name (max 255 chars) |
| `type` | string | **Yes** | `"single"`, `"multiple"`, or `"remove"` |
| `minSelection` | integer | No | Default: `0` |
| `maxSelection` | integer | No | Null = unlimited |
| `isRequired` | boolean | No | Default: `false` |
| `sortOrder` | integer | No | Default: auto-increment |
| `options` | array | No | Up to 10 options |
| `options[].name` | string | Yes (in array) | Option name |
| `options[].priceAdjustment` | float | No | Default: `0` |

**Response `201`:** Same shape as GET list item.

---

### PATCH `/api/vendor/menu/modifier-groups/{groupId}`

Updates a modifier group and syncs its options.

**Request:**
```
PATCH /api/vendor/menu/modifier-groups/1
Authorization: Bearer {token}
Content-Type: application/json
```

**Body:**
```json
{
  "name": "Size",
  "options": [
    { "id": 1, "name": "Small", "priceAdjustment": 0 },
    { "id": 2, "name": "Medium", "priceAdjustment": 2.5 },
    { "name": "Extra Large", "priceAdjustment": 6.0 }
  ]
}
```

**Notes:**
- Options with `id` are updated
- Options without `id` are created
- Options with existing IDs not in the array are **deleted**

**Response `200`:** Same shape as GET list item.

---

### DELETE `/api/vendor/menu/modifier-groups/{groupId}`

Soft deletes a modifier group (`isActive = false`).

**Request:**
```
DELETE /api/vendor/menu/modifier-groups/1
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "message": "Modifier group deactivated"
}
```

---

## 5. Reference Lookups

These endpoints return the system-wide lists used to populate dropdowns in the UI.

### GET `/api/vendor/allergens`

Returns all system allergens (EU standard 14 major allergens + extras).

**Request:**
```
GET /api/vendor/allergens
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "data": [
    { "id": 1, "name": "Gluten", "icon": "🌾" },
    { "id": 2, "name": "Dairy", "icon": "🥛" },
    { "id": 3, "name": "Eggs", "icon": "🥚" },
    { "id": 4, "name": "Nuts", "icon": "🥜" },
    { "id": 5, "name": "Peanuts", "icon": "🥜" },
    { "id": 6, "name": "Soy", "icon": "🌱" },
    { "id": 7, "name": "Fish", "icon": "🐟" },
    { "id": 8, "name": "Sesame", "icon": "🫙" },
    { "id": 9, "name": "Shellfish", "icon": "🦐" },
    { "id": 10, "name": "Mustard", "icon": "🌿" }
  ]
}
```

---

### GET `/api/vendor/special-tags`

Returns all system special tags available for menu items.

**Request:**
```
GET /api/vendor/special-tags
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "data": [
    { "id": 1, "slug": "recommended", "label": "Recommended", "icon": "⭐" },
    { "id": 2, "slug": "chefs-pick", "label": "Chef's Pick", "icon": "👨‍🍳" },
    { "id": 3, "slug": "todays-special", "label": "Today's Special", "icon": "🌟" },
    { "id": 4, "slug": "organic", "label": "Organic / Bio", "icon": "🌿" },
    { "id": 5, "slug": "halal", "label": "Halal", "icon": "🔌" },
    { "id": 8, "slug": "popular", "label": "Popular", "icon": null },
    { "id": 9, "slug": "new", "label": "New", "icon": null },
    { "id": 10, "slug": "spicy", "label": "Spicy", "icon": null },
    { "id": 11, "slug": "vegetarian", "label": "Vegetarian", "icon": null },
    { "id": 12, "slug": "vegan", "label": "Vegan", "icon": null }
  ]
}
```

---

## Error Responses

### Validation Error `422`
```json
{
  "message": "The name field is required.",
  "errors": {
    "name": ["The name field is required."]
  }
}
```

### Not Found `404`
```json
{
  "message": "No query results for model [App\\Models\\MenuItem] 999"
}
```

### Unauthorized `401`
```json
{
  "message": "Unauthenticated."
}
```

---

## Complete Flow Example

### 1. Get tax categories for the vendor's country
```
GET /api/vendor/menu/tax-categories
```

### 2. Create a category with the appropriate tax category
```json
POST /api/vendor/menu/categories
{
  "name": "Pizza",
  "taxCategoryId": 1
}
```

### 3. Create modifier groups for the category's items
```json
POST /api/vendor/menu/modifier-groups
{
  "name": "Size",
  "type": "single",
  "minSelection": 1,
  "maxSelection": 1,
  "isRequired": true,
  "options": [
    { "name": "Small (25cm)", "priceAdjustment": 0 },
    { "name": "Large (35cm)", "priceAdjustment": 4.0 }
  ]
}
```

### 4. Get system allergens and tags for dropdowns
```
GET /api/vendor/allergens
GET /api/vendor/special-tags
```

### 5. Create a menu item with all relationships
```json
POST /api/vendor/menu/items
{
  "categoryId": 30,
  "name": "Pizza Margherita",
  "description": "San Marzano tomato sauce, fresh mozzarella, basil, and extra virgin olive oil.",
  "price": 12.90,
  "dietaryPreference": "vegetarian",
  "allergenIds": [1, 2],
  "tagIds": [1],
  "modifierGroupIds": [1],
  "hasDiscount": false,
  "translations": [
    { "language": "en", "name": "Margherita Pizza", "description": "Classic tomato and mozzarella." },
    { "language": "de", "name": "Margherita Pizza", "description": "Klassische Tomaten-Mozzarella Pizza." }
  ]
}
```

### 6. Toggle item availability (sold out)
```
PATCH /api/vendor/menu/items/46/toggle
```

---

## Architecture Notes

### Database Structure
```
menu_categories ──FK──→ tax_categories (system controlled)
      │
      └──→ menu_items
                │
                ├── menu_item_allergens ──→ allergens
                ├── menu_item_tags ──→ special_tags
                ├── menu_item_modifier_groups ──→ modifier_groups
                │                                       └── modifier_options
                ├── menu_item_translations
                └── menu_item_ingredients ──→ inventory_items
```

### Soft Delete Behavior
- Items in orders: `is_active = false` (hidden, preserved in DB)
- Items not in orders: permanent hard delete
- Soft-deleted items are excluded from all GET endpoints

### Tax Calculation
- VAT rates are system-controlled per country — vendors cannot override
- Creating an item without `taxCategoryId` inherits the category's tax category
- Changing category tax category does **not** retroactively update item VAT rates
- Each item stores its resolved `vatRate` value at creation/update time
