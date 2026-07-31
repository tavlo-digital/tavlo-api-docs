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

### Delete Item

**DELETE** `/{vendorId}/inventory/items/{itemId}`

```json
{ "message": "Inventory item deleted" }
```

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
