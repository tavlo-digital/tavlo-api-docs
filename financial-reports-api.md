# Financial Reports API

**Base URL prefix:** `/api/vendor/{vendorId}`
**Authentication:** `Authorization: Bearer {token}` — vendor-owner token only. Team-member tokens receive `403` (see [Access Control](#access-control)).
**Content-Type:** `application/json`

`vendorId` accepts either the `vendor_public_id` (e.g. `V-ABC12345`) or the numeric primary key.

> See `analytics-api.md` for the Analytics API — this file follows the same house style.

---

## Table of Contents

1. [Overview & Concepts](#1-overview--concepts)
2. [Get Financial Report](#2-get-financial-report)
3. [Manual Expenses ("Other Costs")](#3-manual-expenses-other-costs)
4. [Access Control](#access-control)
5. [Database Schema Reference](#database-schema-reference)

---

## 1. Overview & Concepts

This is a bookkeeping report for the vendor's own records — **not** a certified tax-authority export. It exists to give a vendor (and their accountant) revenue, tax, refund, and cost figures for a period, and to let them download it as an Excel workbook.

### Cash basis, not order-placed basis
Every figure is windowed by the moment money actually moved:
- Orders are windowed by `payment_confirmed_at` (when the order was actually paid), not `created_at`. This is **intentionally different from `GET /vendor/{vendorId}/analytics`**, which windows by `created_at`. An order placed on day N but paid on day N+2 will appear in a different period here than in Analytics — that is correct for a bookkeeping report and not a bug.
- Refunds are windowed by `refunds.resolved_at` (when the refund was actually approved), not the original order's date. A refund against a prior period's sale is a cash event that happened in the period it was resolved in, and must reconcile against that period's bank/Stripe statement.

### "Net Revenue" is gross minus VAT only — and VAT is always derived fresh
`summary.netRevenue` = gross revenue − VAT, the same formula `GET /vendor/{vendorId}/analytics` uses. The VAT figure itself, however, is **always recomputed from `cart_items` via `TaxCalculationService`**, never read from `orders.vat_amount` — that column was found to be stale/zero on real orders whose cart items clearly carry non-zero VAT, which would otherwise make `totalTax` contradict the Tax Breakdown card on the same page. This means Analytics and this endpoint can legitimately disagree on Net Revenue for a vendor whose `orders.vat_amount` is stale; that's expected, not a bug to reconcile. Net Revenue is **not** reduced by refunds, tips, service fees, or costs — those are their own sections. Do not re-derive a "money actually kept" figure by subtracting `refunds`/`costs` from `netRevenue` without labeling it clearly as an estimate; this endpoint does not attempt that calculation itself because Stripe payout timing/fees are not modeled anywhere in this codebase.

**`menu_items.price` is NET (VAT-exclusive), consistently across the whole app** — `TaxCalculationService::gross()` adds VAT on top everywhere a customer-facing charge is computed (`CartController`, `PaymentController`) and everywhere this endpoint derives VAT. This was double-checked against a claim that menu prices might already be VAT-inclusive; they are not, for either real customer charges or this endpoint's figures. Every `computeTaxGroups()` call in this service passes `applySharing: true` — omitting it (a bug this endpoint shipped with briefly) counts a cart line shared across a split order at its full gross instead of dividing by the sharer count, which is how `CartController`/`PaymentController` actually computed `orders.amount`; without the flag, `taxBreakdown.totals.grossAmount` and `summary.totalTax` silently overstate any vendor using order-sharing.

### No English strings in the payload
Every label-worthy field (`taxCategory`, `orderType`, `reason`) is returned as a stable machine slug, not a translated or English display string. The dashboard renders these in the vendor's `dashboardLanguage` client-side. Do not add a human-readable `label` field to this endpoint — if a new value needs a display string, it belongs in the frontend's translation dictionary.

### Periods
| Key | Meaning |
|---|---|
| `today` | Vendor's current calendar day |
| `week` | Vendor's current calendar week (Monday–Sunday) |
| `month` | Vendor's current calendar month |
| `last_month` | The calendar month before this one |
| `year` | Vendor's current calendar year |
| `custom` | Vendor-chosen `from`/`to`, inclusive, max 731 days |

All boundaries are computed in the vendor's own timezone (`Vendor::resolveTimezone()`), same as Analytics.

A `custom` range longer than 731 days is not rejected — it's silently shortened to 731 days from `from`, same behavior as Analytics (366-day cap there). Surfaced the same way: `range.truncated` is `true` and `range.maxCustomDays` is `731` whenever this happened (always `false`/`731` otherwise); `range.from`/`range.to` always reflect the actual (possibly shortened) window used. 2026-09-07 audit finding, fixed the same day.

### Plan gating

`GET /financial-reports`, and all four Manual Expenses routes below (`/financial-expenses*`), check the vendor's *current* subscription plan for the same **"Basic Analytics"** feature Analytics requires — `GatesAnalyticsFeature::hasAnalyticsAccess()` (`app/Http/Controllers/Api/Vendor/Concerns/GatesAnalyticsFeature.php`, shared with `AnalyticsController`), never a hardcoded plan tier. **This did not exist before 2026-09-07** — every vendor-owner token previously got the real payload and full expense CRUD regardless of plan, an audit finding; the founder's decision was to reuse Analytics' existing feature rather than introduce a second, separately-priced one.

`GET /financial-reports` without the feature gets a **locked response** instead of the real payload — same `200` status, cheap to compute (checked before period resolution, so a locked vendor never triggers the background-report threshold logic below either):

```json
{ "locked": true, "requiredFeature": "Basic Analytics", "currency": "EUR" }
```

Unlike Analytics' `GET /analytics`, there is no `preview` field here — this endpoint has no equivalent cheap "today" snapshot query to build one from. Frontend clients should check `locked` before reading any other field; `tavlo-vendor`'s `FinancialReportResult` type models this as a discriminated union (`FinancialReportPayload | FinancialReportProcessing | FinancialReportLockedPayload`) in `app/vendor/financial-reports/types.ts`.

The four Manual Expenses routes (writes, not reads) return a plain `403` instead of a locked JSON shape when the feature is missing — there is no payload to gracefully degrade, and a vendor without access has no legitimate way to reach that UI in the first place (the whole page shows a locked preview instead).

### Rate limiting

`GET /financial-reports` shares one named limiter with all five Analytics routes: **30 requests/minute per authenticated vendor** (not per IP). Added because this route has no caching for most of its sub-aggregations and rebuilding the full payload is one of the most expensive reads in the vendor API — see `analytics-api.md`'s own Rate limiting section for the full rationale. Exceeding it returns `429`. The Manual Expenses routes below (`/financial-expenses*`) have their own, separate limiter — **20 requests/minute per authenticated vendor** — added 2026-09-07 (previously unthrottled entirely).

---

## 2. Get Financial Report

### `GET /api/vendor/{vendorId}/financial-reports`

Returns the full report payload for one period: summary KPIs (each with a vs-previous-period delta, same shape as Analytics' `metric()`), VAT breakdown by tax category, revenue by order type/category/payment method, refunds, voided orders, costs (including vendor-entered manual expenses), inventory waste/losses, an estimated tip distribution across staff, a reminder of upcoming recurring costs, and a daily/weekly/monthly breakdown table. There is no separate export endpoint — the frontend builds both a full `.xlsx` workbook and a summarized accounting CSV (a QuickBooks/Xero-style journal entry, for the vendor's bookkeeper) client-side from this same response.

**Query Parameters**

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `period` | `string` | `week` | One of `today`, `week`, `month`, `last_month`, `year`, `custom` |
| `from` | `string` | — | `Y-m-d`. Required when `period=custom` |
| `to` | `string` | — | `Y-m-d`. Required when `period=custom`, must be `>= from` |

**Response `200`:**

```json
{
  "period": "week",
  "currency": "EUR",
  "range": { "from": "2026-08-24", "to": "2026-08-30", "truncated": false, "maxCustomDays": 731 },
  "previousRange": { "from": "2026-08-17", "to": "2026-08-23" },
  "bucketUnit": "day",
  "hasData": true,
  "summary": {
    "grossRevenue": { "value": 8282.60, "previous": 7900.10, "delta": 4.8, "deltaUnit": "percent" },
    "netRevenue": { "value": 7539.64, "previous": 7190.10, "delta": 4.9, "deltaUnit": "percent" },
    "totalTax": { "value": 742.96, "previous": 710.00, "delta": 4.6, "deltaUnit": "percent" },
    "totalOrders": { "value": 167, "previous": 150, "delta": 17, "deltaUnit": "absolute" },
    "avgOrderValue": { "value": 49.60, "previous": 52.67, "delta": -5.8, "deltaUnit": "percent" },
    "totalTips": { "value": 210.00, "previous": 180.00, "delta": 16.7, "deltaUnit": "percent" },
    "tipRatePercent": { "value": 2.5, "previous": 2.3, "delta": 0.2, "deltaUnit": "absolute" },
    "totalServiceFees": { "value": 0.00, "previous": 0.00, "delta": null, "deltaUnit": "percent" },
    "totalRefunds": { "value": 142.54, "previous": 60.00, "delta": 137.6, "deltaUnit": "percent" },
    "costOfGoodsSold": { "value": 2100.00, "previous": 1950.00, "delta": 7.7, "deltaUnit": "percent" },
    "costOfGoodsSoldPercent": { "value": 27.9, "previous": 27.1, "delta": 0.8, "deltaUnit": "absolute" },
    "grossProfit": { "value": 5439.64, "previous": 5240.10, "delta": 3.8, "deltaUnit": "percent" },
    "grossMarginPercent": { "value": 72.1, "previous": 72.9, "delta": -0.8, "deltaUnit": "absolute" },
    "laborCost": { "value": 1800.00, "previous": 1750.00, "delta": 2.9, "deltaUnit": "percent" },
    "laborCostPercent": { "value": 23.9, "previous": 24.3, "delta": -0.4, "deltaUnit": "absolute" },
    "primeCost": { "value": 3900.00, "previous": 3700.00, "delta": 5.4, "deltaUnit": "percent" },
    "primeCostPercent": { "value": 51.7, "previous": 51.5, "delta": 0.2, "deltaUnit": "absolute" },
    "operatingExpenses": { "value": 1569.00, "previous": 1500.00, "delta": 4.6, "deltaUnit": "percent" },
    "operatingProfit": { "value": 2070.64, "previous": 1990.10, "delta": 4.0, "deltaUnit": "percent" },
    "netProfit": { "value": 1922.10, "previous": 1924.00, "delta": -0.1, "deltaUnit": "percent" },
    "netMarginPercent": { "value": 25.5, "previous": 26.8, "delta": -1.3, "deltaUnit": "absolute" }
  },
  "taxBreakdown": {
    "groups": [
      { "code": "A", "taxCategory": "food", "vatRate": 10.0, "netAmount": 4654.91, "vatAmount": 465.49, "grossAmount": 5120.40 },
      { "code": "B", "taxCategory": "beverage_alcoholic", "vatRate": 20.0, "netAmount": 2635.17, "vatAmount": 527.03, "grossAmount": 3162.20 }
    ],
    "totals": { "netAmount": 7290.08, "vatAmount": 992.52, "grossAmount": 8282.60 }
  },
  "revenueByOrderType": [
    { "orderType": "dine_in", "orders": 140, "grossAmount": 7100.00 },
    { "orderType": "takeaway", "orders": 27, "grossAmount": 1182.60 }
  ],
  "revenueByCategory": [
    { "categoryId": 4, "category": "Mains", "grossAmount": 5230.10, "quantity": 210 },
    { "categoryId": 7, "category": "Drinks", "grossAmount": 1900.00, "quantity": 640 },
    { "categoryId": null, "category": null, "grossAmount": 1152.50, "quantity": 30 }
  ],
  "paymentMethods": [
    { "method": "card", "orders": 150, "grossAmount": 7600.20 },
    { "method": "cash", "orders": 17, "grossAmount": 682.40 }
  ],
  "tipsByPaymentMethod": [
    { "method": "card", "orders": 120, "amount": 180.00 },
    { "method": "cash", "orders": 12, "amount": 30.00 }
  ],
  "refunds": {
    "count": 3,
    "totalAmount": 142.54,
    "byReason": [
      { "reason": "wrong_item", "count": 2, "amount": 92.54 },
      { "reason": "other", "count": 1, "amount": 50.00 }
    ],
    "pendingCount": 1
  },
  "voidedOrders": { "count": 5, "amount": 214.00 },
  "labor": { "amount": 1800.00, "hours": 132.5, "shiftCount": 28, "isDemoData": true },
  "discounts": {
    "discountedOrders": 12,
    "discountedSharePercent": 7.2,
    "discountedRevenue": 340.00,
    "revenueForgone": 60.00,
    "averageDiscountPercent": 15.0,
    "topDiscountedItems": [
      { "name": "Discounted plate", "originalPrice": 20.00, "discountedPrice": 15.00, "discountPercent": 25.0, "quantityOrdered": 4 }
    ]
  },
  "salesInvoices": [
    { "invoiceNumber": "1042", "orderReference": "ord_ab12cd34", "date": "2026-08-24T18:32:10+02:00", "amount": 45.20 }
  ],
  "costs": {
    "inventoryPurchases": { "count": 4, "amount": 620.00 },
    "subscriptionFees": { "count": 1, "amount": 49.00 },
    "manualExpenses": { "count": 2, "amount": 1520.00, "recurringCount": 1, "recurringAmount": 1500.00 },
    "totalCosts": 2189.00,
    "purchaseOrders": [
      { "reference": "PO-AB12CD34EF", "supplierName": "Fresh Farms", "date": "2026-08-20T09:15:00+02:00", "quantity": 10, "unit": "kg", "amount": 35.00, "status": "sent" }
    ],
    "subscriptionInvoices": [
      { "id": 88, "invoiceNumber": "INV-2026-08", "date": "2026-08-05T10:00:00+02:00", "amount": 49.00, "hasDocument": true }
    ],
    "manualExpensesList": [
      {
        "id": 12, "name": "August rent", "category": "rent", "payee": "Landlord GmbH", "amount": 1500.00,
        "paymentMethod": "bank_transfer", "occurredAt": "2026-08-01T09:00:00+02:00", "description": null,
        "isRecurring": true, "recurrenceFrequency": "monthly",
        "attachmentUrl": null, "attachmentOriginalName": null
      },
      {
        "id": 13, "name": "Fix ice machine", "category": "maintenance_repairs", "payee": "ACME Repairs", "amount": 20.00,
        "paymentMethod": "card", "occurredAt": "2026-08-14T15:30:00+02:00", "description": "Compressor replacement",
        "isRecurring": false, "recurrenceFrequency": null,
        "attachmentUrl": "https://app.tavlo.com/media/financial-expenses/6/receipts/abc123.pdf", "attachmentOriginalName": "receipt.pdf"
      }
    ]
  },
  "inventoryWaste": {
    "count": 1,
    "amount": 6.00,
    "items": [
      { "itemName": "Lettuce", "quantity": 3.0, "unit": "kg", "amount": 6.00, "reason": "Spoiled overnight", "date": "2026-08-25T07:10:00+02:00" }
    ]
  },
  "tipDistribution": {
    "totalTips": 210.00,
    "eligibleTeamMemberCount": 3,
    "perPersonAmount": 70.00,
    "members": [
      { "name": "Anna", "role": "waiter" },
      { "name": "Bilal", "role": "kitchen" },
      { "name": "Deniz", "role": "waiter" }
    ]
  },
  "upcomingRecurringExpenses": [
    {
      "name": "Rent", "category": "rent", "recurrenceFrequency": "monthly", "amount": 1500.00,
      "lastOccurredAt": "2026-08-01T09:00:00+02:00", "nextExpectedAt": "2026-09-01T09:00:00+02:00", "isOverdue": false
    }
  ],
  "daily": [
    { "date": "2026-08-24", "dateTo": "2026-08-24", "orders": 17, "grossAmount": 643.60, "vatAmount": 58.42, "refundAmount": 0, "netAfterRefunds": 643.60 }
  ],
  "salesByHour": [
    { "hour": 0, "orders": 0, "grossAmount": 0 },
    { "hour": 12, "orders": 22, "grossAmount": 890.40 },
    { "hour": 19, "orders": 41, "grossAmount": 1820.10 },
    { "hour": 23, "orders": 0, "grossAmount": 0 }
  ],
  "cashFlow": {
    "monthly": [
      {
        "month": "2026-08-01", "cashIn": 1164.21, "cashOutInventory": 45.00, "cashOutLabor": 70.00,
        "cashOutSubscriptionFees": 49.00, "cashOutOtherExpenses": 1500.00, "cashOutRefunds": 30.00,
        "netCashFlow": -529.79
      }
    ],
    "totals": {
      "cashIn": 1164.21, "cashOutInventory": 45.00, "cashOutLabor": 70.00,
      "cashOutSubscriptionFees": 49.00, "cashOutOtherExpenses": 1500.00, "cashOutRefunds": 30.00,
      "netCashFlow": -529.79
    },
    "isLaborDemoData": true,
    "expandedToFullMonth": true
  },
  "loyaltyLiability": { "pointsOutstanding": 500, "pointValue": 0.02, "estimatedLiability": 10.00, "isDemoData": false }
}
```

**Field notes**

| Field | Notes |
|---|---|
| `currency` | `Vendor::currency` accessor (derived from `countries.currency` via the vendor's `country`) — never a stored/editable value. |
| `bucketUnit` | `day`, `week`, or `month` — the granularity `daily` was bucketed at, chosen by range length (≤62 days → day, ≤370 → week, else → month). The frontend must read this instead of assuming daily rows for long custom ranges. |
| `taxBreakdown.groups[].taxCategory` | Raw `menu_items.tax_category` slug. In practice this dataset has both `beverage_alcoholic`/`beverage_non_alcoholic` (the `tax_categories` seed slugs) and legacy `drinks_alcoholic`/`drinks_non_alcoholic` values for the same VAT treatment — not a display label; the frontend has translation keys for both spellings. |
| `revenueByOrderType[].orderType` | Raw `orders.order_type` value as stored (`dine_in`/`dine-in`, `takeaway`, `pickup`, ...) — both hyphen and underscore forms exist in historical data; the frontend normalizes before translating. |
| `refunds.pendingCount` | Refunds/disputes with `status=open` created in the period — informational, excluded from every monetary total because that money hasn't moved yet. |
| `costs.inventoryPurchases` | `quantity × unit_cost` on `inventory_purchase_orders` rows with `status != 'failed'`, windowed by `created_at` (the purchasing decision date — there is no delivery/received status to key off instead). |
| `costs.subscriptionFees` | Paid Tavlo subscription invoices for the vendor in the period. This is the only real "platform fee" this system charges today — there is no per-order application fee anywhere in this codebase, so none is reported. |
| `summary.costOfGoodsSold` | Real *consumption* cost: `inventory_stock_movements` rows with `type='order'` (the automatic per-order deduction `InventoryConsumptionService` writes) × each item's `cost_per_unit`. This is a different question from `costs.inventoryPurchases` (buying vs. selling) and will read `0` for any menu item without recipe/inventory tracking configured — it is a floor on true cost, not guaranteed complete. |
| `summary.grossProfit` / `grossMarginPercent` | `netRevenue − costOfGoodsSold`, and that divided by `netRevenue` as a percentage. `grossMarginPercent.delta` is `deltaUnit: "absolute"` (percentage points), not a percent-of-a-percent change. |
| `revenueByCategory[].category` | The vendor's own menu category name (e.g. "Mains") — free text they wrote, not a slug. `null` when the cart item's menu category was deleted or never set; the frontend renders that row as "Uncategorized". |
| `paymentMethods[].method` | Raw `orders.payment_method`. In practice this dataset has both `cash` and `Stripe` (capitalized, not `card`) for the same card-processing rail — the frontend has a translation alias for both spellings, same pattern as the `drinks_*`/`beverage_*` tax category aliasing above. Surfaced because cash needs manual bank deposit/reconciliation while card settles automatically via Stripe, the same distinction Toast/Square call out explicitly in their own reports. |
| `voidedOrders` | Orders with `cancelled_at` set in the period, windowed by `cancelled_at` (not `payment_confirmed_at` — cancelled orders don't have one). Purely informational: these were never paid and are not part of any revenue figure above. |
| `labor` / `summary.laborCost` | Real hours × `staff_shifts.hourly_rate` (snapshotted per shift, not read live from `team_members.hourly_wage`), windowed by `clock_in_at`. **`labor.isDemoData` is `true` whenever ANY shift behind the figure has `source='demo'`** (not "every" — a period mixing real and demo shifts must still be flagged, rather than presenting a partially-fabricated figure as fully measured) — there is no real staff clock-in feature anywhere in this codebase yet, so until one exists, all data here comes from `php artisan financials:seed-demo-labor`. The frontend must show a visible "demo data" indicator whenever this flag is true, never present it silently as measured. |
| `summary.costOfGoodsSoldPercent` / `laborCostPercent` | "Food Cost %" and "Labor Cost %" — `costOfGoodsSold`/`laborCost` each divided by `netRevenue`. The two halves restaurant operators watch day to day; `primeCostPercent` below is their sum, useful as one health check but hides which half is driving it. Both `deltaUnit: "absolute"` (percentage points). |
| `summary.primeCost` / `primeCostPercent` | `costOfGoodsSold + laborCost`, and that divided by `netRevenue` — the standard restaurant "prime cost" health metric (industry-typical healthy range is 55-65% of net revenue). Inherits the COGS caveat above, plus the labor demo-data caveat. This is a cost total, not a waterfall step — it is not itself subtracted anywhere below. |
| `summary.operatingExpenses` / `operatingProfit` / `netProfit` / `netMarginPercent` | Continues the waterfall past Gross Profit: `operatingExpenses` = manual "other costs" (`costs.manualExpenses.amount`) + `costs.subscriptionFees.amount`; `operatingProfit` = `grossProfit − laborCost − operatingExpenses`; `netProfit` = `operatingProfit − inventoryWaste.amount − totalRefunds`; `netMarginPercent` = `netProfit ÷ netRevenue`, `deltaUnit: "absolute"`. `costs.inventoryPurchases` is deliberately excluded from this chain — it is cash-basis purchasing already represented via `costOfGoodsSold` (what sold) or `inventoryWaste` (what spoiled); including it a third time would double-count the same outflow. `netProfit` inherits both the COGS caveat and the labor demo-data caveat above, so it is a floor on true profitability, not a guaranteed-complete figure. |
| `discounts` | Reuses the exact same logic as `VendorAnalyticsService::discounts()` — a cart line counts as discounted when the `MenuItem` version it points at had `has_discount=true` at the moment it was ordered (historically accurate because a discount change forces a new `MenuItem` version). `revenueForgone` = list-price gross minus actually-charged gross, both VAT-inclusive — the headline bookkeeping figure. `discountedSharePercent` is what share of ALL orders this period contained at least one discounted item; `averageDiscountPercent` is the plain average of each discounted `MenuItem`'s own configured `discount_percent` (not weighted by revenue or quantity). `topDiscountedItems` lists every distinct discounted `MenuItem` row seen this period — keyed by id, not name, so a mid-period price change (a new `MenuItem` version) appears as a separate row — ranked by revenue forgone, most impactful first. `originalPrice`/`discountedPrice` are the menu's own displayed unit prices (`MenuItem.price`/`discounted_price` as stored), not the VAT-grossed, quantity-scaled figures above — those answer a different question. `quantityOrdered` is a physical unit count: summed straight from `cart_items.quantity` with **no** VAT/gross scaling and, unlike the revenue figures, **not** divided by sharer count when a line is split across orders (`cartItemsFor()`'s owned+sharedInto expansion) — deduped by cart-item id first so a single shared plate is counted once, not once per person splitting it. This is real, already-computed-elsewhere data, not new logic. |
| `cashFlow` | A direct-method Cash Flow Statement, **always bucketed by calendar month** regardless of `bucketUnit` (many restaurant costs — rent, subscriptions, monthly supplier settlements — only happen once a month, so a daily/weekly bucket would mostly be empty rows). `monthly` covers every calendar month in `range` inclusive, even ones with zero activity (so a chart's bar spacing reflects real elapsed time, same reasoning as the `daily`-gap-filling the frontend does for the Revenue Trend chart) — this is the one place in this API that returns zero-filled rows rather than omitting inactive buckets. `cashOutInventory` is cash actually *paid* for stock (`costs.inventoryPurchases`), **not** `summary.costOfGoodsSold` — COGS is accrual-basis consumption (what was sold), purchases are the real cash outflow (what was paid to suppliers), and the two happen at different times; the P&L waterfall above deliberately excludes purchases to avoid double-counting against COGS, while a cash flow statement wants the opposite. `cashOutLabor` reuses the same `staff_shifts` data as `labor` above (see `isLaborDemoData`, same semantics as `labor.isDemoData`). There is no Investing or Financing section and no running cash balance — Tavlo tracks neither a bank balance nor any investing/financing activity (equipment purchases, loans, owner draws), so this only ever covers Operating Activities and never fabricates the rest; the frontend surfaces that as a visible caveat, not a silent omission. `expandedToFullMonth` is `true` whenever the selected report period (Today, This week, a custom range) is narrower than a calendar month — `monthly`/`totals` above still cover the *full* month(s) containing that selection (see `FinancialReportPeriod::cashFlowRange()`), not just the narrower window every other field in this response reflects; the frontend uses this flag to tell the vendor so. |
| `summary.tipRatePercent` / `tipsByPaymentMethod` | `orders.tip_amount` sits outside `amount` entirely (confirmed in `VendorAnalyticsService`'s own doc comment) — tips are never part of gross/net revenue above. `tipRatePercent` is tips ÷ gross revenue. `tipsByPaymentMethod` matters for bookkeeping specifically: a card tip is money the vendor physically holds via Stripe until paid out to staff (a liability), while a cash tip is already with the staff member who received it — the same cash/card distinction `paymentMethods` and `costs` draw elsewhere in this report. This is deliberately a narrower, bookkeeping-focused view than Analytics' own tip reporting (participation rate, service-speed correlation, etc.), which this feature does not duplicate. |
| `salesInvoices[].orderReference` | The order's `order_public_id`. The frontend passes this straight into the **existing** `OrderReceiptModal` component / `api.getOrderReceipt(vendorId, orderReference)` call already used on the Orders page — so a vendor can view, print, and download the actual invoice PDF from here, not just see a row of numbers. This endpoint was not added for this feature; it already existed. |
| `salesInvoices[].date` / `costs.purchaseOrders[].date` / `costs.subscriptionInvoices[].date` | Full ISO 8601 datetime (not date-only), so the frontend can render both the vendor's date format and time format settings. |
| `costs.subscriptionInvoices[].id` | The raw `invoices.id`, passed to the **existing** `GET /vendor/{vendorId}/billing/invoices/{id}/download` endpoint (`api.downloadInvoice()`) — each row is a real downloadable PDF/Stripe-hosted invoice, reusing Billing's own download flow rather than building a new one. |
| `costs.subscriptionInvoices[].hasDocument` | `true` when `invoices.pdf_url` or `stripe_hosted_url` is populated — mirrors the exact same check `BillingSubscription.tsx` already does client-side. Real seeded invoices in dev have neither set yet, so this is `false` there; the frontend disables the Download button rather than let the vendor hit a 404. |
| `costs.purchaseOrders` / `costs.subscriptionInvoices` | Itemized versions of the `inventoryPurchases`/`subscriptionFees` aggregates above, nested under `costs` (not a separate top-level "invoices" section) because both are expenditures — a subscription bill and a supplier order are money going out, and belong next to the cost totals they back up. |
| `costs.manualExpenses` / `manualExpensesList` | Vendor-entered costs with no other real data source in Tavlo (rent, repairs, equipment, insurance, ...) — see [§3](#3-manual-expenses-other-costs). Real, vendor-attested cash outflows, so unlike `inventoryWaste` below they ARE folded into `costs.totalCosts`. `recurringCount`/`recurringAmount` sub-total rows the vendor tagged `isRecurring` — this is a label only, it does not mean the amount recurs automatically; nothing generates future rows. |
| `inventoryWaste` | Real inventory shrinkage — `inventory_stock_movements` rows the vendor logged with `type='waste'` via the existing Inventory page's "Adjust Stock" action (spoilage, breakage, expiry). Cost = quantity × the item's current `cost_per_unit`, same valuation `costOfGoodsSold` uses. **Deliberately excluded from `costs.totalCosts`**: the cash for this stock was already spent when purchased (`costs.inventoryPurchases`) or already baked into `costOfGoodsSold` if it was consumed by an order — this section answers "how much of what I already paid for went to waste" (a loss/liability figure), not an additional cash outflow. Folding it into total costs would double-count the same money leaving the business. |
| `tipDistribution` | An **equal-split estimate** of the tip pool across `team_members` rows with `status='active'` — matches Square's simplest tip-pooling mode ("split equally among all tip-eligible team members"), not hours-weighted or role-weighted. The vendor owner is never a `TeamMember` row so is excluded automatically; `invited`/`suspended` members are excluded as not currently working. `perPersonAmount` is `null` when `eligibleTeamMemberCount` is 0. This is a bookkeeping estimate the vendor can act on, not a legally binding payout — every restaurant's actual tip-out policy differs, and the frontend must present it as such. |
| `loyaltyLiability` | A real accounting liability (deferred revenue / "breakage"): the vendor's outstanding loyalty-point balance × their configured `point_value`. `null` when loyalty isn't enabled. Deliberately **period-independent** — a balance-sheet snapshot as of now, not scoped to the selected report window, same reasoning as `upcomingRecurringExpenses`. Never a cash outflow; this figure lives nowhere else in this report (`VendorAnalyticsService::loyalty()` surfaces the same underlying number for operational analysis, not bookkeeping). `isDemoData` is `true` when any wallet behind the number came from `App\Console\Commands\SeedDemoLoyaltyData` (tagged `source='demo'` on `customer_loyalty_points`) rather than real points activity — no points-earning feature exists yet, so every vendor with loyalty enabled is demo-flagged until one ships, exactly like `laborCost.isDemoData` in the Profit & Loss summary. |
| `salesByHour` | Gross revenue and order count bucketed by hour-of-day (0-23, the vendor's own local timezone), summed across every day in `range` — answers "when do we get busy", distinct from `daily` (which/day). Always all 24 hours, even ones with zero orders, same zero-filling reasoning as `cashFlow.monthly` — the example above omits the empty hours for brevity, the real response always has 24 rows. Reuses the same order set as `summary`/`daily`, no extra query. |
| `upcomingRecurringExpenses` | **Deliberately independent of the requested `period`** — always reflects the vendor's most recent `is_recurring=true` `FinancialExpense` rows regardless of which date range is selected, so the reminder doesn't disappear just because the vendor is looking at "Today". Occurrences are grouped heuristically by `name+category+recurrenceFrequency` (there is no formal "series" concept in the schema — see `FinancialExpense`'s doc comment); `nextExpectedAt` is the last occurrence plus one frequency interval, and `isOverdue` is `true` when that projected date has already passed **`now()` in the vendor's own timezone (`Vendor::resolveTimezone()`)**, matching every other date computation in this service rather than the application server's timezone. This is a reminder computed from the vendor's own data, not a forecast or a commitment — nothing auto-creates future rows. |

**Errors**

| Status | Condition |
|---|---|
| 401 | Missing or invalid token |
| 403 | Token belongs to a different vendor, or is a team-member token (see below) |
| 404 | Vendor not found |
| 429 | Rate limit exceeded — 30 requests/minute per vendor, shared across all Analytics + Financial Reports routes (see [Rate limiting](#rate-limiting)) |
| 422 | `period=custom` without `from`/`to`, invalid `Y-m-d` format, or `to` before `from` |

A `200` with `{"locked": true, ...}` is returned instead of `403` when the vendor's plan lacks "Basic Analytics" — see [Plan gating](#plan-gating). That is a successful request, not an error.

---

## 3. Manual Expenses ("Other Costs")

CRUD for vendor-entered costs with no other real data source in Tavlo (rent, repairs, new equipment, insurance, licenses, ...). There is **no list/index endpoint** — these rows are surfaced through `GET /vendor/{vendorId}/financial-reports` itself (`costs.manualExpensesList`), the same single-source-of-truth pattern the rest of this report follows. The frontend re-fetches the report after any create/update/delete here.

### `POST /api/vendor/{vendorId}/financial-expenses/upload-attachment`

Uploads a receipt/invoice file (PDF or image) and returns its path/URL — the same two-step "upload first, reference the path in the create/update JSON" convention as `MenuItemController::uploadImage`. This endpoint never receives the rest of the expense fields; it only stores a file.

**Request:** `multipart/form-data`

| Field | Type | Notes |
|---|---|---|
| `attachment` | `file` | Required. `pdf`, `jpg`, `jpeg`, `png`, or `webp`. Max 10MB. |

**Response `200`:**
```json
{
  "attachmentPath": "financial-expenses/6/receipts/abc123.pdf",
  "attachmentUrl": "https://app.tavlo.com/media/financial-expenses/6/receipts/abc123.pdf",
  "attachmentOriginalName": "receipt.pdf"
}
```

### `POST /api/vendor/{vendorId}/financial-expenses`

Creates a manual expense.

**Request body:**

| Field | Type | Notes |
|---|---|---|
| `name` | `string` | Required, max 255. |
| `category` | `string` | Required. One of `rent`, `utilities`, `maintenance_repairs`, `equipment`, `supplies`, `insurance`, `marketing`, `professional_services`, `licenses_permits`, `vat_payment`, `other`. |
| `payee` | `string\|null` | Optional, max 255 — who was paid (e.g. "Landlord GmbH"). |
| `amount` | `number` | Required, `> 0`. |
| `paymentMethod` | `string\|null` | Optional. One of `cash`, `card`, `bank_transfer`, `other`. |
| `occurredAt` | `string` | Required. Any format `Carbon::parse()` accepts — the frontend sends full ISO 8601 datetime. |
| `description` | `string\|null` | Optional, max 2000. |
| `isRecurring` | `boolean` | Optional, default `false`. Metadata only — does **not** create future rows automatically. |
| `recurrenceFrequency` | `string\|null` | Required when `isRecurring=true`. One of `daily`, `weekly`, `monthly`, `yearly`. |
| `attachmentPath` | `string\|null` | The `attachmentPath` returned by the upload-attachment call above, or omit/`null` for no attachment. **Must start with `financial-expenses/{vendorId}/receipts/`** (this vendor's own upload prefix) — any other value is rejected with `422`. Without this check a vendor could point an expense at an arbitrary file already served under `/media/*` (e.g. another vendor's logo) and later have it permanently deleted via `update()`/`destroy()`, which delete whatever path is stored with no other ownership check. |
| `attachmentOriginalName` | `string\|null` | The original filename, for display — from the same upload response. |

**Response `201`:** `{ "data": { ...same shape as costs.manualExpensesList[] entries... } }`

### `PUT /api/vendor/{vendorId}/financial-expenses/{expenseId}`

Same fields as create, all optional (`sometimes`). **Attachment behavior:** only send `attachmentPath`/`attachmentOriginalName` when the attachment actually changed (new upload, or explicitly cleared by sending `attachmentPath: null`) — omitting the key entirely leaves the existing attachment untouched. When the attachment does change, the previous file is deleted from storage.

**Response `200`:** `{ "data": { ... } }`

### `DELETE /api/vendor/{vendorId}/financial-expenses/{expenseId}`

Deletes the expense and its attachment file (if any).

**Response `200`:** `{ "message": "Deleted." }`

**Errors** (all four endpoints)

| Status | Condition |
|---|---|
| 401 | Missing or invalid token |
| 403 | Token belongs to a different vendor, is a team-member token, or the vendor's current plan lacks "Basic Analytics" (see [Plan gating](#plan-gating) — unlike `GET /financial-reports`, these write routes 403 rather than returning a locked JSON body) |
| 404 | Vendor or expense not found |
| 422 | Validation failure (e.g. `isRecurring=true` without `recurrenceFrequency`, invalid `category`, unsupported file type/size) |
| 429 | Rate limit exceeded — 20 requests/minute per vendor, this feature's own limiter (see [Rate limiting](#rate-limiting)) |

---

## Access Control

Registered under the same middleware group as Analytics: `auth:vendor,team_member` + `vendor.staff.access`. The route names `financial-reports.index` and `financial-expenses.*` are **not** added to `EnsureStaffCanAccessVendorRoute`'s allow-lists, so any `TeamMember` token — kitchen, waiter, or any future role — receives `403` automatically on all of them (report read AND expense CRUD). Only the vendor-owner token can view or manage financial data. If a specific staff role should get access later, extend the allow-list in `app/Http/Middleware/EnsureStaffCanAccessVendorRoute.php` — that file is intentionally not touched by this feature.

---

## Database Schema Reference

Three migrations are new; everything else is computed at request time from tables that already existed:

| Table | Used for |
|---|---|
| `orders` | `amount`, `vat_amount`, `service_fee`, `tip_amount`, `order_type`, `payment_confirmed_at` |
| `cart_items` + `menu_items` | Per-line VAT decomposition via the existing `TaxCalculationService::computeTaxGroups()` |
| `tax_categories` | VAT rate per country + tax category slug |
| `refunds` | Approved refund totals (`status=approved`, `type=refund`), windowed by `resolved_at` |
| `inventory_purchase_orders` | Inventory purchasing cost estimate (`costs.inventoryPurchases`) |
| `inventory_stock_movements` + `inventory_items` | Real cost of goods sold (`summary.costOfGoodsSold`) AND real inventory waste/loss (`inventoryWaste`, `type='waste'` rows — logged via the existing Inventory page's "Adjust Stock" action, not a new feature). |
| `invoices` / `subscriptions` | Subscription/platform fee cost |
| `menu_categories` | Revenue by category |
| `staff_shifts` (new table) + `team_members.hourly_wage` (new column) | Labor cost / prime cost. Both introduced by the labor-cost feature — nothing like a timesheet or wage rate existed anywhere before. See `2026_08_29_120001_add_hourly_wage_to_team_members_table.php` and `2026_08_29_120002_create_staff_shifts_table.php`. |
| `menu_items.has_discount` / `discounted_price` / `discount_percent` | Discounts & promotions reporting — the exact same fields `VendorAnalyticsService::discounts()` already reads. |
| `team_members` (`status`, `role`) | Tip distribution eligibility (`status='active'`) and per-member role display — no schema change, existing columns. |
| `financial_expenses` (new table) | Manual "other cost" entries — see [§3](#3-manual-expenses-other-costs). Introduced by this feature; nothing like it existed before. See `2026_08_29_112322_create_financial_expenses_table.php`. Also the source for `upcomingRecurringExpenses` (`is_recurring=true` rows, queried independently of the report's `period`). |
