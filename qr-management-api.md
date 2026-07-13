# QR & Table Management API — Documentation

**Base URL:** `/api/vendor`  
**Authentication:** `Authorization: Bearer {token}` (vendor token via Sanctum)  
**Content-Type:** `application/json`

Authenticated endpoints require a valid vendor token obtained via `POST /api/vendor/login`.  
Endpoints marked **Public** do not require authentication — they are called by the customer-facing QR scan page.

---

## Table of Contents

1. [Tables — CRUD](#1-tables--crud)
2. [Table QR Codes](#2-table-qr-codes)
3. [Table Scan (Public)](#3-table-scan-public)
4. [Takeaway QR Code](#4-takeaway-qr-code)
5. [Takeaway Scan (Public)](#5-takeaway-scan-public)
6. [Sync Tables](#6-sync-tables)
7. [Close Active Table Sessions](#7-close-active-table-sessions)
8. [Table Status Logic](#8-table-status-logic)
9. [Error Reference](#9-error-reference)

---

## 1. Tables — CRUD

### GET `/api/vendor/{vendorId}/tables`

Returns all tables for the vendor, each with a computed real-time `status`.

**Response `200`:**
```json
[
  {
    "id": "1",
    "number": 1,
    "name": "Table 1",
    "qrToken": "550e8400-e29b-41d4-a716-446655440000",
    "isActive": true,
    "status": "idle",
    "qrCreatedAt": "2026-03-29T10:00:00.000Z",
    "lastScannedAt": null
  }
]
```

**Status values:**

| Value | Meaning | Color |
|---|---|---|
| `idle` | No active session | 🟢 Green |
| `active` | Order in progress (pending / confirmed / preparing / ready) | 🟡 Yellow |
| `waiting_payment` | Order served but payment pending | 🔴 Red |

---

### POST `/api/vendor/{vendorId}/tables`

Creates a new table and generates a unique QR token.

**Request:**
```json
{
  "number": 5,
  "name": "VIP Room"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `number` | integer | ✅ | Must be ≥ 1 |
| `name` | string | ❌ | Defaults to `"Table {number}"` if omitted |

**Response `201`:**
```json
{
  "id": "5",
  "number": 5,
  "name": "VIP Room",
  "qrToken": "a3f1c2e0-...",
  "isActive": true,
  "status": "idle",
  "qrCreatedAt": "2026-03-29T11:00:00.000Z",
  "lastScannedAt": null
}
```

---

### PATCH `/api/vendor/{vendorId}/tables/{tableId}`

Updates a table's name or active state.

**Request (all fields optional):**
```json
{
  "name": "Window Table",
  "is_active": false
}
```

**Response `200`:** Updated table object (same shape as above).

---

### DELETE `/api/vendor/{vendorId}/tables/{tableId}`

Deletes a table.

> **Note:** Deletion is blocked if the table has an active order (`pending`, `confirmed`, `preparing`, or `ready`). Returns `409 Conflict` in that case.

**Response `200`:**
```json
{ "message": "Table deleted" }
```

**Response `409`:**
```json
{ "message": "Cannot delete table with active orders" }
```

---

## 2. Table QR Codes

### POST `/api/vendor/{vendorId}/tables/{tableId}/refresh-qr`

Regenerates the QR token for a single table. The old token becomes invalid immediately.

**Response `200`:** Updated table object with new `qrToken`.

---

### POST `/api/vendor/{vendorId}/tables/regenerate-all`

Regenerates QR tokens for **all** tables at once. Use with caution — all physically printed QR codes must be replaced after this operation.

**Response `200`:** Array of all table objects with new `qrToken` values.

> ⚠️ This is a destructive operation. Any customer currently scanning an old QR code will receive a "no longer valid" error.

---

## 3. Table Scan (Public)

### POST `/api/vendor/{vendorId}/tables/{tableId}/scan`

**Public — no authentication required.**

Called by the customer-facing QR landing page when a customer scans a table QR code. Records the scan timestamp.

**Query Parameter (optional):**

| Param | Type | Notes |
|---|---|---|
| `token` | string | If provided, must match the table's current `qr_token`. Returns `410` if expired. |

**Response `200`:**
```json
{
  "message": "Scan recorded",
  "vendorId": "V-ABC12345",
  "tableId": "3",
  "tableName": "Table 3",
  "tableNumber": 3
}
```

**Response `410` (stale QR token):**
```json
{ "message": "This QR code is no longer valid" }
```

---

## 4. Takeaway QR Code

The takeaway QR is a single, persistent QR code per vendor that is **not linked to any table**. It enables customers to place pickup/takeaway orders by scanning it at the entrance, counter, or from printed materials.

### GET `/api/vendor/{vendorId}/tables/takeaway-qr`

Returns the vendor's takeaway QR. Creates one automatically on first request.

**Response `200`:**
```json
{
  "id": "1",
  "qrToken": "b7c2d3e4-f5a6-7890-abcd-ef1234567890",
  "qrCreatedAt": "2026-03-29T10:00:00.000Z",
  "lastRegeneratedAt": null,
  "lastScannedAt": null,
  "isActive": true
}
```

This endpoint is **idempotent** — calling it multiple times always returns the same record without creating duplicates.

---

### POST `/api/vendor/{vendorId}/tables/takeaway-qr/refresh`

Regenerates the takeaway QR token. The old token becomes invalid immediately.

**Response `200`:** Updated takeaway QR object with new `qrToken` and `lastRegeneratedAt` timestamp.

---

## 5. Takeaway Scan (Public)

### POST `/api/vendor/{vendorId}/takeaway/scan`

**Public — no authentication required.**

Called by the customer-facing takeaway landing page. Records a scan against the vendor's takeaway QR record.

**Query Parameter (optional):**

| Param | Type | Notes |
|---|---|---|
| `token` | string | If provided, must match the current `qr_token`. Returns `410` if mismatched. |

**Response `200`:**
```json
{
  "message": "Scan recorded",
  "vendorId": "V-ABC12345",
  "type": "takeaway"
}
```

**Response `410` (stale or missing QR):**
```json
{ "message": "This QR code is no longer valid" }
```

---

## 6. Sync Tables

### POST `/api/vendor/{vendorId}/tables/sync`

Synchronises the number of tables to match a desired count. Useful when a vendor changes the table count in Settings — the frontend calls this to keep the DB in sync.

- If `count` > current tables → creates missing tables (numbered sequentially)
- If `count` < current tables → removes highest-numbered excess tables
- If `count` == current tables → no-op

**Request:**
```json
{ "count": 10 }
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `count` | integer | ✅ | Range: 0 – 500 |

**Response `200`:** Array of all current table objects after sync.

---

## 7. Close Active Table Sessions

### POST `/api/vendor/{vendorId}/tables/{tableId}/close-session`

Closes all active `table_scan_sessions` for one restaurant table. This is the waiter close-table action used after a table visit ends.

**Auth:** Vendor owner/manager or active `waiter` staff token. Other staff roles receive `403`.

**Request:**
```json
{
  "force": true
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `force` | boolean | ❌ | Overrides an unpaid-balance warning only. It never overrides unfinished order items. |

**Response `200`:**
```json
{
  "message": "Table session closed",
  "table": {
    "id": "3",
    "number": 3,
    "name": "Table 3",
    "qrToken": "550e8400-e29b-41d4-a716-446655440000",
    "isActive": true,
    "status": "idle",
    "qrCreatedAt": "2026-03-29T10:00:00.000Z",
    "lastScannedAt": "2026-05-08T12:00:00.000Z"
  },
  "closedSessionIds": ["18", "19"],
  "paymentSummary": {
    "totalAmount": 46,
    "paidAmount": 46,
    "remainingAmount": 0,
    "cashPendingOrders": 0,
    "ordersCount": 2
  }
}
```

**Response `409` unpaid balance:**
```json
{
  "message": "This table still has unpaid balances.",
  "code": "unpaid_balance",
  "paymentSummary": {
    "totalAmount": 46,
    "paidAmount": 20,
    "remainingAmount": 26,
    "cashPendingOrders": 1,
    "ordersCount": 2
  }
}
```

After showing the warning to the waiter, call the same endpoint with `{ "force": true }` to close anyway.

**Notifications on success:** every customer with an active session at the table receives a `session_expire` notification (template `session.closed`, message "Your table session has been closed."), and a visible `table_session_changed` notification is created for the vendor, all active waiters, and all active kitchen staff. No notifications are created on `404`/`409` responses.

**Response `409` unfinished items:**
```json
{
  "message": "This table still has unfinished order items.",
  "code": "unfinished_items",
  "fulfillmentSummary": {
    "unfinishedOrdersCount": 1,
    "unservedItemsCount": 2
  },
  "paymentSummary": {
    "totalAmount": 46,
    "paidAmount": 46,
    "remainingAmount": 0,
    "cashPendingOrders": 0,
    "ordersCount": 1
  }
}
```

Every cart item linked directly through `order_id` or through `shared_order_ids`
must be served before the table can close. Draft and cancelled orders do not
block closure. This conflict cannot be overridden with `force`; staff must serve
the remaining items or explicitly cancel the affected order first.

**Error Responses:**
- `403` — authenticated staff role cannot close tables, or vendor/table mismatch
- `404` — table has no active scan session
- `409` — unpaid balance requiring confirmation, or unfinished order items that must be resolved
- `422` — invalid `force` value

---

## 8. Transfer Active Table Sessions

### POST `/api/vendor/{vendorId}/tables/{tableId}/transfer`

Moves every active scan session on the source table to one empty, active target
table. Existing session IDs remain unchanged, so customer carts, payments,
reviews, and authenticated table sessions continue without requiring another QR
scan. Linked dine-in orders have their denormalized `table_number` updated to the
target table number.

The target must not have any active `table_scan_sessions`. Transfers never merge
two occupied tables.

**Auth:** Vendor owner/manager or active `waiter` staff token. Other staff roles receive `403`.

**Request:**
```json
{
  "target_table_id": 12
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `target_table_id` | integer | ✅ | Database ID of a different, active, empty table belonging to the same vendor |

**Response `200`:**
```json
{
  "message": "Transferred to Table 12.",
  "source_table_id": "7",
  "target_table_id": "12",
  "session_ids": ["31", "32"],
  "sessions_transferred": 2,
  "orders_updated": 3
}
```

The source table's waiter-call alert is cleared and follows the session to the
target table. Customer and operational realtime notifications are emitted so
connected screens can reload the new table assignment.

**Response `409` occupied target:**
```json
{
  "message": "Table 12 is occupied. Select an empty table.",
  "code": "target_table_occupied",
  "target_table_id": "12"
}
```

**Error Responses:**
- `403` — authenticated actor belongs to another vendor or is not an owner/waiter
- `404` — source/target table does not exist, or the source has no active session
- `409` — target table is occupied or inactive
- `422` — `target_table_id` is missing/invalid, or source and target are the same table

---

## 9. Table Status Logic

Status is **computed dynamically** on every `GET /tables` request — it is never stored in the database.

```
IF any active table_scan_sessions for this restaurant_table_id have unpaid orders
    → status = "waiting_payment"   (red)
ELSE IF any active table_scan_sessions exist for this restaurant_table_id
    → status = "active"   (yellow)
ELSE
    → status = "idle"   (green)
```

The lookup is based on `table_scan_sessions.restaurant_table_id`, joined to `orders.table_scan_session_id` for unpaid state. It no longer depends on the stale `orders.table_session_id` or string matching through `orders.table_number`.

---

## 10. Error Reference

| HTTP Status | Meaning |
|---|---|
| `200` | Success |
| `201` | Resource created |
| `401` | Missing or invalid authentication token |
| `403` | Token belongs to a different vendor |
| `404` | Table or vendor not found |
| `409` | Conflict — table has active orders, close-session found unpaid balances, or transfer target is occupied/inactive |
| `410` | QR token is stale / no longer valid |
| `422` | Validation failed (see `errors` in response body) |

---

## Route Summary

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/vendor/{vendorId}/tables` | ✅ | List all tables with status |
| `POST` | `/api/vendor/{vendorId}/tables` | ✅ | Create a table |
| `PATCH` | `/api/vendor/{vendorId}/tables/{tableId}` | ✅ | Update table name / active state |
| `DELETE` | `/api/vendor/{vendorId}/tables/{tableId}` | ✅ | Delete table (blocked if active orders) |
| `POST` | `/api/vendor/{vendorId}/tables/{tableId}/refresh-qr` | ✅ | Refresh QR for one table |
| `POST` | `/api/vendor/{vendorId}/tables/regenerate-all` | ✅ | Regenerate all table QR tokens |
| `POST` | `/api/vendor/{vendorId}/tables/{tableId}/scan` | 🔓 Public | Record table QR scan |
| `GET` | `/api/vendor/{vendorId}/tables/takeaway-qr` | ✅ | Get (or create) takeaway QR |
| `POST` | `/api/vendor/{vendorId}/tables/takeaway-qr/refresh` | ✅ | Regenerate takeaway QR token |
| `POST` | `/api/vendor/{vendorId}/takeaway/scan` | 🔓 Public | Record takeaway QR scan |
| `POST` | `/api/vendor/{vendorId}/tables/sync` | ✅ | Sync table count to desired number |
| `POST` | `/api/vendor/{vendorId}/tables/{tableId}/close-session` | ✅ Waiter/owner | Close active table scan sessions |
| `POST` | `/api/vendor/{vendorId}/tables/{tableId}/transfer` | ✅ Waiter/owner | Move active sessions to an empty table |
