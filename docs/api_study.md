# Uber Web GraphQL API — Fare Estimation

**Endpoint:** `POST https://m.uber.com/go/graphql`  
**Operation:** `Products`  
**Discovered:** June 2026 via DevTools network interception on `m.uber.com`

---

## Table of Contents

1. [Request Structure](#1-request-structure)
2. [Headers](#2-headers)
3. [Authentication](#3-authentication)
4. [Variables](#4-variables)
5. [Response Structure](#5-response-structure)
6. [Fields We Collect](#6-fields-we-collect)
7. [The Meta Field](#7-the-meta-field)
8. [Schema Mapping](#8-schema-mapping)
9. [Irrelevant Query Parts](#9-irrelevant-query-parts)
10. [Trimmed Query](#10-trimmed-query)
11. [Known Unknowns](#11-known-unknowns)

---

## 1. Request Structure

Every request is a standard GraphQL POST with three top-level keys:

```json
{
  "operationName": "Products",
  "variables": { ... },
  "query": "..."
}
```

- `operationName` — always `"Products"`. Never changes.
- `variables` — the only thing that changes between collection runs (pickup/destination coordinates).
- `query` — the full GraphQL query string. Static. Never changes.

---

## 2. Headers

Only the headers listed below are required. Everything else in the original cURL (sec-ch-ua, priority, accept-language, etc.) is browser noise and can be omitted.

| Header | Value | Notes                        |
|---|---|------------------------------|
| `content-type` | `application/json` | Required                     |
| `x-csrf-token` | `x` | Required. See note below.    |
| `x-uber-rv-session-type` | `desktop_session` | Required.                    |
 | `Cookie` | `sid`, `csid` | Required for auth |


### On `x-csrf-token: x`

This is intentional, not a bug. Uber uses the **Double Submit Cookie** pattern for CSRF protection. The server verifies that the header *exists*, not that it contains a specific value. The security guarantee comes from the fact that a third-party site cannot read Uber's cookies due to browser same-origin policy — so if a request arrives with both the session cookie and this header, it must have originated from legitimate JavaScript running on `m.uber.com`. Sending `x` as the value is a common convention for this pattern.

---

## 3. Authentication

Only two cookies are required. The full cookie string in the original cURL contains many tracking and analytics cookies that are irrelevant to authentication.

### `sid` (Session ID)
- **What it is:** Your Uber account session token. The primary authentication credential.
- **Format:** Uber proprietary, not a JWT. Opaque to the client.
- **Lifespan:** Weeks to months. Survives browser restarts. Invalidated on logout.
- **How to refresh:** Log into `m.uber.com`, open DevTools → Application → Cookies, copy the `sid` value.

### `csid` (Client Session ID)
- **What it is:** A secondary session identifier. Exact purpose unclear — likely ties the session to a specific browser client context.
- **Format:** Proprietary. Not a JWT.
- **Lifespan:** Unknown. Likely tied to `sid`.
- **How to refresh:** Same as `sid` — copy from DevTools after login.

### `jwt-session` (Not Required)
- Present in the original cURL but **not required** for fare estimation requests.
- A standard JWT. Expires in ~24 hours.
- Payload contains: `slate-expires-at`, `tenancy: uber/production`, standard `iat`/`exp` fields.
- Empty strings for `User-Agent`, `x-uber-client-id`, `x-uber-device` — suggests it is generated server-side with minimal client context.

### `__cf_bm` (Not Required for Now)
- Cloudflare Bot Management token. Generated from browser fingerprinting.
- Expires in ~30 minutes.
- Not required for fare estimation — Uber appears to apply lighter bot protection on this endpoint compared to booking endpoints.
- If requests start getting blocked, this is the first thing to investigate adding back.

---

## 4. Variables

The variables object is the only part of the request that changes between collection runs.

### Required Variables

```json
{
  "destinations": [
    {
      "latitude": 12.9778874,
      "longitude": 77.5925547
    }
  ],
  "pickup": {
    "latitude": 12.974812100016372,
    "longitude": 77.59086850001289
  },
  "payment": {
    "uberCashToggleOn": true
  }
}
```

| Variable | Type | Required | Notes |
|---|---|---|---|
| `pickup` | `InputCoordinate` | Yes | Origin coordinates |
| `destinations` | `[InputCoordinate!]!` | Yes | Array but always one element for standard rides |
| `payment.uberCashToggleOn` | Boolean | Yes | Must be `true` to get fare estimates. Setting to `false` returns no fares. |
| `includeRecommended` | Boolean | No | Always `false`. Omitting it is fine. |

### Unused Optional Variables

These appear in the query signature but are never populated by the web app for standard fare requests. Send them as `null` or omit entirely.

| Variable | Purpose |
|---|---|
| `boostedVehicleId` | Promotes a specific vehicle in results |
| `capacity` | Filter by passenger capacity |
| `isRiderCurrentUser` | Unknown |
| `paymentProfileUUID` | Business profile payments |
| `pickupFormattedTime` | Scheduled rides |
| `profileType` | Business/personal profile |
| `profileUUID` | Business profile UUID |
| `returnByFormattedTime` | Round trip scheduled rides |
| `stuntID` | Internal A/B testing |
| `targetProductType` | Filter by product type |

---

## 5. Response Structure

The response follows standard GraphQL structure:

```json
{
  "data": {
    "products": {
      "defaultVVID": "...",
      "tiers": [ ... ],
      "productsUnavailableMessage": "...",
      ...
    }
  }
}
```

### Top Level: `products`

| Field | Type | Notes |
|---|---|---|
| `defaultVVID` | String | The vehicle view ID Uber recommends by default |
| `tiers` | Array | Groups of products. Usually 2 tiers: recommended and economy |
| `productsUnavailableMessage` | String | Non-null when Uber is unavailable in the area |
| `hourlyTiersWithMinimumFare` | Array | Hourly ride packages. Not relevant. |
| `intercity` | Object | Intercity ride options. Not relevant. |
| `links` | Array | Promotional links. Not relevant. |

### Tier Structure

Products are grouped into tiers. Each tier has a `title` (e.g. "Rides we think you'll like", "Economy") and a `products` array. For data collection purposes, tiers are irrelevant — flatten all products from all tiers into individual rows.

```json
{
  "title": "Rides we think you'll like",
  "products": [ ... ]
}
```

### Product Structure

Each product in a tier represents one ride type (Uber Go AC, Go Non AC, UberXL, etc.). One product = one CSV row.

```json
{
  "displayName": "Uber Go AC",
  "productClassificationTypeName": "UBERGO",
  "currencyCode": "INR",
  "cityID": "130",
  "estimatedTripTime": 3008,
  "etaInMin": 5,
  "isAvailable": true,
  "fares": [ ... ],
  ...
}
```

---

## 6. Fields We Collect

### From the Product Object

| Response Field | CSV Column | Notes |
|---|---|---|
| `displayName` | `display_name` | Human-readable name e.g. "Uber Go AC" |
| `productClassificationTypeName` | `product_id` | Canonical type e.g. `UBERGO`, `UBERX`, `UBERXL`, `COMFORT`, `BLACK` |
| `currencyCode` | `currency_code` | Always `INR` for Bengaluru |
| `cityID` | — | Always `"130"` for Bengaluru. Use as a sanity check, don't store. |
| `estimatedTripTime` | `duration_seconds` | Trip duration in seconds |
| `etaInMin` | — | Driver ETA in minutes. Not in current schema but worth considering. |
| `isAvailable` | — | Skip the row if `false` |
| `productUuid` | `product_id` (alternative) | Stable UUID for the product type. More reliable than classification name for joins. |

### From `fares[0]`

Each product has a `fares` array. It always contains exactly one element for standard rides.

| Response Field | CSV Column | Notes |
|---|---|---|
| `fare` | — | Human-readable string e.g. `"₹662.04"`. Don't store — parse `fareAmountE5` instead. |
| `fareAmountE5` | `fare` | Fare in units of 1/100000 rupees. Divide by 100000 to get rupee value. e.g. `66204000 / 100000 = 662.04` |
| `capacity` | — | Max passengers. Not in current schema but potentially useful. |
| `hasPromo` | — | Whether a promo is applied. Worth storing if you want to filter promo-influenced fares. |
| `meta` | — | JSON string containing surge multiplier, distance, and origin/destination coordinates. **Requires a second parse step.** See Section 7. |

---

## 7. The Meta Field

The `meta` field inside each fare is a **JSON string embedded inside the JSON response**. It must be parsed separately.

```java
String metaJson = fare.getMeta();
JsonNode meta = objectMapper.readTree(metaJson);
```

### Key Fields Inside Meta

```json
{
  "upfrontFare": {
    "surgeMultiplier": 1.0,
    "fare": "662.04",
    "originLat": 12.974812100016372,
    "originLng": 77.59086850001289,
    "destinationLat": 13.1987889,
    "destinationLng": 77.7152899,
    "unmodifiedDistance": 34691,
    "estimatedDuration": null
  },
  "dynamicFareInfo": {
    "multiplier": 1.0,
    "isSobriety": false,
    "uuid": "..."
  },
  "fareSessionUUID": "...",
  "fareFlowUUID": "...",
  "pricingParams": { ... }
}
```

| Meta Field | CSV Column | Notes |
|---|---|---|
| `upfrontFare.surgeMultiplier` | `surge_multiplier` | 1.0 = no surge. >1.0 = active surge. |
| `upfrontFare.unmodifiedDistance` | `distance_metres` | Distance in metres. Divide by 1000 for km. |
| `dynamicFareInfo.multiplier` | — | Same as `surgeMultiplier`. Use either one. |
| `upfrontFare.fare` | — | Redundant with `fareAmountE5`. Ignore. |

### Why Meta Exists as a String

The `meta` field is likely passed through the GraphQL layer without being parsed by Uber's BFF (Backend for Frontend). It originates from a downstream pricing service and is forwarded as an opaque string to avoid the GraphQL schema needing to mirror the pricing service's internal schema. Common pattern in large microservice architectures.

---

## 8. Schema Mapping

Complete mapping from response fields to dataset CSV columns.

| CSV Column | Source | Transformation |
|---|---|---|
| `query_id` | Generated | UUID per collection run |
| `timestamp` | Generated | UTC ISO 8601 at time of request |
| `pickup_latitude` | `variables.pickup.latitude` | Pass through |
| `pickup_longitude` | `variables.pickup.longitude` | Pass through |
| `drop_latitude` | `variables.destinations[0].latitude` | Pass through |
| `drop_longitude` | `variables.destinations[0].longitude` | Pass through |
| `weather` | Weather API | Fetched separately |
| `temperature` | Weather API | Fetched separately |
| `hour` | Generated | Hour extracted from timestamp |
| `weekday` | Generated | 0=Monday, 6=Sunday |
| `weekend` | Generated | true if weekday >= 5 |
| `product_id` | `productClassificationTypeName` | e.g. UBERGO, UBERXL |
| `display_name` | `displayName` | e.g. "Uber Go AC" |
| `currency_code` | `currencyCode` | Always INR |
| `fare` | `fares[0].fareAmountE5` | Divide by 100000 |
| `surge_multiplier` | `meta → upfrontFare.surgeMultiplier` | Requires meta parse |
| `duration_seconds` | `estimatedTripTime` | Already in seconds |
| `distance_metres` | `meta → upfrontFare.unmodifiedDistance` | In metres |

> **Note:** The original spec uses `distance_miles`. Given this is Bengaluru data from an Indian Uber instance, the distance returned is in metres. Storing as `distance_metres` is more accurate. Update the spec accordingly.

---

## 9. Irrelevant Query Parts

These fragments are requested by the web app's generic query but return no data useful for fare collection. They are safe to remove from the query sent by the collector.

| Fragment | Why Irrelevant |
|---|---|
| `HourlyTierFragment` | Hourly rental packages, not point-to-point rides |
| `IntercityFragment` | Multi-city rides |
| `IntercityConfigFragment` | Intercity booking configuration |
| `IntercityTimePickerFragment` | Intercity time selection UI |
| `ProductLegalConsentFragment` | UI legal disclaimer text |
| `HourlyOverageRatesFragment` | Overage rates for hourly rentals |
| `BadgesFragment` | UI badges e.g. "Cheaper" label |

These fields on `ProductFragment` are also irrelevant:

| Field | Why Irrelevant |
|---|---|
| `badges` | UI decoration |
| `discountPrimary` | Promo display string |
| `etaMax` | Always null |
| `etaStringShort` | Always empty string |
| `hasBenefitsOnFare` | Always false for standard rides |
| `hasRidePass` | Ride pass subscription |
| `hourly` | Hourly rental |
| `iconType` | Always empty string |
| `is3p` | Third-party provider flag, always false in Bengaluru |
| `legalConsent` | Always null |
| `preAdjustmentValue` | Always empty string |
| `productImageUrl` | UI image URL |
| `rankedPricingExplainerText` | Always empty string |
| `reserveEnabled` | Scheduled ride flag |
| `parentProductUuid` | Internal grouping UUID |

---

## 10. Trimmed Query

A minimal query that requests only the fields the collector actually uses.

```graphql
query Products(
  $destinations: [InputCoordinate!]!,
  $pickup: InputCoordinate!,
  $payment: InputPayment,
  $includeRecommended: Boolean = false
) {
  products(
    destinations: $destinations
    pickup: $pickup
    payment: $payment
    includeRecommended: $includeRecommended
  ) {
    productsUnavailableMessage
    tiers {
      products {
        displayName
        productClassificationTypeName
        productUuid
        currencyCode
        cityID
        estimatedTripTime
        etaInMin
        isAvailable
        fares {
          fareAmountE5
          hasPromo
          meta
        }
      }
    }
  }
}
```

> **Caution:** The trimmed query has not been tested yet. Uber's server may reject queries that deviate significantly from what the web app sends, or may return unexpected null fields. Start with the full query for Milestone 1 and switch to the trimmed version once the pipeline is stable.

---

## 11. Known Unknowns

- **Will Uber reject the trimmed query?** Unknown. Their GraphQL server may validate against an expected query shape. Test during Milestone 1.
- **How long does `sid` actually last?** Empirically unknown. Uber's session management may invalidate it after extended inactivity.
- **Does `csid` need to match `sid`?** They were captured together from the same session. Unknown whether mismatched values cause auth failures.
- **Is `__cf_bm` actually not required?** Confirmed by testing with just `sid` + `csid`. May change if Uber tightens bot detection on this endpoint.
- **Will request rate trigger blocking?** Unknown threshold. Start conservatively at one request per 15-30 minutes and monitor telemetry for non-200 responses.
- **Does `payment.uberCashToggleOn` affect pricing?** It must be `true` to receive fares. Whether `false` returns different prices or simply no prices is untested.
- **Is `unmodifiedDistance` in metres?** Inferred from the value `34691` for a ~35km trip (Cubbon Park to Airport). Not explicitly documented.