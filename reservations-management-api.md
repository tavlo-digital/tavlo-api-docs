# Vendor Reservations Management API

Base URL: `https://your-domain.com/api`

Endpoints require `Authorization: Bearer {token}`. Vendor owners have access to
their own reservations; staff roles do not have access to reservation management.

---

## 1. List Reservations

**`GET /vendor/{vendorId}/reservations`**

Returns the vendor's reservations ordered by date and time.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `vendorId` | string | Vendor public ID or numeric ID. Must match the authenticated vendor. |

### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Optional: `pending`, `confirmed`, `cancelled`, `completed`, or `no_show`. |
| `date` | string | Optional exact reservation date in `YYYY-MM-DD` format. |

### Response `200 OK`

```json
[
  {
    "id": "25",
    "reservationPublicId": "res_a1b2c3d4",
    "customer": {
      "id": "8",
      "name": "Ada Lovelace",
      "email": "ada@example.com",
      "phone": "+43 1 2345678"
    },
    "guestName": "Ada Lovelace",
    "guestEmail": "ada@example.com",
    "guestPhone": "+43 1 2345678",
    "date": "2026-08-20",
    "time": "18:30:00",
    "partySize": 4,
    "status": "pending",
    "customerNote": "Window seat, please.",
    "vendorNote": null,
    "tableNumber": null,
    "createdAt": "2026-08-14T15:00:00.000000Z"
  }
]
```

`customer` is `null` for a reservation without a linked customer account. Use
the `guestName`, `guestEmail`, and `guestPhone` snapshot fields in that case.

### Error Responses

- `403` — the authenticated vendor does not own `{vendorId}`.
- `422` — an invalid `status` or non-`YYYY-MM-DD` `date` was supplied.

---

## 2. Update Reservation Status

**`PATCH /vendor/reservations/{reservationId}/status`**

`reservationId` may be the public reservation ID or numeric database ID. A
numeric database lookup is attempted only when the supplied value contains
digits only, so public IDs are never cast to PostgreSQL's bigint `id` type. The
reservation must belong to the authenticated vendor.

### Request Body

```json
{
  "status": "confirmed",
  "vendorNote": "Table 4"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | string | Yes | `pending`, `confirmed`, `cancelled`, `completed`, or `no_show`. |
| `vendorNote` | string or null | No | Internal vendor note, up to 1000 characters. |

Declining a pending request is stored as `cancelled`; `declined` is not a
canonical reservation status.

### Response `200 OK`

Returns the updated reservation in the same shape as a row from the list
endpoint.

### Error Responses

- `403` — the reservation belongs to another vendor.
- `404` — no reservation matches `{reservationId}`.
- `422` — validation failed.
