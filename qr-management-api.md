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
7. [Table Status Logic](#7-table-status-logic)
8. [Error Reference](#8-error-reference)

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

## 7. Table Status Logic

Status is **computed dynamically** on every `GET /tables` request — it is never stored in the database.

```
IF any order for this table has status IN (pending, confirmed, preparing, ready)
    → status = "active"   (yellow)
ELSE IF any order for this table has status = "served" AND payment_pending = true
    → status = "waiting_payment"   (red)
ELSE
    → status = "idle"   (green)
```

The `table_number` field on the `orders` table is used for matching (string comparison). Two SQL queries load all active and unpaid table numbers for the vendor in bulk — there is no N+1 query.

---

## 8. Error Reference

| HTTP Status | Meaning |
|---|---|
| `200` | Success |
| `201` | Resource created |
| `401` | Missing or invalid authentication token |
| `403` | Token belongs to a different vendor |
| `404` | Table or vendor not found |
| `409` | Conflict — table has active orders, cannot delete |
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
