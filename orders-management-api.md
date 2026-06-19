# Orders Management API — Documentation

**Base URL:** `/api`  
**Authentication:** `Authorization: Bearer {token}` (vendor owner token or active team member token via Sanctum)  
**Content-Type:** `application/json`

Authenticated endpoints require a token obtained via `POST /api/vendor/login`. Vendor owners/managers can access all vendor routes. Staff tokens are restricted by role: kitchen staff can operate kitchen item states, and waiters can serve items, confirm cash, and close tables.

---

## Table of Contents

1. [Overview & Concepts](#1-overview--concepts)
2. [List Orders (Grouped)](#2-list-orders-grouped)
3. [Get Single Order](#3-get-single-order)
4. [Generic Order Update](#4-generic-order-update)
5. [Waiter Confirm Order](#5-waiter-confirm-order)
6. [Confirm Cash Payment](#6-confirm-cash-payment)
7. [Mark Order Ready](#7-mark-order-ready)
8. [Update Item Status](#8-update-item-status)
9. [Mark Order Picked Up](#9-mark-order-picked-up)
10. [Mark Order Served](#10-mark-order-served)
11. [Cancel Order](#11-cancel-order)
12. [Release Batch to Kitchen](#12-release-batch-to-kitchen)
13. [Fire Next Course](#13-fire-next-course)
14. [Close Legacy Table Session](#14-close-legacy-table-session)
15. [Response Schemas](#15-response-schemas)
16. [Error Reference](#16-error-reference)

---

## 1. Overview & Concepts

### Table Scan Sessions

Dine-in orders are grouped from active `table_scan_sessions`, not the legacy `table_sessions` flow. One physical table can have multiple active scan sessions, so the list endpoint groups active scan sessions by `restaurant_table_id`:

- When a customer scans a QR code and places an order, a session is created (or reused if already active).
- All active scan sessions for the same restaurant table are returned as one table group.
- The group exposes `tableId`, `sessionIds`, guest count, orders, totals, payment state, and kitchen summary.
- The waiter close-table endpoint closes all active scan sessions for that table (`POST /api/vendor/{vendorId}/tables/{tableId}/close-session`).

### Batch Consolidation Window

After the first order arrives at a table, a **90-second batch window** opens. Additional orders placed during this window are held together before being released to the kitchen. This allows a table to order drinks + starters together rather than firing each item separately.

- `batchStartedAt` — when the timer began.
- `batchWindowSeconds` — configurable window (default: 90).
- `batchReleasedAt` — set when the batch is released (manually or after the timer expires).
- `batchOpen` — `true` while the window is still running.

The vendor can release the batch immediately using `POST .../release`.

### Course Sequence

Each session tracks a `currentCourse`. The progression is:

```
drinks → appetizers → mains → desserts
```

Call `POST .../fire-course` to advance to the next course.

### Order Types

| `orderType` | Description |
|---|---|
| `dine-in` | Table service — grouped into sessions |
| `takeaway` | Collection orders — not grouped into sessions |

### Order Statuses

| Status | Description |
|---|---|
| `pending` | Order placed, awaiting confirmation |
| `confirmed` | Confirmed by waiter or system |
| `preparing` | In the kitchen |
| `ready` | Ready for pickup / serving |
| `delivered` | Delivered to customer |
| `picked_up` | Customer collected (takeaway) |
| `served` | Served at the table (dine-in) |
| `cancelled` | Cancelled |

---

## 2. List Orders (Grouped)

### `GET /api/vendor/{vendorId}/orders`

Returns orders split into two groups:
- **`sessions`** — active dine-in table groups built from `table_scan_sessions`, grouped by `restaurant_table_id`.
- **`takeaway`** — takeaway/collect orders not linked to a session.

**Query Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `status` | `string` | Filter orders by status (e.g. `pending`, `ready`) |
| `orderType` | `string` | Filter takeaway orders by type (`takeaway`, etc.) |

**Response `200`:**
```json
{
  "sessions": [
    {
      "sessionId": "table-3",
      "tableId": "3",
      "sessionIds": ["18", "19"],
      "vendorId": "5",
      "tableNumber": 3,
      "tableName": "Table 3",
      "status": "active",
      "guestCount": 2,
      "totalAmount": 28.50,
      "paidAmount": 0,
      "paymentStatus": "cash_pending",
      "cashPending": true,
      "closedAt": null,
      "kitchenSummary": {
        "total": 3,
        "pending": 1,
        "draft": 0,
        "confirmed": 1,
        "preparing": 1,
        "ready": 0,
        "served": 0,
        "cancelled": 0
      },
      "orders": [ { /* order object */ } ],
      "createdAt": "2026-03-29T12:00:00.000Z",
      "updatedAt": "2026-03-29T12:01:00.000Z"
    }
  ],
  "takeaway": [
    { /* order object */ }
  ]
}
```

---

## 3. Get Single Order

### `GET /api/vendor/{vendorId}/orders/{orderId}`

Returns the full detail of one order. `{orderId}` may be the numeric `id` or the `orderPublicId` (UUID).

**Response `200`:** Returns a single [Order Object](#order-object).

---

## 4. Generic Order Update

### `PATCH /api/orders/{orderId}`

Update status or payment fields on any order. Use the dedicated endpoints for common actions (ready, served, cancel) instead of this when possible.

**Request Body (all fields optional):**
```json
{
  "status": "preparing",
  "paymentPending": false,
  "paymentReceived": true,
  "paymentNote": "Paid by card"
}
```

| Field | Type | Allowed Values |
|---|---|---|
| `status` | `string` | `pending`, `confirmed`, `preparing`, `ready`, `delivered`, `picked_up`, `cancelled` |
| `paymentPending` | `boolean` | — |
| `paymentReceived` | `boolean` | Setting to `true` also sets `paymentConfirmedAt` |
| `paymentNote` | `string\|null` | Max 500 chars |

**Response `200`:** Returns the updated [Order Object](#order-object).

---

## 5. Waiter Confirm Order

### `PATCH /api/orders/{orderId}/confirm`

Confirms a pending order (typically for non-prepaid / walk-in orders). Sets `status → confirmed`, `waiterConfirmed → true`, and records `waiterConfirmedAt`.

**Request Body:** none

**Response `200`:**
```json
{
  "status": "confirmed",
  "waiterConfirmed": true,
  "waiterConfirmedAt": "2026-03-29T12:05:00.000Z"
}
```

---

## 6. Confirm Cash Payment

### `PATCH /api/orders/{orderId}/confirm-cash`

Records that cash payment has been received. Sets `paymentReceived → true`, `paymentPending → false`, and records `paymentConfirmedAt`.

**Request Body:** none

**Response `200`:**
```json
{
  "paymentReceived": true,
  "paymentPending": false,
  "paymentConfirmedAt": "2026-03-29T12:10:00.000Z"
}
```

---

## 7. Mark Order Ready

### `PATCH /api/orders/{orderId}/ready`

Marks the order as ready for pickup or serving. Sets `status → ready` and stamps `cart_items.ready_at = now()` on every cart_item linked to this order (owned by the order's session, plus any cart_item whose `shared_order_ids` JSON contains the order's id). The order-level `readyAt` returned in the response is the latest such timestamp once *every* linked cart_item has been marked ready; otherwise `null`.

**Request Body:** none

**Response `200`:** Returns the updated [Order Object](#order-object) with `status: "ready"`.

---

## 8. Update Item Status

### `PATCH /api/vendor/orders/{orderId}/items/{cartItemId}`

Updates one cart item linked to the order. This is the canonical endpoint used by the KDS and waiter pages.

**Auth:** Vendor owner/manager, kitchen staff, or waiter staff.

Role restrictions:
- `kitchen` staff may set `new`, `preparing`, or `ready`.
- `waiter` staff may set only `served`.
- Vendor owner/manager may set any allowed status.

Item status is computed from `cart_items` timestamps:

| API `status` | Timestamp effect | Returned item `status` |
|---|---|---|
| `new` | clears `preparing_start_at`, `ready_at`, `served_at` | `new` |
| `preparing` | sets `preparing_start_at` | `in_progress` |
| `ready` | sets `preparing_start_at` and `ready_at` | `ready` |
| `served` | sets `preparing_start_at`, `ready_at`, and `served_at` | `served` |

### Request Body
```json
{
  "status": "ready"
}
```

Allowed values: `new`, `preparing`, `ready`, `served`.

### Response `200`
Returns the updated [Order Object](#order-object).

### Error Responses
- `403` — staff role is not allowed to set the requested state
- `404` — cart item is not linked to this order
- `422` — invalid status

---

## 9. Mark Order Picked Up

### `PATCH /api/orders/{orderId}/picked-up`

Marks a takeaway order as collected. Sets `status → picked_up`. (No timestamp is recorded — pickup is reflected by the status alone.)

**Request Body:** none

**Response `200`:** Returns the updated [Order Object](#order-object) with `status: "picked_up"`.

---

## 10. Mark Order Served

### `PATCH /api/orders/{orderId}/served`

Marks a dine-in order as served. Sets `status → served` and records `servedAt`.

**Request Body:** none

**Response `200`:** Returns the updated [Order Object](#order-object) with `status: "served"`.

---

## 11. Cancel Order

### `PATCH /api/orders/{orderId}/cancel`

Cancels an order. Sets `status → cancelled`, records `cancelledAt`, and optionally stores a reason.

**Request Body (optional):**
```json
{
  "reason": "Customer changed their mind"
}
```

| Field | Type | Description |
|---|---|---|
| `reason` | `string\|null` | Optional cancellation reason (max 500 chars) |

**Response `200`:** Returns the updated [Order Object](#order-object) with `status: "cancelled"`.

---

## 12. Release Batch to Kitchen

### `POST /api/vendor/{vendorId}/sessions/{sessionId}/release`

Immediately releases the current batch to the kitchen, overriding the batch window timer. Sets `batchReleasedAt` and marks `batchOpen → false`.

**Request Body:** none

**Response `200`:** Returns the updated legacy session object with `batchOpen: false`.

---

## 13. Fire Next Course

### `POST /api/vendor/{vendorId}/sessions/{sessionId}/fire-course`

Advances the session to the next course in the sequence:

```
drinks → appetizers → mains → desserts
```

Returns `422` if already on `desserts`.

**Request Body:** none

**Response `200`:** Returns the updated legacy session object with the new `currentCourse`.

**Response `422`:**
```json
{
  "message": "Already on the last course (desserts)."
}
```

---

## 14. Close Legacy Table Session

### `POST /api/vendor/{vendorId}/sessions/{sessionId}/close`

Closes a legacy `table_sessions` record. New waiter flows should use the table-scan endpoint documented in `documentation/qr-management-api.md`:

`POST /api/vendor/{vendorId}/tables/{tableId}/close-session`

This legacy endpoint sets `status → closed` and records `closedAt` on the requested legacy session.

**Request Body:** none

**Response `200`:** Returns the updated legacy session object with `status: "closed"`.

---

## 15. Response Schemas

### Order Object

```json
{
  "id": "42",
  "orderPublicId": "ord-abc123",
  "orderNumber": "#9001",
  "orderType": "dine-in",
  "tableNumber": "3",
  "tableId": "3",
  "tableScanSessionId": "18",
  "course": "mains",
  "waiterConfirmed": true,
  "waiterConfirmedAt": "2026-03-29T12:05:00.000Z",
  "customer": {
    "id": "10",
    "name": "Jane Smith",
    "email": "jane@example.com",
    "phone": "+43 660 1234567"
  },
  "status": "preparing",
  "itemsCount": 3,
  "items": [
    {
      "cartItemId": 17,
      "menuItemId": 42,
      "name": "Schnitzel",
      "imageUrl": null,
      "quantity": 1,
      "notes": null,
      "unitPrice": 18.50,
      "lineTotal": 18.50,
      "status": "in_progress",
      "sharedBetween": 1,
      "sharedWithOrderIds": [],
      "preparingStartAt": "2026-03-29T12:03:00.000Z",
      "readyAt": null,
      "servedAt": null
    }
  ],
  "amount": 18.50,
  "serviceFee": 1.50,
  "vatAmount": 2.00,
  "currency": "EUR",
  "paymentMethod": "cash",
  "paymentPending": false,
  "paymentReceived": true,
  "paymentConfirmedAt": "2026-03-29T12:10:00.000Z",
  "paymentNote": null,
  "readyAt": null,
  "servedAt": null,
  "cancelledAt": null,
  "cancelledReason": null,
  "timeline": [
    { "status": "received", "timestamp": "2026-03-29T12:00:00.000Z" }
  ],
  "createdAt": "2026-03-29T12:00:00.000Z",
  "updatedAt": "2026-03-29T12:05:00.000Z"
}
```

**Notes:**
- `itemsCount` is computed live as the sum of `quantity` across linked cart_items.
- `items[]` is built live from `cart_items` (owned by the order's session, plus any cart_item whose `shared_order_ids` JSON contains the order id).
- Item `status` is derived from `served_at`, `ready_at`, and `preparing_start_at`: `served`, `ready`, `in_progress`, or `new`.
- `readyAt` reflects the order-level rollup (latest timestamp once *all* linked cart_items have `ready_at` set). The per-item `readyAt` is the canonical value.
- `pickedUpAt` and `guestCount` are no longer returned — both columns were dropped from `orders`.

### Table Scan Session Group Object

```json
{
  "sessionId": "table-3",
  "tableId": "3",
  "sessionIds": ["18", "19"],
  "vendorId": "5",
  "tableNumber": 3,
  "tableName": "Table 3",
  "status": "active",
  "guestCount": 2,
  "totalAmount": 46.00,
  "paidAmount": 20.00,
  "paymentStatus": "partial",
  "cashPending": false,
  "closedAt": null,
  "kitchenSummary": {
    "total": 4,
    "pending": 0,
    "confirmed": 1,
    "preparing": 2,
    "ready": 1,
    "served": 0,
    "cancelled": 0
  },
  "orders": [ { /* Order Object */ } ],
  "createdAt": "2026-03-29T12:00:00.000Z",
  "updatedAt": "2026-03-29T12:05:00.000Z"
}
```

---

## 15.1 Customer Notifications

All vendor order actions that modify order or cart item state automatically create notifications for all customers with active sessions at the same table. This enables the customer app to poll for updates.

| Vendor Action | Event | Message |
|---|---|---|
| Generic update (`PATCH /api/orders/{orderId}`) | `order_updated` | Your order status has been updated. |
| Waiter confirm | `order_updated` | Your order has been confirmed by the waiter. |
| Confirm cash payment | `payment_updated` | Your cash payment has been confirmed. |
| Mark ready | `order_updated` | Your order is ready! |
| Update item status | `cart_item_updated` | {Item name} is now being prepared / is ready / has been served. |
| Mark picked up | `order_updated` | Your order has been picked up. |
| Mark served | `order_updated` | Your order has been served. Enjoy! |
| Cancel order | `order_updated` | Your order has been cancelled. |

Customers can retrieve notifications via `GET /api/customer/notifications` (see Customer API docs §4.10).

---

## 16. Error Reference

| HTTP Code | Meaning |
|---|---|
| `401 Unauthorized` | Missing or invalid bearer token |
| `403 Forbidden` | Token belongs to a different vendor |
| `404 Not Found` | Order or session does not exist or belongs to another vendor |
| `422 Unprocessable Entity` | Validation failed (e.g. firing next course when already on desserts) |
| `500 Internal Server Error` | Unexpected server error |

---

## Route Summary

| Method | Path | Action |
|---|---|---|
| `GET` | `/api/vendor/{vendorId}/orders` | List all orders (grouped) |
| `GET` | `/api/vendor/{vendorId}/orders/{orderId}` | Get single order |
| `PATCH` | `/api/orders/{orderId}` | Generic update (status / payment) |
| `PATCH` | `/api/orders/{orderId}/confirm` | Waiter confirm order |
| `PATCH` | `/api/orders/{orderId}/confirm-cash` | Confirm cash payment received |
| `PATCH` | `/api/orders/{orderId}/ready` | Mark order ready |
| `PATCH` | `/api/vendor/orders/{orderId}/items/{cartItemId}` | Update cart item status for KDS/waiter |
| `PATCH` | `/api/orders/{orderId}/picked-up` | Mark order picked up (takeaway) |
| `PATCH` | `/api/orders/{orderId}/served` | Mark order served (dine-in) |
| `PATCH` | `/api/orders/{orderId}/cancel` | Cancel order |
| `POST` | `/api/vendor/{vendorId}/sessions/{sessionId}/release` | Release batch to kitchen now |
| `POST` | `/api/vendor/{vendorId}/sessions/{sessionId}/fire-course` | Advance to next course |
| `POST` | `/api/vendor/{vendorId}/sessions/{sessionId}/close` | Close legacy table session |
