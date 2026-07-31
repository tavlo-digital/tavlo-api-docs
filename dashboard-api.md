# Vendor Dashboard API

## GET `/api/vendor/{vendorId}/dashboard`

Returns the vendor's live operational dashboard. Authentication is required with a vendor-owner bearer token. `{vendorId}` accepts the vendor public ID or numeric ID; a vendor cannot read another vendor's dashboard.

```http
GET /api/vendor/VID-1234/dashboard
Authorization: Bearer {token}
Accept: application/json
```

### Response `200`

```json
{
  "liveStatus": {
    "occupiedTables": 4,
    "totalTables": 12,
    "activeOrders": 7,
    "tablesWaitingToPay": 2,
    "ordersWaitingToPay": 3,
    "kitchenLoad": "medium"
  },
  "todayKPIs": {
    "ordersToday": 24,
    "ordersYesterday": 20,
    "paidOrdersToday": 18,
    "paidOrdersYesterday": 16,
    "revenueToday": 720.5,
    "revenueYesterday": 640,
    "tipsToday": 48.5,
    "tipsYesterday": 40,
    "avgOrderValue": 40.03,
    "avgOrderValueYesterday": 40,
    "customerRating": 4.7,
    "customerRatingYesterday": 4.5,
    "currency": "EUR"
  },
  "alerts": [
    {
      "id": "unpaid-old",
      "severity": "warning",
      "message": "2 orders have been unpaid for over 10 minutes",
      "navigateTo": "orders"
    }
  ],
  "recentOrders": [
    {
      "id": "720",
      "orderNumber": "#720",
      "orderType": "dine-in",
      "tableNumber": 8,
      "status": "in_progress",
      "amount": 42.5,
      "currency": "EUR",
      "paymentPending": false,
      "paymentReceived": false,
      "createdAt": "2026-07-19T18:02:00.000000Z",
      "customer": {
        "name": "Alex Smith",
        "phone": "+431234567"
      }
    }
  ],
  "topItems": [
    {
      "id": "46",
      "name": "Margherita Pizza",
      "price": 12.5,
      "orderedCount": 14,
      "quantitySold": 19
    }
  ],
  "revenueAtRisk": {
    "total": 85,
    "currency": "EUR"
  },
  "timezone": "Europe/Vienna",
  "generatedAt": "2026-07-19T18:05:00.000000Z"
}
```

### Metric definitions

- Daily ranges use the vendor country's timezone and are converted to UTC for database queries.
- `ordersToday` and `ordersYesterday` count submitted orders and exclude drafts and cancelled orders. The confirmation timestamp is used when present, with `created_at` as the legacy fallback.
- Revenue and tips use `payment_confirmed_at`, not order creation time. Paid legacy rows without that timestamp fall back to `created_at`.
- Average order value is paid revenue divided by the number of paid orders in that day.
- `occupiedTables` counts distinct physical tables with active sessions containing active orders.
- `tablesWaitingToPay` counts distinct physical tables with outstanding payable orders. `ordersWaitingToPay` includes both dine-in and takeaway orders.
- `revenueAtRisk.total` is the current outstanding value of all unpaid in-progress, served, or picked-up orders. It is not restricted to orders created today.
- Top items are calculated from submitted `cart_items`, exclude draft/cancelled orders, and aggregate menu-item versions through `product_uid`. `orderedCount` is the number of distinct orders; `quantitySold` is the item quantity.
- Inventory alerts respect the vendor's `alerts.dashboardAlerts` inventory setting.

### Errors

- `401` when no valid token is supplied.
- `403` when the authenticated vendor requests another vendor's dashboard.
- `404` when `{vendorId}` cannot be resolved.
