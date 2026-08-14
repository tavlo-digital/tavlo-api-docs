# Inventory Management API

Base URL: `/api/vendor`

All routes require `Authorization: Bearer {token}` and the vendor must own the requested inventory.

## Inventory Categories

### List Categories

**GET** `/{vendorId}/inventory/categories`

Returns categories localized to the vendor's `dashboardLanguage`.

```json
[
  {
    "id": 8,
    "name": "Gemüse",
    "translations": {
      "en": { "name": "Vegetables" },
      "de": { "name": "Gemüse" }
    }
  }
]
```

### Create Category

**POST** `/{vendorId}/inventory/categories`

```json
{
  "name": "Vegetables",
  "translations": {
    "en": { "name": "Vegetables" },
    "de": { "name": "Gemüse" }
  }
}
```

`name` is optional when a name exists in English or another provided translation. Returns the category with status `201`.

### Update Category

**PATCH** `/{vendorId}/inventory/categories/{categoryId}`

```json
{
  "translations": {
    "de": { "name": "Frisches Gemüse" }
  }
}
```

All fields are optional. Translation updates are partial and omitted languages remain unchanged.

### Delete Category

**DELETE** `/{vendorId}/inventory/categories/{categoryId}`

```json
{ "message": "Inventory category deleted" }
```

Deleting a category does not delete its inventory items; their category reference is cleared.

## Inventory Units

Units are stored as JSON objects with translation support. Legacy plain-string units are normalized to objects on read.

### List Units

**GET** `/{vendorId}/inventory/units`

Auth: `vendor` token. Returns an array of unit objects:

```json
[
  {
    "name": "kg",
    "translations": { "en": { "name": "kg" }, "ar": { "name": "كغ" } }
  }
]
```

### Create Unit

**POST** `/{vendorId}/inventory/units`

Auth: `vendor` token.

| Field          | Type   | Required | Notes                       |
|----------------|--------|----------|-----------------------------|
| `name`         | string | no       | Resolved from translations if omitted |
| `translations` | object | no       | `{ lang: { name: "..." } }` |

Returns the full units array with status `201`. Duplicate names (case-insensitive) return `422`.

### Update Unit Translations

**PATCH** `/{vendorId}/inventory/units/{name}`

Auth: `vendor` token.

| Field          | Type   | Required | Notes                       |
|----------------|--------|----------|-----------------------------|
| `translations` | object | yes      | `{ lang: { name: "..." } }` |

Returns the full units array. The base `name` is re-derived from translations.

### Delete Unit

**DELETE** `/{vendorId}/inventory/units/{name}`

Returns the remaining units array.

## Inventory Items

### List Items

**GET** `/{vendorId}/inventory/items`

Ingredient names, supplier names, and category names are localized to the vendor's `dashboardLanguage`.

```json
[
  {
    "id": "12",
    "name": "Tomaten",
    "translations": {
      "en": { "name": "Tomatoes", "supplier": "Fresh Foods" },
      "de": { "name": "Tomaten", "supplier": "Frische Lebensmittel" }
    },
    "categoryId": 8,
    "category": "Gemüse",
    "quantity": 5,
    "unit": "kg",
    "minStock": 1,
    "reorderQuantity": 5,
    "costPerUnit": 2,
    "supplier": "Frische Lebensmittel",
    "isCritical": false,
    "autoReorder": false,
    "trackStock": true
  }
]
```

### Create Item

**POST** `/{vendorId}/inventory/items`

```json
{
  "name": "Tomatoes",
  "categoryId": 8,
  "quantity": 5,
  "unit": "kg",
  "minStock": 1,
  "reorderQuantity": 5,
  "costPerUnit": 2,
  "supplier": "Fresh Foods",
  "trackStock": true,
  "translations": {
    "en": { "name": "Tomatoes", "supplier": "Fresh Foods" },
    "de": { "name": "Tomaten", "supplier": "Frische Lebensmittel" }
  }
}
```

Returns the created item with status `201`. Quantity, unit, costs, nutrition, and stock rules are shared. Ingredient and supplier names are translated.

`supplier` is optional. When the initial quantity is greater than zero, the API also creates an `Initial Stock` activity entry.

### Bulk Import Items

**POST** `/{vendorId}/inventory/items/bulk`

Auth: authenticated `vendor` or authorized vendor team-member token. The authenticated actor must belong to `{vendorId}`.

Creates or updates up to 500 inventory items in one request. Matching is case-insensitive by ingredient name. Optional fields that are omitted are preserved on existing items and use their normal defaults for new items. Named categories are reused or created automatically.

```json
{
  "items": [
    {
      "ingredientName": "Tomatoes",
      "unit": "kg",
      "category": "Vegetables",
      "currentStock": 25,
      "reorderLevel": 10,
      "reorderQuantity": 20,
      "supplier": "Fresh Foods",
      "costPerUnit": 2.5
    }
  ]
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `items` | array | yes | 1–500 rows |
| `items.*.ingredientName` | string | yes | Maximum 255 characters; case-insensitive upsert key |
| `items.*.unit` | string | yes | Maximum 20 characters |
| `items.*.category` | string/null | no | Maximum 255 characters; auto-created when missing |
| `items.*.currentStock` | number/null | no | Minimum 0 |
| `items.*.reorderLevel` | number/null | no | Minimum 0 |
| `items.*.reorderQuantity` | number/null | no | Minimum 0 |
| `items.*.supplier` | string/null | no | Maximum 255 characters |
| `items.*.costPerUnit` | number/null | no | Minimum 0 |

Response `200`:

```json
{
  "created": 1,
  "updated": 0,
  "skipped": 0,
  "errors": []
}
```

Rows are committed independently. A database failure for one row rolls back that row, records it in `errors`, and does not undo successful rows. Request validation failures return `422` before any row is changed.

The dashboard's downloadable CSV template contains the required column headers only. It does not include an example inventory row.

### Update Item

**PATCH** `/{vendorId}/inventory/items/{itemId}`

```json
{
  "translations": {
    "de": {
      "name": "Kirschtomaten",
      "supplier": "Frische Lebensmittel"
    }
  }
}
```

All fields are optional. Translation updates are partial.

When `quantity` changes, the API records a `Manual Update` stock activity entry. Use the dedicated adjustment endpoint below when the reason and adjustment type are known.

### Get Item Details

**GET** `/{vendorId}/inventory/items/{itemId}/details`

Returns the live menu-recipe links for the ingredient and up to 50 newest persisted stock activities.

```json
{
  "affectedMenuItems": [
    {
      "id": "42",
      "name": "Tomato Pasta",
      "quantity": 0.25,
      "unit": "kg",
      "isCritical": true,
      "available": true
    }
  ],
  "activityLog": [
    {
      "id": "91",
      "date": "2026-08-15T12:30:00.000000Z",
      "type": "delivery",
      "source": "Supplier Delivery",
      "amount": 10,
      "quantityBefore": 5,
      "quantityAfter": 15,
      "note": "Saturday delivery",
      "user": "Manager"
    }
  ]
}
```

`affectedMenuItems` comes from `menu_item_ingredients`; it is empty when the ingredient is not used in a recipe. Activity history starts when the stock-movement migration is deployed—no demo or inferred historical events are returned.

### Adjust Item Stock

**POST** `/{vendorId}/inventory/items/{itemId}/adjust-stock`

Atomically changes the quantity and records the adjustment in the stock activity log.

```json
{
  "amount": -2.5,
  "type": "waste",
  "reason": "Spoiled batch"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `amount` | number | yes | Signed, non-zero change. Deliveries must be positive and waste must be negative. |
| `type` | string | yes | `delivery`, `waste`, or `correction` |
| `reason` | string/null | no | Maximum 500 characters |

The response contains the updated inventory `item` and the new `activity` entry. An adjustment that would make stock negative returns `422` and does not change the item.

Bulk imports also record `Excel Import` activity whenever an imported row changes an item quantity.

Completed order deductions record an `order` activity with the order number as its source.

### Delete Item

**DELETE** `/{vendorId}/inventory/items/{itemId}`

```json
{ "message": "Inventory item deleted" }
```

## Purchase Orders

Purchase orders use the supplier definitions stored in Inventory Settings. The server resolves the supplier, ingredient, ordering destination, unit cost, and currency from trusted vendor data instead of accepting those values from the browser.

### List Purchase Orders

**GET** `/{vendorId}/inventory/purchase-orders`

Returns the 100 newest persisted purchase orders.

### Create and Dispatch a Purchase Order

**POST** `/{vendorId}/inventory/purchase-orders`

```json
{
  "supplierId": "supplier-1",
  "inventoryItemId": 12,
  "quantity": 20,
  "estimatedDeliveryDate": "2026-08-18",
  "notes": "Deliver before noon"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `supplierId` | string | yes | Must identify an active supplier linked to the ingredient |
| `inventoryItemId` | integer | yes | Must belong to the authenticated vendor |
| `quantity` | number | yes | Greater than zero and not below the supplier's minimum quantity |
| `estimatedDeliveryDate` | date/null | no | Expected delivery date |
| `notes` | string/null | no | Maximum 1,000 characters |

The order is persisted before dispatch. Dispatch behavior is based on the supplier's configured `orderingMethod`:

- `Email`: sends a purchase-order email to the configured ordering email.
- `API`: posts the purchase-order payload to the configured ordering URL.
- `Phone`: persists the order for manual phone placement; no external request is attempted.

Response `201`:

```json
{
  "id": "9",
  "purchaseOrderPublicId": "PO-G7X2K9Q4MV",
  "inventoryItemId": "12",
  "ingredientName": "Tomatoes",
  "supplierId": "supplier-1",
  "supplierName": "Fresh Foods",
  "orderingMethod": "Email",
  "quantity": 20,
  "unit": "kg",
  "unitCost": 2.5,
  "totalCost": 50,
  "currency": "GBP",
  "estimatedDeliveryDate": "2026-08-18",
  "notes": "Deliver before noon",
  "status": "sent",
  "dispatchError": null,
  "createdBy": "Manager",
  "createdAt": "2026-08-15T13:00:00.000000Z"
}
```

Possible statuses are `sent`, `manual_action_required`, and `failed`. A dispatch failure does not discard the purchase order; the response contains `status: "failed"` and a `dispatchError` for follow-up.

The `currency` is derived from the vendor's approved country in Legal Information through the countries table. If the country has no configured currency, the fallback is `EUR`.

## Inventory Settings and Automatic Deduction

Supplier settings support `Email`, `API`, and `Phone` ordering and persist their email, phone, ordering URL, cutoff time, minimum order quantity, supported ingredients, and ingredient-specific supply configuration.

When both `general.enableInventoryTracking` and `general.enableAutoStockDeduction` are enabled, stock is deducted as linked cart items are served or picked up. A completed order also processes any linked items that were completed through the order-level status endpoint.

Deduction behavior:

- Uses the ingredient quantities configured in `menu_item_ingredients` and multiplies them by the cart-item quantity.
- Converts compatible `g`/`kg`, `ml`/`liter`, and piece/count units.
- Deducts tracked inventory items only.
- Honors removed ingredients recorded on the cart item.
- Clamps stock to zero unless `general.allowNegativeStock` is enabled.
- Creates an `order` stock activity for each changed ingredient.
- Marks each cart item as processed and enforces a unique cart-item/ingredient movement, so retrying a completion action cannot deduct twice. This also protects shared-cart items from duplicate deductions across linked orders.

When automatic deduction is disabled, completing an order does not modify inventory.

## Translation Payload

Translations use a language-keyed object:

```json
{
  "translations": {
    "en": { "name": "English name", "supplier": "English supplier" },
    "de": { "name": "Deutscher Name", "supplier": "Deutscher Lieferant" }
  }
}
```

Enabled language tabs come from vendor settings. Missing ingredient or supplier translations fall back to the stored base value.
