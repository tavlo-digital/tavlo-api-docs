# Restaurant Public API

Public endpoints used by the restaurant-facing website.

**Base URL prefix:** `/api/restaurant`

---

## Get Subscription Plans

Returns active subscription plans created by the admin, including their prices,
assigned features, and a dynamically generated comparison table.

**Method:** `GET`

**URL:** `/api/restaurant/plans`

**Authentication:** not required.

**Request body:** none.

**Response (200):**

```json
{
  "success": true,
  "data": {
    "hero": {
      "title": "Plans that fit your",
      "titleHighlight": "needs",
      "billingOptions": {
        "default": "monthly",
        "monthlyLabel": "Monthly",
        "yearlyLabel": "Yearly",
        "yearlyBadge": "Save up to 30%"
      }
    },
    "plans": [
      {
        "id": "basic",
        "name": "Basic",
        "link": "https://vendor.example.com/activate?plan=1",
        "prices": {
          "monthly": {
            "amount": 99,
            "currency": "EUR",
            "period": "/Month"
          },
          "yearly": {
            "amount": 831.6,
            "currency": "EUR",
            "period": "/Year",
            "monthlyEquivalent": 69.3
          }
        },
        "features": [
          "Basic Menu Management",
          "Menu Categories"
        ]
      }
    ],
    "logoSection": {
      "title": "Join hundreds of restaurants with Tavlo",
      "logos": [
        {
          "name": "Qormuz",
          "image": "https://cdn.example.com/logos/qormuz.svg"
        }
      ]
    },
    "comparison": {
      "title": "Compare Plans",
      "plans": ["basic"],
      "features": [
        {
          "id": "basic-menu-management",
          "label": "Basic Menu Management",
          "availability": {
            "basic": true
          }
        }
      ]
    }
  }
}
```

Only plans where `subscription_plans.is_active = true` are returned. Plan and
feature IDs in this response are URL-friendly slugs generated from their admin
names. `monthlyEquivalent` is calculated from the saved yearly price divided by
12. Each plan's `link` uses `VENDOR_FRONTEND_URL` and opens the vendor
subscription flow for that plan. Logged-out users are sent to login, can switch
to registration if needed, and return to the selected subscription after
authentication.
