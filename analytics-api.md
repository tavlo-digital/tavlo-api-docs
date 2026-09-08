# Analytics API

**Base URL prefix:** `/api/vendor/{vendorId}`
**Authentication:** `Authorization: Bearer {token}` — vendor-owner token only. Team-member tokens receive `403` (see [Access Control](#access-control)).
**Content-Type:** `application/json`

`vendorId` accepts either the `vendor_public_id` (e.g. `V-ABC12345`) or the numeric primary key.

This file documents all six routes on `AnalyticsController`. The first two (`GET /analytics`, `GET /analytics/insights`) pre-date this file and are covered here at an overview level — their exhaustive field-by-field behavior lives in `App\Services\Analytics\VendorAnalyticsService` and `App\Services\Analytics\InsightEngine`, each documented inline per-section/per-rule in code, and exercised by `tests/Feature/Analytics/VendorAnalyticsApiTest.php`. `/ask` and `/suggested-questions` are the queryable-assistant feature. `/forecast` is Tavlo's first forward-looking analytics feature — documented in full below, exercised by `tests/Feature/Analytics/VendorForecastApiTest.php`.

---

## Table of Contents

1. [Overview & Concepts](#1-overview--concepts)
2. [Get Analytics](#2-get-analytics)
3. [Get Insights](#3-get-insights)
4. [Ask the Assistant](#4-ask-the-assistant)
5. [Get Suggested Questions](#5-get-suggested-questions)
6. [Get Forecast](#6-get-forecast)
7. [Access Control](#access-control)

---

## 1. Overview & Concepts

Operational/behavioral reporting for a vendor's own trading data — order volume, peak times, service speed, tips, retention, reviews, inventory, loyalty, and (new) a natural-language layer on top of all of it. This is **not** the bookkeeping report; see `financial-reports-api.md` for VAT/cost/profit figures, which deliberately windows by payment date rather than `created_at` and can legitimately disagree with this endpoint on totals for that reason.

### Periods
Same `AnalyticsPeriod` resolver across all four routes:

| Key | Meaning |
|---|---|
| `daily` | Last 7 days |
| `weekly` (default) | Last 12 weeks |
| `monthly` | Last 12 months |
| `today` | Vendor's current calendar day |
| `custom` | Vendor-chosen `from`/`to`, inclusive, max 366 days |

All boundaries are computed in the vendor's own timezone (`Vendor::resolveTimezone()`).

A `custom` range longer than 366 days is not rejected — it's silently shortened to 366 days from `from`. This is surfaced in the response so a client can tell the vendor: `range.truncated` is `true` and `range.maxCustomDays` is `366` whenever this happened (always `false`/`366` otherwise). `range.from`/`range.to` always reflect the actual (possibly shortened) window used, never the raw request. 2026-09-07 audit finding: previously this had no visible signal at all — a vendor could pick a multi-year range and see fewer months than they asked for with nothing indicating why.

### `hasData`
`GET /analytics` returns `hasData: false` when the vendor has zero confirmed orders in the window — every other section is still present but reads as empty/null rather than omitted, so the frontend has one flag to gate an empty state rather than checking each section individually.

### Two ways to reach an insight's substance
- **`GET /analytics/insights`** generates a list of insight cards — deterministic, sample-gated alerts, each with `whatHappening`/`whyMatters`/`suggestedAction` prose already written out. This only surfaces a rule when its figure is *notable* (e.g. a wide quiet-hour gap), not just measurable.
- **`POST /analytics/insights/ask`** answers a vendor's own question against the same underlying data — either a follow-up on one of those cards (`insightId`), or a standalone topic question ("what's my busiest hour?") answered whenever there's *enough sample to measure it*, regardless of whether it's alarming enough to have generated a card. Neither endpoint calls out to an LLM — every answer is a fixed sentence built from a measured figure, matched by a fixed keyword list, so a vendor's business data never leaves the request and an unmatched question gets an honest "can't answer that yet" rather than an invented answer.

### Plan gating

All five routes below (`GET /analytics`, `GET /analytics/insights`, `POST /analytics/insights/ask`, `GET /analytics/insights/suggested-questions`, `GET /analytics/forecast`) check the vendor's *current* subscription plan for a feature named **"Basic Analytics"** before doing any real work — `AnalyticsController::hasAnalyticsAccess()` reads this from the vendor's live `plan.features` (via `Vendor::subscriptions()->latest()->first()?->plan?->features`), never a hardcoded plan tier or name. Which plan(s) actually carry "Basic Analytics" is entirely admin-configurable in `plan_features` and can change without a code deploy.

A vendor whose plan lacks the feature gets a **locked response** instead of the real payload — same `200` status, different shape, cheap to compute (no full analytics pipeline runs):

```json
{
  "locked": true,
  "requiredFeature": "Basic Analytics",
  "currency": "EUR",
  "preview": { "ordersToday": 4, "grossRevenueToday": 87.5 }
}
```

`GET /analytics` is the only route whose locked response includes a real (not fake) `preview` — today's order count and gross revenue, from a small dedicated query, not the full pipeline. The other four routes' locked responses carry the same `locked`/`requiredFeature` pair plus whatever fields their own payload always requires: `/insights` and `/insights/suggested-questions` still return `period`/`currency` with an empty list; `/insights/ask` still echoes back `period`/`question`/`insightId`, with `matched: false`, `answer: ''`, and empty/null everywhere else; `/forecast` still returns `timezone`/`generatedAt`/`minWeeksRequired`/`available: false` — see each route's section below for its exact locked shape. Frontend clients should check `locked` before reading any other field; `tavlo-vendor`'s `AnalyticsResponse` type models this as a discriminated union (`AnalyticsPayload | AnalyticsLockedPayload`) in `lib/analytics.ts`.

### Rate limiting

All five routes (plus Financial Reports' `GET /financial-reports`) share one named limiter, **30 requests/minute per authenticated vendor** (not per IP — a shared restaurant network isn't penalized as a single client). Added because these endpoints have no caching for most of their sub-aggregations and rebuilding the full payload is the most expensive read in the vendor API; the limiter is keyed so hitting it on one of these six routes counts against the same budget as hitting any other. Exceeding it returns `429` with Laravel's standard rate-limit response body (`Retry-After` header included).

---

## 2. Get Analytics

### `GET /api/vendor/{vendorId}/analytics`

Returns the full metrics payload for one period.

**Query Parameters**

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `period` | `string` | `weekly` | One of `daily`, `weekly`, `monthly`, `today`, `custom` |
| `from` | `string` | — | `Y-m-d`. Used when `period=custom` |
| `to` | `string` | — | `Y-m-d`. Used when `period=custom` |

**Response `200` (top-level shape):**

```json
{
  "period": "weekly",
  "currency": "EUR",
  "timezone": "Europe/Vienna",
  "range": { "from": "...", "to": "...", "label": "Last 12 weeks", "periodLabel": "Last 12 weeks", "comparisonLabel": "vs previous 12 weeks", "truncated": false, "maxCustomDays": 366 },
  "hasData": true,
  "summary": { "orders": { "value": 167, "previous": 150, "delta": 17, "deltaUnit": "absolute" }, "grossRevenue": {}, "paidOrders": {}, "avgOrderValue": {} },
  "trend": [],
  "menu": [],
  "soldOut": [],
  "categories": [],
  "discounts": {},
  "payments": {},
  "tips": {},
  "service": {},
  "channels": {},
  "cancellations": {},
  "peak": { "busiest": {}, "quietest": {}, "days": [], "hours": [] },
  "customers": {},
  "reservations": {},
  "retention": {},
  "reviews": { "count": 0, "averageRating": null, "unanswered": 0, "unansweredCritical": 0 },
  "inventory": {},
  "loyalty": {},
  "modifiers": {}
}
```

Every metric that carries a period-over-period comparison uses the shared `{ value, previous, delta, deltaUnit }` shape (`deltaUnit: "percent"` or `"absolute"`).

### `modifiers` — paid add-ons, free add-ons, and removable items

*New.* Business-insight analytics for menu customization — not just selection counts, but recommendations: what to promote, what to reconsider or remove, whether a default item composition should change, where a paid add-on opportunity is missing, and whether a paid add-on's price looks off relative to its own adoption. Built by `App\Services\Analytics\ModifierAnalyticsService`, exercised by `tests/Feature/Analytics/ModifierAnalyticsApiTest.php`.

Tavlo has two parallel, coexisting customization systems, both covered in one unified view here:
- **"addon"** system — free-text definitions on the menu item itself (`paid_addons` / `free_addons` / `removable_items` configured per dish).
- **"modifier"** system — reusable, relational modifier groups/options (e.g. a "Size" group shared across several dishes). A group's `type: "remove"` is this system's analogue of a removable item.

Every list item shares one shape:

```json
{
  "system": "addon",
  "kind": "paid_addon",
  "name": "Extra Cheese",
  "groupName": null,
  "menuItemName": "Bruschetta al Pomodoro",
  "categoryName": "Starters",
  "allowsMultiple": false,
  "price": 2.00,
  "timesSelected": 41,
  "eligible": 82,
  "selectionRatePercent": 50.0,
  "revenue": 82.00
}
```

`kind` is `paid_addon` / `free_addon` / `removable_item` (classified by price, or by a modifier group's `remove` type — never a separate boolean). `eligible` is how many times the *owning menu item itself* was ordered this period — the number of times a guest could have picked this modifier — so `selectionRatePercent` is a real adoption/removal rate, not a raw count that favors whatever dish sells the most. `price` is the average gross price actually charged once there's real revenue (falls back to the currently-configured price for a never-selected entry). Selection counts and `eligible` dedupe a shared/split cart line the same way `VendorAnalyticsService::aggregateLines()` does (so a dish split across two orders counts once, not twice); `revenue` does not dedupe — it's attributed to each order's own share, same as every other money figure in this file.

Below `MIN_SAMPLE` (10) eligible occurrences, an entry is excluded from every ranked list — a single lucky or unlucky order never produces a confident-sounding recommendation.

```json
{
  "available": true,
  "ordersAnalyzed": 167,
  "minSampleSize": 10,
  "summary": {
    "totalAddonModifierRevenue": 612.40,
    "revenueSharePercent": 8.3,
    "avgModifierRevenuePerOrder": 3.67,
    "configuredNeverSelectedCount": 2
  },
  "paidAddonsRevenue": [ "...top paid add-ons/options by revenue, highest first (#1, #5)" ],
  "freeAddonsImpact": {
    "available": true,
    "baselineAvgOrderValue": 44.20,
    "items": [ "...free add-ons, each with avgOrderValueWithAddon and upliftPercent vs the baseline (#2)" ]
  },
  "mostPopular": [ "...highest selectionRatePercent, any kind (#3, #4)" ],
  "leastPopular": [ "...lowest selectionRatePercent, excludes removable items (#4)" ],
  "promotionCandidates": [ "...selectionRatePercent >= 40% — candidates to promote or offer more prominently (#6)" ],
  "underperforming": [ "...selectionRatePercent <= 5%, each with a `recommendation` string — covers both 'rarely selected, reposition' (#7) and 'low-value, consider removing' (#12)" ],
  "removableItems": [ "...every removable item/component with a `removalRatePercent` (#8)" ],
  "significantRemovals": [ "...removalRatePercent >= 20%, each with a `recommendation` string proposing the default composition change (#9, #10)" ],
  "newAddonOpportunities": [
    { "menuItemName": "Caprese Salad", "categoryName": "Starters", "eligibleQty": 58, "categoryBestAdoptionPercent": 50.0 }
  ],
  "pricingAnalysis": {
    "available": true,
    "baselineAdoptionRatePercent": 28.4,
    "items": [ "...paid add-ons/options with >= 15 eligible occurrences, each with `verdict`: possibly_underpriced | possibly_overpriced | appropriately_priced | insufficient_data, plus a per-item `note` (#13, #14)" ]
  },
  "note": "Popularity, promotion, and pricing read-outs below are derived from this period's order behavior — treat them as analytical signals to investigate, not definitive conclusions. A pricing verdict in particular reflects adoption relative to your own catalog, not a controlled price test."
}
```

**`newAddonOpportunities`** (#11) is the one list not built from the shared item shape above: it flags a menu item with real order volume and **no paid add-on configured at all**, sitting in a category where a sibling item's paid add-on already sees real, meaningful adoption (`categoryBestAdoptionPercent` >= 15%) — i.e. real in-category evidence that guests are willing to pay for an extra, not a guess. Scoped to the "addon" system only.

**`pricingAnalysis` verdicts** are a heuristic read of an add-on's adoption rate against the vendor's own average paid-add-on adoption rate this period (`baselineAdoptionRatePercent`) — not a price experiment or elasticity model. `possibly_underpriced` fires at ≥1.5× the baseline; `possibly_overpriced` at ≤0.4× the baseline; everything else reads `appropriately_priced`. Every response carries the top-level `note` disclaiming this explicitly, and every pricing item repeats it with its own reasoning in a per-item `note`.

When the vendor has no paid orders with any line-item data in the selected period, the whole block collapses to `{ "available": false, "ordersAnalyzed": 0, "minSampleSize": 10 }` rather than empty-but-present lists.

**Errors**

| Status | Condition |
|---|---|
| 401 | Missing or invalid token |
| 403 | Token belongs to a different vendor, or is a team-member token |
| 404 | Vendor not found |
| 429 | Rate limit exceeded — 30 requests/minute per vendor, shared across all Analytics + Financial Reports routes (see [Rate limiting](#rate-limiting)) |

---

## 3. Get Insights

### `GET /api/vendor/{vendorId}/analytics/insights`

Runs `InsightEngine::derive()` over the same payload `GET /analytics` builds and returns a sorted list of generated insight cards. Kept as a separate endpoint so the main dashboard renders without waiting on it, and so rules can be tightened without touching the metrics contract.

Stays silent (`insights: []`) whenever `hasData` is `false` or total orders are below `InsightEngine::MIN_ORDERS` (25) — a quiet vendor sees an empty list rather than confident conclusions drawn from a handful of orders. Individual rules have their own, sometimes higher, sample gates on top of that floor.

**Query Parameters:** same as [§2](#2-get-analytics).

**Response `200`:**

```json
{
  "period": "weekly",
  "currency": "EUR",
  "insights": [
    {
      "id": "quiet-hour",
      "category": "operational",
      "priority": 2,
      "title": "Tue at 15:00 is your quietest slot",
      "description": "2 orders, against 20 at your peak (Fri 19:00)",
      "impact": { "value": 18, "unit": "orders", "label": "order gap vs peak hour" },
      "confidence": "medium",
      "whatHappening": "Across last 12 weeks, Tue at 15:00 recorded 2 orders while your busiest hour recorded 20.",
      "whyMatters": "Staff, rent and utilities are paid the same in a quiet hour as a busy one, so idle capacity costs the same as full capacity.",
      "suggestedAction": "Run a time-boxed offer over that window and measure it against this same hour next period.",
      "actionScreen": "loyalty",
      "actionLabel": "Set up an offer"
    }
  ]
}
```

`id` is the stable slug used as `insightId` in [§4](#4-ask-the-assistant) to follow up on a specific card. The full set of possible ids and their firing conditions are each documented as a rule method in `App\Services\Analytics\InsightEngine`.

**Errors:** same as [§2](#2-get-analytics).

---

## 4. Ask the Assistant

### `POST /api/vendor/{vendorId}/analytics/insights/ask`

*New — queryable-assistant feature.* Answers one free-text question, either standalone or as a follow-up on a specific insight card, using `App\Services\Analytics\InsightQueryEngine`.

**Request body**

| Field | Type | Notes |
|---|---|---|
| `question` | `string` | Required, 2–300 chars. Free text — matched by keyword, not stored or forwarded anywhere. |
| `insightId` | `string\|null` | Optional. An `id` from the `insights` array in [§3](#3-get-insights). When present, the question is answered as a follow-up on that specific card instead of matched against the standalone topic list. |
| `period` | `string` | Optional, same values as [§2](#2-get-analytics). Default `weekly`. |
| `from` / `to` | `string` | `Y-m-d`. Required when `period=custom`. |

**Behavior**

- **With `insightId`:** the engine re-derives that insight from the current data. If it's still active, the answer is drawn verbatim from that card's own text — `why`/`matter` in the question selects `whyMatters`; `what should`/`how do i`/`fix`/`action` selects `suggestedAction`; anything else defaults to `whatHappening`. This guarantees the answer never contradicts the card the vendor is looking at. If that insight is no longer active (its numbers changed, or the id doesn't exist), `matched` is `false` with an explanatory message — never a stale or fabricated answer.
- **Without `insightId`:** the question is matched against a fixed, ordered list of topics by keyword (busiest/quietest hour, slowest day, payment failures, service speed, retention, reviews, top customers, discounts, tips, sold-out items, menu performance, inventory, loyalty, and a general orders/revenue summary). The first topic whose keywords match is used.
  - If a topic matches **and** there's enough sample to answer it, `matched: true` with a direct, data-backed sentence.
  - If a topic matches but there isn't enough data yet (e.g. loyalty asked about on a vendor who has never enabled it), `matched: false`, `topic` is still set, and the message says so specifically rather than falling back to a generic "I don't understand."
  - If nothing matches at all, `matched: false`, `topic: null`, with a generic fallback message.
- **`suggestedQuestions`** is always present in the response (regardless of `matched`) — see [§5](#5-get-suggested-questions) for how it's chosen.

**Response `200`:**

```json
{
  "period": "weekly",
  "currency": "EUR",
  "question": "What's my busiest hour?",
  "insightId": null,
  "matched": true,
  "topic": "busiest-quietest",
  "answer": "Your busiest slot is Fri at 19:00 with 20 orders. Your quietest is Tue at 15:00 with 2.",
  "data": { "busiest": { "day": "Fri", "hour": 19, "orders": 20 }, "quietest": { "day": "Tue", "hour": 15, "orders": 2 } },
  "confidence": "medium",
  "suggestedQuestions": ["Which day of the week is slowest?", "How are my tips trending?"]
}
```

`data` is `null` when `matched` is `false`. `confidence` (`high`/`medium`/`low`) is `null` when `matched` is `false`, and otherwise reflects the same sample-size scale `GET /analytics/insights` uses (high ≥200, medium ≥50, else low).

**Topic reference** (the `topic` value when `matched` is `true`, or when a topic matched but lacked data):

| Topic | Answers questions about |
|---|---|
| `busiest-quietest` | Peak/quiet hour and how many orders each carried |
| `slowest-day` | Which day of the week is slowest/busiest, against the daily average |
| `payment-failures` | Payment failure rate and count |
| `service-speed` | Table turnover vs. baseline, lunch vs. dinner service time |
| `retention` | First-time guests and their 30-day return rate |
| `reviews` | Unanswered reviews, how many are critical (≤3★) |
| `revenue-concentration` | Share of revenue from the top fifth of identified guests |
| `discounts` | Whether discounted orders produce bigger baskets, revenue given away |
| `tips` | Overall tip rate, tip rate by service speed, cash vs. digital tip capture |
| `sold-out` | Which items went unavailable this period, and how often |
| `menu-items` | Top sellers by quantity, biggest decliner vs. previous period |
| `inventory` | At-risk stock, slow movers, waste, stuck purchase orders, supplier price increases |
| `loyalty` | Visit-frequency change since joining, redemption rate |
| `summary` | Orders, revenue, average order value (the catch-all for general "how am I doing" questions) |
| `insight-followup` | Only set when `insightId` was supplied and the card is still active |

**Errors**

| Status | Condition |
|---|---|
| 401 | Missing or invalid token |
| 403 | Token belongs to a different vendor, or is a team-member token |
| 404 | Vendor not found |
| 429 | Rate limit exceeded — 30 requests/minute per vendor, shared across all Analytics + Financial Reports routes (see [Rate limiting](#rate-limiting)) |
| 422 | `question` missing/too short/too long, or `period=custom` without a valid `from`/`to` |

---

## 5. Get Suggested Questions

### `GET /api/vendor/{vendorId}/analytics/insights/suggested-questions`

*New — queryable-assistant feature.* The empty-state prompt for the assistant's question box: example questions the vendor can usefully ask **right now**, filtered to topics `InsightQueryEngine` actually has enough data to answer for this vendor and period. A brand-new vendor with only a handful of orders sees just `summary`-level questions; nothing prompts them to ask about loyalty or inventory data that doesn't exist yet.

**Query Parameters:** same as [§2](#2-get-analytics).

**Response `200`:**

```json
{
  "period": "weekly",
  "currency": "EUR",
  "questions": [
    "How many orders did I get this period?",
    "What is my busiest hour?",
    "Which day of the week is slowest?"
  ]
}
```

Up to 6 questions, in the same fixed topic order as the table in [§4](#4-ask-the-assistant).

**Errors:** same as [§2](#2-get-analytics).

---

## 6. Get Forecast

### `GET /api/vendor/{vendorId}/analytics/forecast`

*New — Tavlo's first forward-looking analytics feature.* Everything else in this file reports what already happened; this answers "how many orders should I expect this week?" from a simple, explainable 8-week average — not a machine-learning model. See `App\Services\Analytics\ForecastService` for the full calculation, and `tests/Feature/Analytics/VendorForecastApiTest.php` for the behaviors it locks down.

Kept as its own endpoint, same reasoning as `/insights`: the main dashboard renders without waiting on it, and its lookback is fixed (always the last 8 completed calendar weeks) rather than following the `period` tab the vendor has selected elsewhere on the page.

**Query Parameters:** none. The window is always the vendor's last 8 completed calendar weeks (Mon–Sun, vendor timezone) plus the in-progress current week.

### Honesty gate

A forecast requires **8 full completed weeks of order history** to exist before anything is shown — a vendor whose first order was more recent than that sees `available: false` and exactly how many weeks they still need, never a projection built from partial history presented as if it were fully measured. This mirrors the `available`-gated-null pattern used throughout `GET /analytics` (Retention, Loyalty, service-timing baselines) — it is a different convention from `isDemoData` (Financial Reports' labor/loyalty sections), because a thin forecast is real data, just not enough of it yet, not synthetic seeded data standing in for a missing feature.

**Response `200` — insufficient history:**

```json
{
  "currency": "EUR",
  "timezone": "Europe/Vienna",
  "generatedAt": "2026-06-17T12:00:00.000000Z",
  "minWeeksRequired": 8,
  "available": false,
  "weeksOfHistory": 3
}
```

**Response `200` — available:**

```json
{
  "currency": "EUR",
  "timezone": "Europe/Vienna",
  "generatedAt": "2026-06-17T12:00:00.000000Z",
  "minWeeksRequired": 8,
  "available": true,
  "weeksOfHistory": 8,
  "method": "8-week average of confirmed orders per calendar week, with a first-half-vs-second-half trend read",
  "history": [
    { "weekStart": "2026-04-20", "weekEnd": "2026-04-26", "label": "W17", "orders": 132, "revenue": 4210.5 }
  ],
  "projection": {
    "projectedOrders": 143,
    "projectedOrdersLow": 121,
    "projectedOrdersHigh": 165,
    "avgOrdersPerWeek": 143.4,
    "projectedRevenue": 4675.1,
    "avgRevenuePerWeek": 4675.1,
    "trend": { "direction": "up", "changePercent": 12.4, "recentAvgOrders": 151.5, "earlierAvgOrders": 134.8 },
    "confidence": "medium"
  },
  "currentWeek": {
    "weekStart": "2026-06-15",
    "weekEnd": "2026-06-21",
    "daysElapsed": 3,
    "ordersSoFar": 54,
    "revenueSoFar": 1780.25,
    "expectedByNow": 49.2,
    "paceDelta": 4.8,
    "paceStatus": "ahead"
  },
  "dayOfWeekPattern": [
    { "day": "Mon", "avgOrders": 15.2, "shareOfWeek": 10.6 }
  ],
  "note": "Simple average-based projection built from your last 8 weeks of confirmed orders. Treat it as a planning reference, not a guarantee."
}
```

**Field notes**

| Field | Notes |
|---|---|
| `history` | 8 entries, oldest→newest. `orders` = confirmed, non-cancelled orders in that calendar week (same definition as `summary.orders` in `GET /analytics`). `revenue` = gross amount of that week's **paid** orders only (same definition as `summary.grossRevenue`). |
| `projection.projectedOrders` | The headline figure — the straight 8-week average, rounded. |
| `projection.projectedOrdersLow` / `High` | `projectedOrders` ± one sample standard deviation across the 8 weeks, floored at 0. Collapses to a single point on a perfectly flat history. |
| `projection.trend` | Compares the average of the most recent 4 weeks against the earlier 4 — `changePercent` is `null` when the earlier average is 0 (nothing to divide by), and `direction` falls back to a plain increase/decrease/flat comparison in that case. |
| `projection.confidence` | `high` / `medium` / `low`, from the coefficient of variation (stddev ÷ mean) across the 8 weeks: `<15%` high, `<35%` medium, else low. Always `low` when the average is 0. |
| `currentWeek.expectedByNow` | The average number of orders the vendor's own history had accumulated by this same day-of-week, across the 8 history weeks — a day-level pace comparison, not hour-matched. |
| `currentWeek.paceStatus` | `ahead` / `behind` / `on_pace`, from `ordersSoFar` vs `expectedByNow` with a ±10% band for "on pace". `ahead` (with `on_pace` only when literally zero) whenever `expectedByNow` is 0, since any order at all is ahead of a historical zero. |
| `dayOfWeekPattern` | 7 entries, Mon→Sun. `avgOrders` is that weekday's average across the 8 history weeks; `shareOfWeek` is that weekday's share of the 8-week total order volume. |

**Errors**

| Status | Condition |
|---|---|
| 401 | Missing or invalid token |
| 403 | Token belongs to a different vendor, or is a team-member token |
| 404 | Vendor not found |
| 429 | Rate limit exceeded — 30 requests/minute per vendor, shared across all Analytics + Financial Reports routes (see [Rate limiting](#rate-limiting)) |

---

## Access Control

Registered under the same middleware group as Financial Reports: `auth:vendor,team_member` + `vendor.staff.access`. The route names `analytics`, `analytics.insights`, `analytics.insights.ask`, `analytics.insights.suggestedQuestions`, and `analytics.forecast` are **not** added to `EnsureStaffCanAccessVendorRoute`'s allow-lists, so any `TeamMember` token — kitchen, waiter, or any future role — receives `403` automatically on all six. Only the vendor-owner token can view analytics, use the assistant, or view the forecast.
