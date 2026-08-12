# Orders Management API — Documentation

**Base URL:** `/api`  
**Authentication:** `Authorization: Bearer {token}` (vendor owner token or active team member token via Sanctum)  
**Content-Type:** `application/json`

Authenticated endpoints require a token obtained via `POST /api/vendor/login`. Vendor owners/managers can access all vendor routes. Staff tokens are restricted by role: kitchen staff can operate kitchen item states, and waiters can serve items, confirm cash, close tables, browse the menu (`GET /api/vendor/menu/items`), start table sessions, and place staff orders.

---

## Table of Contents

1. [Overview & Concepts](#1-overview--concepts)
   - [Off-premise payment and kitchen release](#off-premise-payment-and-kitchen-release)
   - [Redis-first TeamMember mutations](#redis-first-teammember-mutations)
   - [Get staff command status](#get-staff-command-status)
2. [List Orders (Grouped)](#2-list-orders-grouped)
3. [Get Single Order](#3-get-single-order)
4. [Generic Order Update](#4-generic-order-update)
5. [Waiter Confirm Order](#5-waiter-confirm-order)
6. [Confirm Cash Payment](#6-confirm-cash-payment)
7. [Mark Order Ready](#7-mark-order-ready)
8. [Update Item Status](#8-update-item-status)
   - [Batch item status commands](#batch-item-status-commands)
   - [Serve multiple ready items in one order](#serve-multiple-ready-items-in-one-order)
9. [Mark Order Picked Up](#9-mark-order-picked-up)
10. [Mark Order Served](#10-mark-order-served)
11. [Cancel Order](#11-cancel-order)
12. [Release Batch to Kitchen](#12-release-batch-to-kitchen)
13. [Fire Next Course](#13-fire-next-course)
14. [Close Legacy Table Session](#14-close-legacy-table-session)
15. [Response Schemas](#15-response-schemas)
16. [Realtime Notifications](#16-realtime-notifications)
17. [Error Reference](#17-error-reference)
18. [Start Table Session (Staff)](#18-start-table-session-staff)
19. [Place Staff Order](#19-place-staff-order)
20. [Order History (Paginated)](#20-order-history-paginated)
21. [Order Receipt](#21-order-receipt)

---

## 1. Overview & Concepts

### Table Scan Sessions

Dine-in orders are grouped from active `table_scan_sessions`, not the legacy `table_sessions` flow. One physical table can have multiple active scan sessions, so the list endpoint groups active scan sessions by `restaurant_table_id`:

- When a customer scans a QR code and places an order, a session is created (or reused if already active).
- All active scan sessions for the same restaurant table are returned as one table group.
- The group exposes `tableId`, `sessionIds`, guest count, orders, totals, payment state, and kitchen summary.
- The waiter close-table endpoint closes all active scan sessions for that table (`POST /api/vendor/{vendorId}/tables/{tableId}/close-session`).

### Batch Consolidation Window [Legacy]

> **Deprecation Notice:** Batch consolidation only applies to the legacy `table_sessions` model. Orders placed through the current QR-scan flow (`table_scan_sessions`) are sent directly to the kitchen with no batching. These endpoints will be removed in a future release once the feature is re-implemented on the new model.

After the first order arrives at a table, a **90-second batch window** opens. Additional orders placed during this window are held together before being released to the kitchen. This allows a table to order drinks + starters together rather than firing each item separately.

- `batchStartedAt` — when the timer began.
- `batchWindowSeconds` — configurable window (default: 90).
- `batchReleasedAt` — set when the batch is released (manually or after the timer expires).
- `batchOpen` — `true` while the window is still running.

The vendor can release the batch immediately using `POST .../release`.

### Course Sequence [Legacy]

> **Deprecation Notice:** Course management only applies to the legacy `table_sessions` model. Orders placed through the current QR-scan flow (`table_scan_sessions`) do not support course sequencing. These endpoints will be removed in a future release once the feature is re-implemented on the new model.

Each session tracks a `currentCourse`. The progression is:

```
drinks → appetizers → mains → desserts
```

Call `POST .../fire-course` to advance to the next course.

### Order Types

| `orderType` | Description |
|---|---|
| `dine-in` | Table service — grouped into sessions |
| `pickup` | Public-restaurant pickup; ASAP or scheduled, with a shared PIN group |
| `takeaway` | Takeaway-QR collection; always ASAP, with a shared PIN group |

### Order Statuses

| Status | Description |
|---|---|
| `draft` | Cart confirmed by the customer but not yet submitted/paid |
| `confirmed` | Confirmed by the dine-in flow, or paid off-premise |
| `waiter_confirmed` | Explicitly confirmed by waiter |
| `in_progress` | At least one kitchen action has started |
| `picked_up` | Customer collected (pickup or takeaway) |
| `served` | Served at the table (dine-in) |
| `cancelled` | Cancelled |

### Off-premise payment and kitchen release

Pickup and takeaway use the same order model and operational endpoints as dine-in, but are payment-gated:

- Customer cart confirmation creates a `draft`; it is not an actionable kitchen order.
- Stripe success or vendor/waiter cash confirmation changes every covered off-premise draft to `confirmed` and marks it paid in the same transaction.
- Vendors and waiters receive the paid order immediately.
- A paid ASAP pickup/takeaway receives `kitchenReleasedAt` immediately and enters the normal kitchen queue.
- A paid scheduled pickup remains visible to kitchen in the separate **Pickups** view while `kitchenReleasedAt` is `null`. It enters the normal preparation queue 20 minutes before `scheduledFor`.
- `kitchenReleasedAt` is a visibility/release timestamp, not a replacement for item states such as `in_progress` or `ready`.

The scheduler must run `php artisan kitchen-orders:release-scheduled` every minute. Release is lock-protected and idempotent, so retrying the command or processing the same payment event does not generate a second kitchen release.

### Redis-first TeamMember mutations

Operational mutations have two response modes:

| Authenticated actor | Execution | Immediate response |
|---|---|---|
| Vendor owner (`Vendor`) | The controller performs the mutation synchronously. | Existing domain response (`200`, `201`, `404`, `409`, or `422`). `Idempotency-Key` is not required. |
| Active waiter or kitchen account (`TeamMember`) | Redis atomically reserves and sequences a command, then the `staffcommands` worker performs the mutation. | `202 Accepted` after enqueueing. The response is an acknowledgement, not the domain result. |

The Redis-first path applies to the role-authorized TeamMember actions below:

| Endpoint | Allowed TeamMember | Command operation |
|---|---|---|
| `PATCH /api/vendor/orders/{orderId}/confirm` | Waiter | `order.confirm` |
| `PATCH /api/vendor/orders/{orderId}/confirm-cash` | Waiter | `order.confirm_cash` |
| `PATCH /api/vendor/orders/{orderId}/ready` | Kitchen | `order.ready` |
| `PATCH /api/vendor/orders/{orderId}/items/{cartItemId}` | Kitchen: `new`, `preparing`, `ready`; waiter: `served` | `order.item_status` |
| `PATCH /api/vendor/orders/{orderId}/items/serve-ready` | Waiter | `order.items_serve_ready` |
| `PATCH /api/vendor/orders/{orderId}/picked-up` | Waiter | `order.picked_up` |
| `PATCH /api/vendor/orders/{orderId}/served` | Waiter | `order.served` |
| `PATCH /api/vendor/notifications/{id}/read` | Waiter or kitchen | `notification.read` |
| `POST /api/vendor/notifications/read-all` | Waiter or kitchen | `notification.read_all` |
| Table/session/staff-order mutations | Waiter | See `qr-management-api.md` and sections 18–19 below. |

Every non-batch TeamMember request in that table must send a UUID in the header:

```http
Idempotency-Key: 0190f26e-7c87-7def-8e46-400000000001
```

A successful reservation returns the same shape for every operation:

```json
{
  "command_id": "0190f26e-7c87-7def-8e46-300000000001",
  "idempotency_key": "0190f26e-7c87-7def-8e46-400000000001",
  "operation": "order.item_status",
  "status": "accepted",
  "status_url": "/api/vendor/commands/0190f26e-7c87-7def-8e46-300000000001"
}
```

The worker revalidates the actor, vendor, role, and role-specific operation before mutating data in
a database transaction. Therefore `202` means only that the command was durably reserved and
enqueued. Do not treat it as proof that an order, item, table, or notification changed.

#### Rapid bursts, ordering, and retries

- Idempotency keys are scoped to the authenticated TeamMember. Retrying the same canonical command
  with the same key returns the existing `command_id` and does not enqueue a second mutation.
- Reusing that actor's key for a different operation, payload, vendor, or resource returns `409`
  with `code: "idempotency_key_reused"` and the original `command_id`.
- Commands receive monotonic sequences for every affected resource. Commands touching the same
  order, table, or actor notification feed execute in request-reservation order. Commands on
  unrelated resources may run concurrently; a table transfer is ordered against both tables.
- The worker also uses resource locks and a persistent `staff_commands` audit row, so queue retries
  cannot repeat an already completed mutation.
- Conditions may change after acceptance. For example, a later item-status command can finish with
  `409` because an earlier command already advanced the item. Read the terminal command response
  instead of assuming every accepted burst succeeded.
- If Redis or queue dispatch is unavailable after async commands are enabled, the API fails closed
  with `503` and `code: "staff_commands_unavailable"`; it does not silently execute the mutation in
  the request.

#### Get staff command status

**GET** `/api/vendor/commands/{commandId}`  
**Authentication:** Required; active `TeamMember` token belonging to the actor who created the
command. Vendor-owner tokens receive `403`; another staff actor receives `404`.  
**Request body:** None

While queued or running, `status` is `accepted` or `processing`:

```json
{
  "command_id": "0190f26e-7c87-7def-8e46-300000000001",
  "idempotency_key": "0190f26e-7c87-7def-8e46-400000000001",
  "team_member_id": 19,
  "vendor_id": 7,
  "actor_role": "kitchen",
  "operation": "order.item_status",
  "status": "processing",
  "resources": ["vendor:7:order:41"],
  "resource_sequences": {
    "vendor:7:order:41": 18
  }
}
```

A successful terminal status preserves the original controller HTTP response in `response`:

```json
{
  "command_id": "0190f26e-7c87-7def-8e46-300000000001",
  "idempotency_key": "0190f26e-7c87-7def-8e46-400000000001",
  "team_member_id": 19,
  "vendor_id": 7,
  "actor_role": "kitchen",
  "operation": "order.item_status",
  "status": "completed",
  "resources": ["vendor:7:order:41"],
  "resource_sequences": {
    "vendor:7:order:41": 18
  },
  "http_status": 200,
  "response": {
    "id": "41",
    "status": "in_progress",
    "items": [{ "cartItemId": 82, "status": "in_progress" }]
  },
  "error": null,
  "processed_at": "2026-07-17T12:00:01.000000Z"
}
```

Expected domain errors are terminal command failures rather than polling-endpoint HTTP errors. The
poll itself still returns `200`; inspect `status`, `http_status`, `response`, and `error`:

```json
{
  "command_id": "0190f26e-7c87-7def-8e46-300000000002",
  "idempotency_key": "0190f26e-7c87-7def-8e46-400000000002",
  "team_member_id": 19,
  "vendor_id": 7,
  "actor_role": "kitchen",
  "operation": "order.item_status",
  "status": "failed",
  "resources": ["vendor:7:order:41", "vendor:7:table:12"],
  "resource_sequences": {
    "vendor:7:order:41": 19,
    "vendor:7:table:12": 24
  },
  "http_status": 409,
  "response": {
    "message": "Item status has already advanced.",
    "current_status": "served",
    "requested_status": "ready"
  },
  "error": "Item status has already advanced.",
  "processed_at": "2026-07-17T12:00:02.000000Z"
}
```

The initiating actor also receives a silent `.notification.created` Pusher event named
`staff_command_completed` or `staff_command_failed`. Its metadata contains `command_id`,
`command_status`, `operation`, `resources`, `http_status`, `response`, and `error`. Use that event as
a prompt to reconcile the status URL. It is targeted only to the initiating waiter/kitchen channel,
is marked read, and does not appear in the visible notification list or unread count.

---

## 2. List Orders (Grouped)

### `GET /api/vendor/{vendorId}/orders`

Returns orders split into two groups:
- **`sessions`** — active dine-in table groups built from `table_scan_sessions`, grouped by `restaurant_table_id`.
- **`takeaway`** — the flat off-premise collection containing both `pickup` and `takeaway` orders. These orders can still belong to customer PIN-group `table_scan_sessions`; they are not grouped as physical tables in this response.

**Query Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `status` | `string` | Filter orders by persisted status (e.g. `confirmed`, `in_progress`, `picked_up`) |
| `orderType` | `string` | Filter by type: `dine-in`, `pickup`, or `takeaway` |

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
    {
      "id": "42",
      "orderType": "pickup",
      "orderMode": "pickup",
      "status": "confirmed",
      "paymentReceived": true,
      "scheduledFor": "2026-08-10T19:30:00.000000Z",
      "kitchenReleasedAt": null,
      "pickupTime": "2026-08-10T19:30:00.000000Z",
      "pickupStatus": "pending"
    }
  ]
}
```

Actor-specific visibility:

- Vendor owners and waiters receive submitted off-premise orders immediately. Paid orders are confirmed; a cash-requested order remains a visible `draft` with `paymentPending: true` until cash is confirmed.
- Kitchen never receives an unpaid draft.
- Kitchen receives a paid future scheduled `pickup` with `kitchenReleasedAt: null` so it can be displayed in the separate **Pickups** tab. The client sorts that tab by `scheduledFor` day/time and treats unreleased rows as read-only.
- The kitchen's normal **New/Ready** queues include an off-premise order only after `kitchenReleasedAt` is set. These cards display `Pickup at {pickupTime}`; after release their preparation behavior is identical to dine-in.
- Takeaway is always ASAP, so a paid takeaway is released immediately rather than appearing as a future scheduled pickup.

---

## 3. Get Single Order

### `GET /api/vendor/{vendorId}/orders/{orderId}`

Returns the full detail of one order. `{orderId}` may be the numeric `id` or the `orderPublicId` (UUID).

**Response `200`:** Returns a single [Order Object](#order-object).

---

## 4. Generic Order Update

### `PATCH /api/vendor/orders/{orderId}`

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
| `status` | `string` | `draft`, `confirmed`, `waiter_confirmed`, `in_progress`, `served`, `picked_up`, `cancelled` |
| `paymentPending` | `boolean` | — |
| `paymentReceived` | `boolean` | Setting to `true` also sets `paymentConfirmedAt` |
| `paymentNote` | `string\|null` | Max 500 chars |

**Response `200`:** Returns the updated [Order Object](#order-object).

---

## 5. Waiter Confirm Order

### `PATCH /api/vendor/orders/{orderId}/confirm`

Confirms a pending order (typically for non-prepaid / walk-in orders). Sets `status → confirmed`, `waiterConfirmed → true`, and records `waiterConfirmedAt`.

**Request Body (optional):**

```json
{ "paymentNote": "Cash collected at the table" }
```

`paymentNote` is nullable text up to 500 characters.

**Execution:** A vendor owner receives the synchronous `200` response below. An active waiter must
send a UUID `Idempotency-Key` header and receives the shared `202` staff-command acknowledgement;
poll its `status_url` for the eventual controller response.

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

### `PATCH /api/vendor/orders/{orderId}/confirm-cash`

Records that cash payment has been received. Sets `paymentReceived → true`, `paymentPending → false`, records `paymentConfirmedAt`, and sets `paymentMethod → cash`. Waiters can collect any unpaid order this way, even without a customer cash request or when a different payment method had previously been selected.

For pickup/takeaway, a customer cash request leaves the covered orders as drafts. Confirming cash atomically marks every order covered by that cash payment as paid and changes each off-premise draft to `confirmed`. Each confirmed order is then passed through the same ASAP/scheduled kitchen-release rules described above. This is the only cash-payment step that makes those orders actionable by the kitchen.

**Request Body (optional):**

```json
{
  "tipAmount": 5.00,
  "paymentNote": "Cash collected at the table"
}
```

`tipAmount` must be between `0` and `999999.99`. It is stored separately from the order's base
`amount`, matching customer card-payment tip accounting.

**Execution:** A vendor owner receives the synchronous `200` response below. An active waiter must
send a UUID `Idempotency-Key` header and receives `202`; payment is not confirmed until the command
finishes with `status: "completed"`.

**Response `200`:**
```json
{
  "paymentReceived": true,
  "paymentPending": false,
  "tip": 5.00,
  "paymentConfirmedAt": "2026-03-29T12:10:00.000Z"
}
```

---

## 7. Mark Order Ready

### `PATCH /api/vendor/orders/{orderId}/ready`

Marks every linked item ready for pickup or serving. It stamps `cart_items.ready_at = now()` (owned by the order's session, plus any cart item whose `shared_order_ids` contains the order ID), ensures the persisted order is `in_progress`, and records `in_progress_at`. The returned order-level `readyAt` is the latest item timestamp once *every* linked item is ready; `pickupStatus` is then `ready`.

**Request Body:** none

**Execution:** A vendor owner receives the synchronous `200` response below. Active kitchen staff
must send a UUID `Idempotency-Key` header and receive `202`; use the command status for the domain
result.

**Response `200`:** Returns the updated [Order Object](#order-object) with `status: "in_progress"`, every item `status: "ready"`, and `pickupStatus: "ready"` for off-premise.

For pickup/takeaway, the endpoint returns `409` if payment is not confirmed or `kitchenReleasedAt` is still `null`. A future order shown in the kitchen's **Pickups** tab cannot be prepared early through this endpoint.

---

## 8. Update Item Status

### `PATCH /api/vendor/orders/{orderId}/items/{cartItemId}`

Updates one cart item linked to the order. This is the canonical endpoint used by the KDS and waiter pages.

**Auth:** Vendor owner/manager, kitchen staff, or waiter staff.

Role restrictions:
- `kitchen` staff may set `new`, `preparing`, or `ready`.
- `waiter` staff may set only `served`.
- Vendor owner/manager may set any allowed status.

**Execution:** The vendor owner receives the synchronous response documented below. An active
kitchen/waiter TeamMember must send a UUID `Idempotency-Key` header and receives `202`. For a staff
command, `404`/`409` responses discovered during execution appear inside the terminal command's
`http_status` and `response` fields.

Item status is computed from `cart_items` timestamps:

| API `status` | Timestamp effect | Returned item `status` |
|---|---|---|
| `new` | no-op while the item is still `new` or `received`; never clears progress | `new` or `received` |
| `preparing` | sets `preparing_start_at` | `in_progress` |
| `ready` | sets `preparing_start_at` and `ready_at` | `ready` |
| `served` | sets `preparing_start_at`, `ready_at`, and `served_at` | `served` |

Status changes are forward-only: `new/received → preparing → ready → served`.
Direct forward jumps are allowed. Repeating the current status is an idempotent
`200` response and does not create duplicate notifications. A request that would
move an item backward is rejected with `409 Conflict` and does not alter any
timestamps.

For pickup/takeaway, item mutations also return `409` while the order is unpaid/draft or while `kitchenReleasedAt` is `null`. Once released, the same forward-only kitchen state machine applies as for dine-in.

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
- `409` — the item has already advanced beyond the requested state
- `422` — invalid status

### Response `409`
```json
{
  "message": "Item status has already advanced.",
  "current_status": "ready",
  "requested_status": "new"
}
```

### Batch item status commands

**POST** `/api/vendor/orders/items/status-batch`  
**Authentication:** Required; active waiter or kitchen `TeamMember` only. Vendor-owner tokens and
other staff roles receive `403`.  
**Content-Type:** `application/json`

This endpoint is optimized for rapid KDS/waiter bursts. It accepts 1–50 item transitions and
dispatches each entry as an independent ordered `order.item_status` command. It is always async and
requires `STAFF_ASYNC_COMMANDS_ENABLED=true`.

The standard `Idempotency-Key` request header is not used for this endpoint. Every entry must carry
its own distinct UUID `idempotency_key` so it can be retried and reconciled independently.

**Request body:**

```json
{
  "commands": [
    {
      "idempotency_key": "0190f26e-7c87-7def-8e46-400000000011",
      "order_id": "ord-a1b2c3d4e5f6",
      "cart_item_id": 82,
      "status": "preparing"
    },
    {
      "idempotency_key": "0190f26e-7c87-7def-8e46-400000000012",
      "order_id": "ord-a1b2c3d4e5f6",
      "cart_item_id": 83,
      "status": "ready"
    }
  ]
}
```

| Field | Type | Rules |
|---|---|---|
| `commands` | array | Required; 1–50 entries. |
| `commands.*.idempotency_key` | UUID string | Required and distinct within this request. Scoped to the authenticated TeamMember across requests. |
| `commands.*.order_id` | string | Required; numeric ID or public order ID, maximum 64 characters. |
| `commands.*.cart_item_id` | integer | Required; item must be linked to the specified order. |
| `commands.*.status` | string | Kitchen: `new`, `preparing`, `ready`; waiter: `served`. |

**Response `202 Accepted`:**

```json
{
  "commands": [
    {
      "command_id": "0190f26e-7c87-7def-8e46-300000000011",
      "idempotency_key": "0190f26e-7c87-7def-8e46-400000000011",
      "operation": "order.item_status",
      "status": "accepted",
      "status_url": "/api/vendor/commands/0190f26e-7c87-7def-8e46-300000000011"
    },
    {
      "command_id": "0190f26e-7c87-7def-8e46-300000000012",
      "idempotency_key": "0190f26e-7c87-7def-8e46-400000000012",
      "operation": "order.item_status",
      "status": "accepted",
      "status_url": "/api/vendor/commands/0190f26e-7c87-7def-8e46-300000000012"
    }
  ]
}
```

The API validates the full batch shape and every role/status combination before it starts
dispatching. Order access and item linkage are deliberately checked by each worker so the HTTP
request can stay Redis-first. Those domain checks therefore appear as terminal `404` command
results. After dispatch begins, an enqueue or idempotency conflict can occur partway through. In
that case, `accepted_commands` contains commands already reserved; keep tracking them and retry only
entries that were not accepted.

**Response `409 Conflict` (a key was reused for a different command):**

```json
{
  "message": "This idempotency key was already used for a different staff command.",
  "code": "idempotency_key_reused",
  "command_id": "0190f26e-7c87-7def-8e46-300000000099",
  "accepted_commands": [
    {
      "command_id": "0190f26e-7c87-7def-8e46-300000000011",
      "idempotency_key": "0190f26e-7c87-7def-8e46-400000000011",
      "operation": "order.item_status",
      "status": "accepted",
      "status_url": "/api/vendor/commands/0190f26e-7c87-7def-8e46-300000000011"
    }
  ]
}
```

**Other errors:**

- `403` — owner token, unsupported staff role, or a waiter/kitchen status outside its allowed set.
- Terminal `404` — an order is outside the actor's vendor or an item is not linked to its order;
  inspect each command's `http_status` and `response`.
- `422` — malformed body, more than 50 commands, duplicate/malformed UUIDs, or invalid fields.
- `503` — async staff commands are disabled/unavailable, or Redis/queue dispatch failed. A dispatch
  failure includes `accepted_commands`, which may be empty.

Each returned status URL is polled independently. Entries that share an order or table are executed
in their reserved sequence even if queue workers receive them out of order.

### Serve multiple ready items in one order

**PATCH** `/api/vendor/orders/{orderId}/items/serve-ready`

**Authentication:** Required; vendor owner or active waiter `TeamMember`. Kitchen staff receive
`403`.

**Content-Type:** `application/json`

Marks 1–50 selected ready items served in one database transaction, one order-sequenced command,
and one customer/operational realtime update. This is the endpoint used by the waiter page's
**Serve all ready** action. The frontend groups a table's items by order so unrelated orders can
process concurrently while each order remains sequenced behind its earlier kitchen commands.

**Request headers for a waiter:**

```http
Authorization: Bearer {waiter-token}
Idempotency-Key: 0190f26e-7c87-7def-8e46-400000000021
```

**Request body:**

```json
{
  "cartItemIds": [82, 83, 84]
}
```

| Field | Type | Rules |
|---|---|---|
| `cartItemIds` | integer array | Required; 1–50 distinct item IDs. Every item must be linked to `{orderId}` and must be ready or already served. |

An active waiter receives `202 Accepted` after the single `order.items_serve_ready` command is
reserved:

```json
{
  "command_id": "0190f26e-7c87-7def-8e46-300000000021",
  "idempotency_key": "0190f26e-7c87-7def-8e46-400000000021",
  "operation": "order.items_serve_ready",
  "status": "accepted",
  "status_url": "/api/vendor/commands/0190f26e-7c87-7def-8e46-300000000021"
}
```

A vendor owner executes synchronously and receives `200` with the updated
[Order Object](#order-object). A completed waiter command stores that same order object in its
terminal `response`.

The worker locks and validates the full selection before updating anything. Already-served items
are idempotent and do not generate duplicate notifications. Errors:

- `403` — the authenticated staff actor is not a waiter.
- `404` — one or more item IDs are not linked to the order; no selected item changes.
- `409` — one or more selected items have not reached `ready`; no selected item changes.
- `422` — missing, duplicate, non-integer, empty, or more than 50 item IDs.
- `503` — Redis-first staff command processing is unavailable.

---

## 9. Mark Order Picked Up

### `PATCH /api/vendor/orders/{orderId}/picked-up`

Marks a paid pickup or takeaway order as collected. This waiter action replaces the dine-in **Serve** action for off-premise orders.

**Request Body:** none

**Execution:** A vendor owner executes synchronously. An active waiter sends a UUID `Idempotency-Key` and receives the shared `202` acknowledgement for operation `order.picked_up`; the order changes only after that command completes. Kitchen staff cannot perform this action.

**Preconditions:**

- The order belongs to an active pickup/takeaway session.
- `paymentReceived` is `true` and the order is no longer `draft`.
- Every linked item has `readyAt` set.
- The order is not `cancelled` or `served`.

On success, the endpoint sets `status → picked_up`, records `pickedUpAt`, and records `pickedUpAt` on every linked item. Repeating the action for an already picked-up order is idempotent and returns its current representation.

**Response `200`:**

```json
{
  "id": "42",
  "orderMode": "pickup",
  "status": "picked_up",
  "displayStatus": "picked-up",
  "pickupStatus": "picked-up",
  "pickedUpAt": "2026-08-10T19:34:00.000000Z",
  "items": [
    {
      "cartItemId": 17,
      "status": "picked_up",
      "pickedUpAt": "2026-08-10T19:34:00.000000Z"
    }
  ]
}
```

**Conflict responses (`409`):**

```json
{ "message": "Only pickup and takeaway orders can be marked picked up." }
```

```json
{ "message": "Payment must be confirmed before pickup." }
```

```json
{ "message": "All order items must be ready before pickup." }
```

---

## 10. Mark Order Served

### `PATCH /api/vendor/orders/{orderId}/served`

Marks a dine-in order as served. Sets `status → served` and records `servedAt`.

**Request Body:** none

**Execution:** A vendor owner receives the synchronous `200` response below. An active waiter must
send a UUID `Idempotency-Key` header and receives `202`; the order is not served until the worker
finishes the command successfully.

**Response `200`:** Returns the updated [Order Object](#order-object) with `status: "served"`.

---

## 11. Cancel Order

### `PATCH /api/vendor/orders/{orderId}/cancel`

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

## 12. Release Batch to Kitchen [Legacy]

> **Deprecated:** This endpoint operates on the legacy `table_sessions` model only. It has no effect on QR-scan sessions (`table_scan_sessions`). It will be removed in a future release.

### `POST /api/vendor/{vendorId}/sessions/{sessionId}/release`

Immediately releases the current batch to the kitchen, overriding the batch window timer. Sets `batchReleasedAt` and marks `batchOpen → false`.

**Request Body:** none

**Response `200`:** Returns the updated legacy session object with `batchOpen: false`.

---

## 13. Fire Next Course [Legacy]

> **Deprecated:** This endpoint operates on the legacy `table_sessions` model only. It has no effect on QR-scan sessions (`table_scan_sessions`). It will be removed in a future release.

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

Closes a legacy `table_sessions` record. New waiter flows should use the table-scan endpoint documented in `qr-management-api.md`:

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
  "orderType": "pickup",
  "orderMode": "pickup",
  "tableNumber": null,
  "tableId": null,
  "tableScanSessionId": "18",
  "course": null,
  "waiterConfirmed": false,
  "waiterConfirmedAt": null,
  "customer": {
    "id": "10",
    "name": "Jane Smith",
    "email": "jane@example.com",
    "phone": "+43 660 1234567"
  },
  "paidBy": {
    "id": "21",
    "name": "Alex Miller"
  },
  "status": "in_progress",
  "displayStatus": "in_progress",
  "pickupStatus": "pending",
  "scheduledFor": "2026-08-10T19:30:00.000000Z",
  "kitchenReleasedAt": "2026-08-10T19:10:00.000000Z",
  "pickupTime": "2026-08-10T19:30:00.000000Z",
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
      "preparingStartAt": "2026-08-10T19:12:00.000000Z",
      "readyAt": null,
      "servedAt": null,
      "pickedUpAt": null
    }
  ],
  "amount": 18.50,
  "serviceFee": 1.50,
  "vatAmount": 2.00,
  "currency": "EUR",
  "paymentMethod": "stripe",
  "paymentProvider": "stripe",
  "paymentMethodLabel": "Apple Pay",
  "paymentMethodDetails": {
    "provider": "stripe",
    "method": "apple_pay",
    "type": "card",
    "displayName": "Apple Pay",
    "cardBrand": "mastercard",
    "cardLast4": "6537",
    "maskedCard": "**** **** **** 6537",
    "walletType": "apple_pay"
  },
  "paymentPending": false,
  "paymentReceived": true,
  "paymentConfirmedAt": "2026-08-10T19:05:00.000000Z",
  "paymentNote": null,
  "readyAt": null,
  "servedAt": null,
  "pickedUpAt": null,
  "cancelledAt": null,
  "cancelledReason": null,
  "timeline": [
    { "status": "confirmed", "timestamp": "2026-08-10T19:05:00.000000Z" },
    { "status": "in_progress", "timestamp": "2026-08-10T19:12:00.000000Z" }
  ],
  "createdAt": "2026-08-10T19:00:00.000000Z",
  "updatedAt": "2026-08-10T19:12:00.000000Z"
}
```

**Notes:**
- `customer` identifies the order owner. `paidBy` is `null` unless another
  customer has explicitly taken responsibility through the pay-for flow. When
  present, waiter clients should attribute the order and its items to that
  payer; both identities expose stable string IDs.
- `itemsCount` is computed live as the sum of `quantity` across linked cart_items.
- `items[]` is built live from `cart_items` (owned by the order's session, plus any cart_item whose `shared_order_ids` JSON contains the order id).
- Item `status` is derived from `picked_up_at`, `served_at`, `ready_at`, and `preparing_start_at`: `picked_up`, `served`, `ready`, `in_progress`, or `new`.
- `readyAt` reflects the order-level rollup (latest timestamp once *all* linked cart_items have `ready_at` set). The per-item `readyAt` is the canonical value.
- `orderMode` is sourced from the active customer session when available and is `dine-in`, `pickup`, or `takeaway`.
- `scheduledFor` is the requested pickup time. `kitchenReleasedAt` is set when the order enters the kitchen preparation queue. `pickupTime` is `scheduledFor` for scheduled pickup, otherwise the order's estimated ASAP collection time.
- `pickupStatus` is `pending`, `ready`, or `picked-up`. `pickedUpAt` is returned at both order and item level after collection.
- `paymentMethod` remains the order-level payment category used by operational logic. Use `paymentMethodDetails` or `paymentMethodLabel` for the exact Stripe wallet, card brand, and safe masked card number.

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

All vendor order actions that modify order or cart item state create notifications for every
customer with an active session at the same physical table. The customer app receives the same
change live through its private Pusher channel; polling the notification feed is recovery, not the
normal update path.

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

### Customer Pusher behavior when food is served

Serving an entire order through `PATCH /api/vendor/orders/{orderId}/served` emits
`.notification.created` on every affected `private-customer.{customer_id}` channel after the
mutation succeeds. For a waiter TeamMember this happens when the queued command completes, not when
the API returns `202`. A failed staff command emits no customer served-state event.

```json
{
  "id": "019b09a1-8cd1-72f2-a123-123456789abc",
  "user_id": null,
  "user_role": "customer",
  "event": "order_updated",
  "message": "Your order has been served. Enjoy!",
  "read": false,
  "metadata": {
    "event_id": "019b09a1-8cd1-72f2-a123-123456789abc",
    "event_version": 1784289600123456,
    "template": "order.served",
    "order_id": 41,
    "order_snapshots": [
      {
        "order_id": 41,
        "order_public_id": "ord-a1b2c3d4e5f6",
        "status": "served",
        "items": [
          {
            "cart_item_id": 82,
            "status": "served",
            "served_at": "2026-07-17T12:00:01.000000Z"
          }
        ]
      }
    ]
  },
  "created_at": "2026-07-17T12:00:01.000000Z",
  "updated_at": "2026-07-17T12:00:01.000000Z"
}
```

Serving one item through the item-status endpoint emits `event: "cart_item_updated"` with template
`cart.item_served`, `cart_item_id`, and item-inclusive `order_snapshots` for every order affected by
that item, including shared orders. The customer should apply those snapshots and deduplicate by
`metadata.event_id`. The vendor/waiter/kitchen terminal command event and the customer served event
are separate messages on separate private channels.

---

## 16. Realtime Notifications

Vendor owners, kitchen staff, and waiters use the same authenticated notification API. Results are scoped to the authenticated actor; customer-targeted notifications are never returned to vendor actors.

### 16.1 Authorize Pusher Channel

**POST** `/api/vendor/broadcasting/auth`
**Authentication:** Required (`vendor` or active `team_member`)  
**Content-Type:** `application/json`

Laravel Echo calls this endpoint when the authenticated actor joins its private Pusher channel.

| Actor | Echo channel | Pusher wire channel |
|---|---|---|
| Vendor owner | `vendor.{vendor_id}` | `private-vendor.{vendor_id}` |
| Waiter | `waiter.{team_member_id}` | `private-waiter.{team_member_id}` |
| Kitchen | `kitchen.{team_member_id}` | `private-kitchen.{team_member_id}` |

An actor may authorize only the channel matching its authenticated model, role, and ID.

**Request body:**

```json
{
  "socket_id": "1234.5678",
  "channel_name": "private-waiter.19"
}
```

**Response (200):**

```json
{
  "auth": "pusher-app-key:channel-signature"
}
```

**Errors:**

- `401` — missing or invalid vendor/staff bearer token.
- `403` — inactive staff or an attempt to authorize another actor/role channel.
- `422` — missing or invalid `socket_id` / `channel_name`.
- `503` — Pusher is not selected or its key/secret is not configured.

### 16.2 List Notifications

**GET** `/api/vendor/notifications`  
**Authentication:** Required (`vendor`, `kitchen`, or `waiter`)  
**Headers:** Optional `Accept-Language`  
**Request body:** None

```json
{
  "notifications": [
    {
      "id": 91,
      "event": "order_confirmed",
      "message": "New order #41 for Table 7.",
      "metadata": {
        "resources": ["orders", "tables", "dashboard", "notifications"],
        "order_id": 41,
        "table_id": 7,
        "severity": "urgent",
        "sound": "new_order"
      },
      "read": false,
      "created_at": "20.06.2026 12:00"
    }
  ],
  "unread_count": 1
}
```

Silent realtime invalidations are not included in this response or the unread count.

### 16.3 Mark Notification Read

**PATCH** `/api/vendor/notifications/{id}/read`  
**Authentication:** Required; the notification must belong to the authenticated actor  
**Request body:** None

- Vendor owner: synchronous `200`; no idempotency header is required.
- Waiter/kitchen TeamMember: UUID `Idempotency-Key` header is mandatory and the endpoint returns the
  shared `202` command acknowledgement. The row remains unread until the command completes.

```json
{ "message": "Notification marked as read." }
```

The synchronous owner response above is also preserved in a completed TeamMember command's
`response` field. For TeamMembers, a notification outside the actor's scoped feed is accepted first
and then finishes with `http_status: 404`; this keeps the mutation request Redis-first.

### 16.4 Mark All Notifications Read

**POST** `/api/vendor/notifications/read-all`  
**Authentication:** Required  
**Request body:** None

- Vendor owner: synchronous `200`; no idempotency header is required.
- Waiter/kitchen TeamMember: UUID `Idempotency-Key` header is mandatory and the endpoint returns
  `202`. All visible unread notifications in that actor's feed are marked read by the worker in
  sequence with the actor's other notification-read commands.

```json
{ "message": "All notifications marked as read." }
```

### 16.5 Realtime Subscription

Subscribe with Laravel Echo to the actor channel from §16.1 and listen for the custom event
`.notification.created`. The bearer token used by the vendor API must also be sent to the
authorization endpoint.

```ts
echo
  .private(`waiter.${teamMemberId}`)
  .listen('.notification.created', (notification) => {
    // Refresh or patch only the keys named by notification.metadata.resources.
  });
```

The backend first persists each actor-scoped notification row, then broadcasts that persisted row
from the dedicated vendor realtime queue. A typical payload is:

```json
{
  "id": 91,
  "user_id": 19,
  "event": "order_confirmed",
  "message": "A new order was confirmed.",
  "read": false,
  "is_silent": false,
  "user_role": "waiter",
  "vendor_id": 7,
  "metadata": {
    "event_id": "019b09a1-8cd1-72f2-a123-123456789abc",
    "event_version": 1784289600123456,
    "resources": ["orders", "tables", "dashboard", "notifications"],
    "order_snapshots": [
      {
        "order_id": 41,
        "status": "confirmed"
      }
    ]
  },
  "created_at": "2026-07-17T12:00:00.000000Z",
  "updated_at": "2026-07-17T12:00:00.000000Z"
}
```

Use `metadata.event_id` to deduplicate job retries. Clients should apply included snapshots when
supported and otherwise refresh the Laravel resources listed in `metadata.resources`. Silent
events have `is_silent: true` and `read: true`; they invalidate application data but do not enter
the visible notification feed or unread count.

### 16.6 Pickup/Takeaway Realtime Sequence

All operational deliveries use the same `.notification.created` subscription and include a full current order at `metadata.order` so kitchen/waiter clients can insert or update the card without a follow-up request.

| Moment | Audience | Event | Delivery behavior |
|---|---|---|---|
| Off-premise payment confirmed | Vendor + waiter | `order_confirmed` | Immediate normal notification; waiter can see the paid collection order |
| Future scheduled pickup paid | Kitchen | `order_scheduled` | Silent (`is_silent: true`, already read); inserts/updates the separate Pickups tab with `kitchenReleasedAt: null`, without toast or sound |
| ASAP payment or scheduled pickup reaches T−20 | Kitchen | `order_confirmed` | Normal urgent notification with `sound: "new_order"`; `metadata.order.kitchenReleasedAt` is set and the card enters the normal queue |
| Order collected | Customer PIN group | `order_updated` | Carries `template: "order.picked_up"` and authoritative `order_snapshots` |
| Order collected | Vendor + waiter + kitchen | `order_picked_up` | Silent operational update; removes/reconciles the active collection card without adding notification noise |

Payment webhook delivery, cash confirmation, and the scheduled-release command all use the same idempotent release service. Kitchen clients may therefore reconcile repeated snapshots safely by event ID and the order's current `kitchenReleasedAt`.

The backend requires the following deployment settings. Operational persistence and Pusher delivery
are isolated from customer notification timing on dedicated Redis queues.

```dotenv
BROADCAST_CONNECTION=pusher
VENDOR_REALTIME_ENABLED=true
VENDOR_QUEUE_CONNECTION=redis
VENDOR_NOTIFICATIONS_QUEUE=vendornotifications
VENDOR_REALTIME_QUEUE=vendorrealtime
STAFF_ASYNC_COMMANDS_ENABLED=true
STAFF_COMMANDS_CONNECTION=redis
STAFF_COMMANDS_QUEUE=staffcommands
STAFF_COMMAND_STATUS_TTL=3600
STAFF_COMMAND_LOCK_SECONDS=120

PUSHER_APP_ID=...
PUSHER_APP_KEY=...
PUSHER_APP_SECRET=...
PUSHER_APP_CLUSTER=...
```

---

## 17. Error Reference

| HTTP Code | Meaning |
|---|---|
| `401 Unauthorized` | Missing or invalid bearer token |
| `403 Forbidden` | Token belongs to a different vendor or actor/role cannot use the route |
| `404 Not Found` | Order or session does not exist or belongs to another vendor |
| `409 Conflict` | Domain state conflict or TeamMember idempotency key reused for a different command |
| `422 Unprocessable Entity` | Validation failed (e.g. firing next course when already on desserts) |
| `503 Service Unavailable` | Pusher is not configured or Redis-first staff command processing is unavailable |
| `500 Internal Server Error` | Unexpected server error |

---

## Route Summary

| Method | Path | Action |
|---|---|---|
| `GET` | `/api/vendor/{vendorId}/orders` | List all orders (grouped) |
| `GET` | `/api/vendor/{vendorId}/orders/{orderId}` | Get single order |
| `PATCH` | `/api/vendor/orders/{orderId}` | Generic update (status / payment) |
| `PATCH` | `/api/vendor/orders/{orderId}/confirm` | Confirm order; waiter TeamMember uses an async command |
| `PATCH` | `/api/vendor/orders/{orderId}/confirm-cash` | Confirm cash; waiter TeamMember uses an async command |
| `PATCH` | `/api/vendor/orders/{orderId}/ready` | Mark ready; kitchen TeamMember uses an async command |
| `PATCH` | `/api/vendor/orders/{orderId}/items/{cartItemId}` | Update cart item status for KDS/waiter |
| `PATCH` | `/api/vendor/orders/{orderId}/items/serve-ready` | Serve 1–50 ready items with one order command |
| `POST` | `/api/vendor/orders/items/status-batch` | Dispatch 1–50 item-status staff commands |
| `PATCH` | `/api/vendor/orders/{orderId}/picked-up` | Mark a ready, paid pickup/takeaway collected; waiter TeamMember uses an async command |
| `PATCH` | `/api/vendor/orders/{orderId}/served` | Mark served; waiter TeamMember uses an async command |
| `PATCH` | `/api/vendor/orders/{orderId}/cancel` | Cancel order |
| `POST` | `/api/vendor/{vendorId}/sessions/{sessionId}/release` | [Legacy] Release batch to kitchen now |
| `POST` | `/api/vendor/{vendorId}/sessions/{sessionId}/fire-course` | [Legacy] Advance to next course |
| `POST` | `/api/vendor/{vendorId}/sessions/{sessionId}/close` | [Legacy] Close legacy table session |
| `POST` | `/api/vendor/{vendorId}/tables/{tableId}/session` | Start a table session (staff) |
| `POST` | `/api/vendor/{vendorId}/tables/{tableId}/staff-order` | Place a staff order for a table |
| `POST` | `/api/vendor/broadcasting/auth` | Authorize the actor's private Pusher channel |
| `GET` | `/api/vendor/commands/{commandId}` | Get initiating TeamMember's staff-command status |
| `GET` | `/api/vendor/notifications` | List actor-scoped notifications |
| `PATCH` | `/api/vendor/notifications/{id}/read` | Mark one notification read |
| `POST` | `/api/vendor/notifications/read-all` | Mark all actor notifications read |

---

## 18. Start Table Session (Staff)

Creates an active `table_scan_session` for a table without a customer QR scan — exactly like a customer-created session, including the 4-digit PIN. The waiter can hand the PIN to guests, who join through the customer app (`POST /api/customer/table/pin`) and order normally.

- **Method:** `POST`
- **URL:** `/api/vendor/{vendorId}/tables/{tableId}/session`
- **Auth:** Bearer token — vendor owner or **waiter** team member
- **Body:** _none_

**Response mode:** The vendor owner receives the synchronous `201`/`200` response documented
below. A waiter TeamMember must send a UUID `Idempotency-Key` header and receives `202`; the eventual
`201`/`200` body is available in the terminal command's `response` field.

Idempotent: if the table already has an active session, the existing session is returned with `created: false` instead of creating a duplicate.

**Response `201 Created`** (new session) / **`200 OK`** (already active):

```json
{
  "message": "Session started.",
  "session_id": "42",
  "pin": "4837",
  "created": true
}
```

**Errors:**

| HTTP Code | Condition |
|---|---|
| `403` | Token belongs to a different vendor, or staff role is not `waiter` |
| `404` | Table not found for this vendor |
| `422` | Table is inactive (`code: "inactive_table"`) |

---

## 19. Place Staff Order

Places a dine-in order on behalf of a table (waiter dashboard **+ ORDER**). Creates `cart_items` and an `orders` row identical to the customer flow: gross (VAT-inclusive) pricing via the tax service, plus the vendor's service fee — `service_fee = round(items_total × service_fee_rate / 100, 2)` and `amount = items_total + service_fee`. If the table has no active session, one is created automatically (with a PIN). The order is marked `placed_by: "waiter"`; when placed by a team member, `placed_by_team_member_id` records who. A newly placed order starts with `payment_pending: false`; that flag becomes true only after a Stripe intent or cash-payment request is created.

**Merging:** as long as the session has an unpaid, non-cancelled waiter order, subsequent staff orders append their items to it and reprice the whole order (items + service fee) — mirroring the customer flow's unpaid-order merge. The response returns the merged order's id. Once that order's payment is collected (`payment_received: true`), the next staff order starts a fresh order.

- **Method:** `POST`
- **URL:** `/api/vendor/{vendorId}/tables/{tableId}/staff-order`
- **Auth:** Bearer token — vendor owner or **waiter** team member

**Response mode:** The vendor owner receives the synchronous `201` response documented below. A
waiter TeamMember must send a UUID `Idempotency-Key` header and receives `202`; menu/modifier or
other domain errors found by the worker appear in the terminal command response.

**Request body:**

```json
{
  "items": [
    {
      "menu_item_id": 12,
      "quantity": 2,
      "notes": "No onions",
      "paid_addons": [{ "id": 1, "name": "Extra cheese", "price": 1.5 }],
      "free_addons": ["Ketchup"],
      "removed_items": ["Pickles"],
      "selected_modifiers": [
        { "modifier_group_id": 3, "option_ids": [7] }
      ]
    }
  ]
}
```

Modifier selections are validated with the same rules as the customer cart: required groups must have a selection, `min_selection`/`max_selection` are enforced, and unknown groups or options are rejected with `422`.

**Response `201 Created`:**

```json
{
  "message": "Order placed successfully.",
  "order_id": "ord-a1b2c3d4e5f6",
  "amount": 26.35,
  "session_id": 42,
  "order": {
    "id": "91",
    "orderPublicId": "ord-a1b2c3d4e5f6",
    "tableId": "8",
    "tableNumber": "8",
    "status": "confirmed",
    "displayStatus": "received",
    "items": [
      {
        "cartItemId": 301,
        "name": "Burger",
        "quantity": 2,
        "price": 11.5,
        "status": "new"
      }
    ],
    "total": 26.35,
    "paymentPending": false,
    "paymentReceived": false
  },
  "participant": {
    "session_id": 42,
    "customer_id": null,
    "name": "Waiter",
    "scanned_at": "2026-07-18T12:00:00.000000Z",
    "status": "active"
  },
  "state_patch": {
    "id": "01981f60-26f4-7000-8000-000000000001",
    "version": 1784376000000000,
    "operation": "order.staff_created",
    "orders": {
      "upsert": [
        {
          "id": 91,
          "order_public_id": "ord-a1b2c3d4e5f6",
          "customer_id": null,
          "table_scan_session_id": 42,
          "status": "confirmed",
          "amount": 26.35,
          "visible_item_ids": [301]
        }
      ],
      "remove_ids": []
    },
    "items": {
      "upsert": [
        {
          "cart_item_id": 301,
          "owner_order_id": 91,
          "owner_table_scan_session_id": 42,
          "name": "Burger",
          "quantity": 2,
          "status": "new"
        }
      ],
      "remove_ids": []
    }
  }
}
```

The `order` snapshot is the same realtime-safe shape sent in operational Pusher metadata. The
initiating waiter applies this response directly, while other waiter/kitchen devices apply the
Pusher snapshot; neither path needs a follow-up orders GET. Every active customer at the table also
receives an `order_updated` Pusher event containing the same authoritative `state_patch` and the
customer-less `participant` named `Waiter`. The customer adds that participant before applying the
patch, so a new waiter order is inserted immediately and later waiter additions update it without
polling or refetching. `state_patch.operation` is `order.staff_created` for a new unpaid waiter order
and `order.staff_updated` when items are merged into the existing order. The order also appears in
`GET /api/vendor/{vendorId}/orders` with `"placedBy": "waiter"` (customer-placed orders have
`"placedBy": "customer"`).

**Errors:**

| HTTP Code | Condition |
|---|---|
| `403` | Token belongs to a different vendor, or staff role is not `waiter` |
| `404` | Table not found for this vendor |
| `422` | Menu item unavailable/inactive, or modifier selection violates group rules |

---

## 20. Order History (Paginated)

### `GET /api/vendor/{vendorId}/orders/history`

Returns a paginated list of all non-draft orders for the vendor, sorted newest first. Useful for reports and historical lookups.

**Auth:** `Bearer {vendorToken}` or team member token

**Query Parameters:**

| Param | Type | Default | Description |
|---|---|---|---|
| `page` | integer | 1 | Page number |
| `perPage` | integer | 20 | Items per page (max 50) |
| `status` | string | — | Filter by order status (`confirmed`, `waiter_confirmed`, `in_progress`, `served`, `picked_up`, `cancelled`) |
| `orderType` | string | — | Filter by order type (`dine-in`, `pickup`, `takeaway`) |
| `payment` | string | — | Filter by payment state (`paid`, `unpaid`, `pending-cash`) |
| `search` | string | — | Search by order number or public ID |
| `dateFrom` | string (`dd.mm.yyyy`) | — | Include orders created on or after this calendar date in the vendor's timezone |
| `dateTo` | string (`dd.mm.yyyy`) | — | Include orders through the end of this calendar date in the vendor's timezone |

**Response `200`:**

```json
{
  "data": [
    { /* standard order object — same shape as single-order responses */ }
  ],
  "meta": {
    "currentPage": 1,
    "lastPage": 5,
    "perPage": 20,
    "total": 93
  }
}
```

If both dates are supplied, `dateTo` must be the same as or later than `dateFrom`; malformed or reversed ranges return `422` validation errors.

---

## 21. Order Receipt

### `GET /api/vendor/{vendorId}/orders/{orderId}/receipt`

Returns the structured receipt for a paid order. `{orderId}` accepts the numeric order ID or public ID. A payment that covered several orders returns all covered orders in the receipt.

**Auth:** `Bearer {vendorToken}` or an authorized team member token

**Request body:** none

The payment block reports the exact Stripe method and only safe card data:

```json
{
  "data": {
    "payment": {
      "provider": "stripe",
      "method": "google_pay",
      "method_details": {
        "provider": "stripe",
        "method": "google_pay",
        "type": "card",
        "display_name": "Google Pay",
        "card_brand": "visa",
        "card_last4": "6537",
        "masked_card": "**** **** **** 6537",
        "wallet_type": "google_pay"
      },
      "status": "CONFIRMED",
      "transaction_id": "pi_3Nxxx",
      "paid_at": "13.07.2026 14:05"
    }
  }
}
```

For cash, `provider`, `method`, and `method_details.method` are `cash`, while card and wallet fields are null. Other Stripe methods use Stripe's method type and a human-readable `display_name`.

**Errors:** `404` for an unknown/vendor-mismatched order; `422` when the order is not paid.
