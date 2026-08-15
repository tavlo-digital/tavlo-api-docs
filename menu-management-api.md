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

### GET `/api/vendor/menu/category-options`

Returns the active master category list created by admins. Vendors select from this list when creating menu categories.

**Request:**
```
GET /api/vendor/menu/category-options
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Pizza",
      "slug": "pizza",
      "icon": "https://api.example.com/media/cat-icons/pizza.png",
      "sortOrder": 0
    }
  ]
}
```

---

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
      "masterCategoryId": 1,
      "name": "Antipasti",
      "slug": "antipasti",
      "icon": "https://api.example.com/media/cat-icons/antipasti.png",
      "taxCategory": {
        "id": 1,
        "slug": "food",
        "name": "Food",
        "vatRate": 10
      },
      "sortOrder": 0,
      "isActive": true,
      "itemCount": 2,
      "translations": {
        "en": { "name": "Starters" },
        "de": { "name": "Vorspeisen" }
      }
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
| `masterCategoryId` | integer\|null | Admin master category selected by the vendor |
| `name` | string | Category name |
| `slug` | string | URL-safe slug |
| `icon` | string\|null | Icon from the admin master category |
| `taxCategory` | object\|null | Linked system tax category |
| `taxCategory.id` | integer | Tax category ID (use for create/update) |
| `taxCategory.slug` | string | Tax category slug |
| `taxCategory.name` | string | Tax category display name |
| `taxCategory.vatRate` | float | VAT percentage |
| `sortOrder` | integer | Display order |
| `isActive` | boolean | Whether category is visible |
| `itemCount` | integer | Count of active items in category |
| `translations` | object | Language-keyed category names |

The top-level `name` is localized to the vendor's `dashboardLanguage`.

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
  "masterCategoryId": 2,
  "taxCategoryId": 2,
  "translations": {
    "en": { "name": "Drinks" },
    "de": { "name": "Getränke" }
  }
}
```

**Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `masterCategoryId` | integer | **Yes** for the vendor UI | Active admin master category ID |
| `taxCategoryId` | integer | No | Tax category FK. Defaults to food rate for vendor's country |
| `translations` | object | No | Language-keyed `{ "name": "..." }` values |

**Response `201`:**
```json
{
  "data": {
    "id": 35,
    "masterCategoryId": 2,
    "name": "Drinks",
    "slug": "drinks",
    "icon": "https://api.example.com/media/cat-icons/drinks.png",
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
  "masterCategoryId": 1,
  "taxCategoryId": 1,
  "sortOrder": 0,
  "isActive": true,
  "translations": {
    "de": { "name": "Neue Vorspeisen" }
  }
}
```

**Response `200`:** Same shape as POST response.

Translation updates are partial. Languages omitted from `translations` remain unchanged.

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

Returns all active menu items with stats. Supports filtering. Accessible to vendor owners and **waiter** team members (used by the waiter ordering sheet).

**Pricing fields:** `price`, `discountedPrice`, paid addon `price`, and modifier option `priceAdjustment` are **net** (the canonical values edited in menu management). Each also has a **gross (VAT-inclusive)** twin computed exactly like the customer menu API — `grossPrice`, `grossDiscountedPrice`, addon `grossPrice`, option `grossPriceAdjustment` (using the modifier group's `vatRate`) — so ordering UIs can show customer-identical prices. The top-level `serviceFeeRate` (percent, from vendor settings) lets clients preview the order total: `round(items_gross × serviceFeeRate/100, 2)` on top of the gross items total.

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
  "serviceFeeRate": 10.0,
  "data": [
    {
      "id": 40,
      "productUid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "categoryId": 26,
      "categoryName": "Antipasti",
      "name": "Bruschetta al Pomodoro",
      "description": "Toasted bread topped with fresh tomatoes, basil, garlic, and extra virgin olive oil.",
      "price": 8.9,
      "grossPrice": 9.79,
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
          "vatRate": 10.0,
          "sortOrder": 0,
          "options": [
            { "id": 1, "name": "Small", "priceAdjustment": 0.0, "grossPriceAdjustment": 0.0, "sortOrder": 0, "isActive": true },
            { "id": 2, "name": "Medium", "priceAdjustment": 2.0, "grossPriceAdjustment": 2.2, "sortOrder": 1, "isActive": true },
            { "id": 3, "name": "Large", "priceAdjustment": 4.0, "grossPriceAdjustment": 4.4, "sortOrder": 2, "isActive": true }
          ]
        }
      ],
      "modifierGroupIds": [1],
      "translations": {
        "en": {
          "name": "Bruschetta al Pomodoro",
          "description": "Toasted bread topped with fresh tomatoes."
        },
        "de": {
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
| `paidAddons` | array | Paid add-on objects. Each entry includes a stable `id`, `name`, `price`, optional `taxCategory`, and optional `translations` |
| `freeAddons` | array | Free add-on objects. Each entry includes a stable `id`, `name`, and optional `translations` |
| `removableItems` | array | Removable item objects. Each entry includes a stable `id`, `name`, and optional `translations` |
| `modifierGroups` | array | Linked modifier groups with options |
| `modifierGroupIds` | array[int] | Linked modifier group IDs in menu-item sort order |
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
  "manualNutritionOverride": true,
  "taxCategoryId": 1,
  "dietaryPreference": "vegetarian",
  "allergenIds": [2],
  "tagIds": [1, 3],
  "modifierGroupIds": [1],
  "hasDiscount": true,
  "discountPercent": 15,
  "translations": {
    "de": {
      "name": "Caprese Salat",
      "description": "Frische Mozzarella, Tomaten und Basilikum."
    },
    "en": {
      "name": "Caprese Salad",
      "description": "Fresh mozzarella, tomatoes, and basil."
    }
  },
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
| `manualNutritionOverride` | boolean | No | Preserves whether nutrition values are manually maintained instead of recipe-calculated |
| `taxCategoryId` | integer | No | FK to tax_categories. Inherits from category if omitted |
| `dietaryPreference` | string | No | e.g. "vegan", "vegetarian", "pescetarian" |
| `allergenIds` | array[int] | No | Array of allergen IDs from `/api/vendor/allergens` |
| `tagIds` | array[int] | No | Array of special tag IDs from `/api/vendor/special-tags` |
| `paidAddons` | array | No | Paid add-ons. `id` is preserved when supplied and generated when omitted. `name` and `price` are required for each entry |
| `paidAddons[].translations` | object | No | Language-keyed add-on names used by customer APIs |
| `freeAddons` | array | No | Free add-ons. Entries may be strings for legacy clients or objects with `id`, `name`, and `translations` |
| `removableItems` | array | No | Removable items. Entries may be strings for legacy clients or objects with `id`, `name`, and `translations` |
| `modifierGroupIds` | array[int] | No | Active modifier group IDs owned by this vendor. Ordering in array is saved as the item's modifier sort order |
| `hasDiscount` | boolean | No | Default: `false` |
| `discountPercent` | float | No | 0-100. Required when hasDiscount=true |
| `translations` | object\|array | No | Preferred shape is a language-keyed object; legacy arrays with a `language` field are also accepted |
| `translations.{language}.name` | string | No | Translated name |
| `translations.{language}.description` | string | No | Translated description |
| `ingredients` | array | No | Recipe ingredient links |
| `ingredients[].inventoryItemId` | integer | Yes (in array) | FK to inventory_items |
| `ingredients[].quantity` | float | Yes (in array) | Quantity per serving |
| `ingredients[].unit` | string | No | Unit: "g", "kg", "ml", "l", "piece" |
| `ingredients[].isCritical` | boolean | No | Default: `false`. Item auto-unavailable when stock = 0 |

**Response `201`:** Same shape as GET single item.

---

### POST `/api/vendor/menu/items/bulk`

Creates or updates up to 500 menu items from a vendor-side CSV/XLSX import. Authentication is required. Existing active items are matched case-insensitively by `name` within the resolved vendor category. Each row is committed independently, so an invalid row does not roll back successful rows.

The vendor dashboard accepts and downloads the full `tavlo-menu-items-full-import-template.xlsx` structure. Its first row contains merged section headings, its second row contains the 46 import column names, and menu data starts on row 3. The importer automatically detects and skips the grouped heading row.

Updates use the same versioning rules as `PATCH /api/vendor/menu/items/{itemId}`. Price, tax, or discount changes therefore create a new item version and preserve historical orders. Optional fields omitted from an existing row remain unchanged.

**Request:**
```http
POST /api/vendor/menu/items/bulk
Authorization: Bearer {token}
Content-Type: application/json
```

**Body:**
```json
{
  "items": [
    {
      "name": "Margherita Pizza",
      "category": "Main Courses",
      "categoryId": 26,
      "price": 12.5,
      "description": "Tomato, mozzarella, and basil",
      "available": true,
      "calories": 720,
      "fat": 20,
      "carbs": 90,
      "protein": 28,
      "manualNutritionOverride": true,
      "taxCategory": "food",
      "dietaryPreference": "vegetarian",
      "allergies": ["gluten", "dairy"],
      "specialTags": ["popular"],
      "hasDiscount": true,
      "discountPercent": 10,
      "translations": {
        "en": { "name": "Margherita Pizza", "description": "Tomato, mozzarella, and basil" },
        "de": { "name": "Pizza Margherita" }
      },
      "ingredients": [
        { "ingredientName": "Tomatoes", "quantity": 0.2, "isCritical": true }
      ],
      "modifierGroupNames": ["Choose a Size"],
      "paidAddons": [
        {
          "name": "Extra Cheese",
          "price": 2.5,
          "taxCategory": "food",
          "translations": { "de": { "name": "Extra Käse" } }
        }
      ],
      "freeAddons": [{ "name": "Ketchup", "translations": {} }],
      "removableItems": [{ "name": "Onion", "translations": { "de": { "name": "Zwiebel" } }]
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `items` | array | **Yes** | 1-500 menu item rows |
| `items[].name` | string | **Yes** | Item name, max 255 characters |
| `items[].categoryId` | integer | Recommended | Vendor category ID already validated by the dashboard preview; takes precedence over `category` when supplied |
| `items[].category` | string | Yes, when `categoryId` is omitted | Existing vendor category name; matching is case-insensitive and includes master and vendor-localized category names |
| `items[].price` | number | **Yes** | Net item price, minimum 0 |
| `items[].description` | string | No | Description, max 5000 characters |
| `items[].available` | boolean | No | Availability; omitted values stay unchanged on update and default to true on create |
| `items[].calories` | integer | No | Non-negative calorie count |
| `items[].fat` | number | No | Non-negative grams |
| `items[].carbs` | number | No | Non-negative grams |
| `items[].protein` | number | No | Non-negative grams |
| `items[].manualNutritionOverride` | boolean | No | Whether imported nutrition values are a manual override |
| `items[].taxCategory` | string | No | Tax category slug |
| `items[].dietaryPreference` | string\|null | No | Active dietary preference slug |
| `items[].allergies` | string[] | No | Allergen keys |
| `items[].specialTags` | string[] | No | Special-tag slugs |
| `items[].discountPercent` | number | No | 0-100; values above 0 enable the discount and 0 disables it |
| `items[].hasDiscount` | boolean | No | Explicitly enables/disables the discount; takes precedence over automatic inference from percentage |
| `items[].translations` | object | No | Language-keyed item names and descriptions |
| `items[].imageUrl` | string | No | External menu-item image URL |
| `items[].ingredients` | array | No | Recipe ingredients resolved against this vendor's inventory by ID or case-insensitive name |
| `items[].modifierGroupNames` | string[] | No | Existing active modifier groups resolved by base or translated name |
| `items[].paidAddons` | array | No | Paid add-ons with names, prices, optional tax category, and translations |
| `items[].freeAddons` | array | No | Free add-ons with translations |
| `items[].removableItems` | array | No | Removable ingredients/options with translations |

### Full XLSX column layout

| Section | Columns |
|---|---|
| Basic Details | `Item Name (EN) *`, item names for `DE`, `AR`, `ZH`, `ES`, `Category *`, `Availability`, descriptions for all five languages, `Image URL` |
| Pricing & Discount | `Price (EUR) *`, `Has Discount`, `Discount %` |
| Tax | `Tax Category` |
| Dietary & Allergens | `Dietary Preference`, `Allergens`, `Special Tags` |
| Nutrition | `Calories (kcal)`, `Fat (g)`, `Carbs (g)`, `Protein (g)`, `Manual Nutrition Override` |
| Recipe & Ingredients | `Ingredient 1 Name`, `Ingredient 1 Quantity per Serving`, `Ingredient 1 Critical` |
| Modifier Groups | `Modifier Group Names` |
| Paid Add-ons | Paid add-on names for `EN`, `DE`, `AR`, `ZH`, `ES`, `Paid Add-on Price (EUR)`, `Paid Add-on Tax Category` |
| Free Add-ons | Free add-on names for `EN`, `DE`, `AR`, `ZH`, `ES` |
| Removable Items | Removable names for `EN`, `DE`, `AR`, `ZH`, `ES` |

Multiple ingredients, modifier groups, or add-ons in one cell use semicolons. Aligned columns must contain the same number of values. For example, `Beef Patty;Cheddar`, `0,5;1`, and `Yes;No` create two recipe links. A single paid-add-on tax category applies to every paid add-on; otherwise provide one semicolon-separated tax category per add-on.

Category, inventory ingredient, and modifier-group references must already exist for the authenticated vendor. Matching is case-insensitive and includes localized category/modifier names. Display labels such as `Food`, `Vegan`, `Gluten`, and `Popular` are normalized to their API slugs/keys during import.

**Response `200`:**
```json
{
  "created": 1,
  "updated": 1,
  "skipped": 1,
  "errors": [
    {
      "row": 4,
      "name": "Tomato Soup",
      "message": "Category \"Soups\" does not exist."
    }
  ]
}
```

`row` is the one-based spreadsheet row number including the header row. Invalid item rows are reported and skipped independently. A request containing more than 500 items, or malformed top-level input, returns `422` validation errors.

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
  "translations": {
    "de": {
      "name": "Bruschetta mit Tomaten",
      "description": "Geröstetes Brot mit frischen Tomaten, Basilikum und Knoblauch."
    }
  }
}
```

**Notes:**
- `modifierGroupIds` — providing this **replaces** the item's linked modifier groups (sync behavior). Omit it to leave existing links unchanged; send `[]` to clear them.
- `translations` — updates existing translations by language code, keeps others untouched
- `ingredients` — providing this **replaces** all ingredients
- VAT is automatically recalculated if `taxCategoryId` changes

**Versioning behavior:** When any of the following fields change, the system creates a **new version** of the menu item (old row is soft-deleted, new row is created with the same `productUid`):
- `price`, `hasDiscount`, `discountPercent` — price-affecting fields
- `taxCategory`, `taxCategoryId` — changes gross price via VAT
- `paidAddons` — addon price changed or addon removed
- `freeAddons` — addon removed
- `modifierGroupIds` — modifier group added or removed

Non-versioned fields (`name`, `description`, `imageUrl`, `available`, `calories`, etc.) update in-place without creating a new version.

When a new version is created, the response returns the **new item's `id`** (different from the original). The `productUid` remains the same across all versions. Existing cart items and orders continue to reference the old version, preserving historical prices.

**Response `200`:** Same shape as GET single item.

---

### DELETE `/api/vendor/menu/items/{itemId}`

Always **soft-deletes** the menu item (sets `isActive = false` and marks as deleted). The row is preserved to maintain historical price data for existing cart items and orders.

**Request:**
```
DELETE /api/vendor/menu/items/40
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "message": "Menu item deleted"
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
      "translations": {
        "en": { "name": "Choose a size" },
        "de": { "name": "Größe wählen" }
      },
      "options": [
        {
          "id": 1,
          "name": "Small",
          "priceAdjustment": 0.0,
          "sortOrder": 0,
          "isActive": true,
          "translations": {
            "en": { "name": "Small" },
            "de": { "name": "Klein" }
          }
        },
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
| `translations` | object | Language-keyed group names |
| `options` | array | List of modifier options |
| `options[].id` | integer | Option ID |
| `options[].name` | string | Option name |
| `options[].priceAdjustment` | float | Price change (0 = free, positive = surcharge) |
| `options[].sortOrder` | integer | Display order |
| `options[].isActive` | boolean | Active state |
| `options[].translations` | object | Language-keyed option names |

Top-level group and option names are localized to the vendor's `dashboardLanguage`.

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
  "translations": {
    "en": { "name": "Cooking Level" },
    "de": { "name": "Garstufe" }
  },
  "options": [
    {
      "name": "Rare",
      "priceAdjustment": 0,
      "translations": {
        "en": { "name": "Rare" },
        "de": { "name": "Blutig" }
      }
    },
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
| `translations` | object | No | Language-keyed group names |
| `options` | array | No | Up to 10 options |
| `options[].name` | string | Yes (in array) | Option name |
| `options[].priceAdjustment` | float | No | Default: `0` |
| `options[].translations` | object | No | Language-keyed option names |

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
  "translations": {
    "de": { "name": "Größe" }
  },
  "options": [
    {
      "id": 1,
      "name": "Small",
      "priceAdjustment": 0,
      "translations": { "de": { "name": "Klein" } }
    },
    { "id": 2, "name": "Medium", "priceAdjustment": 2.5 },
    { "name": "Extra Large", "priceAdjustment": 6.0 }
  ]
}
```

**Notes:**
- Options with `id` are updated
- Options without `id` are created
- Options with existing IDs not in the array are **deleted**
- Translation objects are partial; omitted languages remain unchanged

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

Returns all system allergens (EU standard 14 major allergens + extras). Names are returned in the vendor's dashboard language.

**Request:**
```
GET /api/vendor/allergens
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "data": [
    { "id": 1, "key": "Gluten", "name": "Gluten", "icon": "🌾", "translations": { "en": { "name": "Gluten" }, "de": { "name": "Gluten" } } },
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

### GET `/api/vendor/dietary-preferences`

Returns active dietary preferences in the vendor's dashboard language.

**Authentication:** Vendor or team-member bearer token required.

**Request:** No request body.

```http
GET /api/vendor/dietary-preferences
Authorization: Bearer {token}
```

**Response `200`:**

```json
{
  "data": [
    { "id": 1, "slug": "vegetarian", "name": "Vegetarian", "icon": "🥬", "translations": { "en": { "name": "Vegetarian" }, "de": { "name": "Vegetarisch" } } },
    { "id": 2, "slug": "vegan", "name": "Vegan", "icon": "🌱" },
    { "id": 3, "slug": "pescetarian", "name": "Pescetarian", "icon": "🐟" }
  ]
}
```

---

### GET `/api/vendor/special-tags`

Returns all system special tags available for menu items. Labels are returned in the vendor's dashboard language.

**Request:**
```
GET /api/vendor/special-tags
Authorization: Bearer {token}
```

**Response `200`:**
```json
{
  "data": [
    { "id": 1, "slug": "recommended", "label": "Recommended", "icon": "⭐", "translations": { "en": { "label": "Recommended" }, "de": { "label": "Empfohlen" } } },
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

## Admin Dietary Preference, Allergen & Special Tag Management

Admins manage dietary preferences at `/admin/dietary-preferences`, alongside allergens and special tags. All three support multi-language translations.

### GET `/api/admin/allergens`

Returns all allergens with their translations.

**Request:**
```
GET /api/admin/allergens
Authorization: Bearer {admin-token}
```

**Response `200`:**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Gluten",
      "icon": "🌾",
      "sortOrder": 0,
      "isActive": true,
      "translations": {
        "en": { "name": "Gluten" },
        "de": { "name": "Gluten" },
        "fr": { "name": "Gluten" }
      }
    }
  ]
}
```

### POST `/api/admin/allergens`

Creates a new allergen with optional translations.

**Request body:**
```json
{
  "name": "Celery",
  "icon": "🌿",
  "sortOrder": 10,
  "isActive": true,
  "translations": {
    "de": { "name": "Sellerie" },
    "fr": { "name": "Celeri" }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | English name |
| `icon` | string | No | Emoji icon |
| `sortOrder` | int | No | Sort position |
| `isActive` | bool | No | Active state (default: true) |
| `translations` | object | No | Keyed by language code, value `{ "name": "..." }` |

### PATCH `/api/admin/allergens/{id}`

Updates an allergen and/or its translations. Same fields as POST, all optional.

### DELETE `/api/admin/allergens/{id}`

Deletes an allergen and all its translations.

---

### GET `/api/admin/special-tags`

Returns all special tags with their translations.

**Response `200`:**
```json
{
  "data": [
    {
      "id": 1,
      "slug": "recommended",
      "label": "Recommended",
      "icon": "⭐",
      "sortOrder": 0,
      "isActive": true,
      "translations": {
        "en": { "label": "Recommended" },
        "de": { "label": "Empfohlen" }
      }
    }
  ]
}
```

### POST `/api/admin/special-tags`

Creates a new special tag with optional translations.

**Request body:**
```json
{
  "label": "Seasonal",
  "slug": "seasonal",
  "icon": "🍂",
  "translations": {
    "de": { "label": "Saisonal" },
    "fr": { "label": "Saisonnier" }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `label` | string | Yes | English label |
| `slug` | string | No | URL-safe slug (auto-generated from label if omitted) |
| `icon` | string | No | Emoji icon |
| `sortOrder` | int | No | Sort position |
| `isActive` | bool | No | Active state (default: true) |
| `translations` | object | No | Keyed by language code, value `{ "label": "..." }` |

### PATCH `/api/admin/special-tags/{id}`

Updates a special tag and/or its translations. Same fields as POST, all optional.

### DELETE `/api/admin/special-tags/{id}`

Deletes a special tag and all its translations.

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

### Soft Delete & Versioning Behavior
- All menu item deletions are **soft deletes** — the row is preserved with `is_active = false` and `deleted_at` set
- When a price-affecting field is updated (`price`, `hasDiscount`, `discountPercent`, `taxCategory`, `paidAddons`, `freeAddons`, `modifierGroupIds`), the old item is soft-deleted and a **new version** is created with the same `productUid`
- `productUid` (UUID) links all versions of the same logical product
- Soft-deleted items are excluded from all GET endpoints
- Cart items and order items reference specific menu item versions, preserving historical prices
- Modifier groups and modifier options also use soft deletes

### Tax Calculation
- VAT rates are system-controlled per country — vendors cannot override
- Creating an item without `taxCategoryId` inherits the category's tax category
- Changing category tax category does **not** retroactively update item VAT rates
- Each item stores its resolved `vatRate` value at creation/update time
