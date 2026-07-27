---
name: involve-asia-publisher-api
version: 1.0.0
description: Use this skill when working with the Involve Asia Publisher API — pull conversion data, list offers, generate trackable affiliate deeplinks, or analyse commission performance. Covers JWT auth, the 9-endpoint catalogue, pagination, rate limits, and the (non-standard) error shapes the API actually returns.
---

# Involve Asia Publisher API

Help publishers pull conversion data, list offers, generate trackable
deeplinks, and analyze commission performance via natural language.

## Setup

Set these environment variables before calling the API:

- `IA_API_KEY` — your API key (string).
- `IA_API_SECRET` — your API secret (string).

Request and manage keys at **https://app.involve.asia/v2/publisher/api-keys**
(Tools → API once logged in).
If you need higher limits, contact your Involve Asia account manager.

## Authentication recipe

1. `POST https://api.involve.asia/api/authenticate` with `key` + `secret` (form-urlencoded).
2. Read `data.token` from the JSON response.
3. Cache the token for **~110 minutes** (proactive refresh under the 2h TTL).
4. Send `Authorization: Bearer <token>` on every other call.
5. On `401 Unauthorized`, re-authenticate **once** and retry the original request.

```bash
curl -X POST "https://api.involve.asia/api/authenticate" \
  -H "Accept: application/json" \
  --data-urlencode "key=$IA_API_KEY" \
  --data-urlencode "secret=$IA_API_SECRET"
```

Use `--data-urlencode` (not bare `-d`) so secrets containing `+`, `/`,
or `=` are not mangled by curl's default URL-decode pass.

## Common queries

### "How much did I earn from Shopee MY last month, broken out by sub ID?"

- `POST /conversions/range` with `start_date` = first day of last month,
  `end_date` = last day of last month.
- Filter rows where the offer name matches Shopee MY (`offer_name` contains
  `Shopee` AND `currency` is `MYR` — conversions have no `country` field,
  so use `currency` or the offer-name substring to scope to Malaysia),
  then group by `aff_sub1`..`aff_sub5`.
- Sum `payout` per sub-ID bucket. Exclude today from rollups (24h data lag).

### "Generate 50 deeplinks for these product URLs and tag them with my newsletter slug."

- For each URL, `POST /deeplink/generate` with `offer_id`, `url`, and
  `aff_sub=<newsletter-slug>` (note: sub-ID #1 is sent as `aff_sub`, not
  `aff_sub1` — it surfaces on the conversion record as `aff_sub1`).
- Read `data.tracking_link` from each response.
- Respect the 60 req/min throttle (back off 250 → 500 → 1000 ms on 429).
- Remember the rolling 30-day cap: 1,000 unique tracking links per account.

### "Compare my EPC across Lazada, Shopee, and Tokopedia this quarter."

- `POST /conversions/range` with the quarter's start and end dates.
- Group conversions by offer (Lazada / Shopee / Tokopedia).
- EPC = (sum of `payout` for `conversion_status=approved`) / (clicks for the
  same offer; clicks come from your reporting source — not exposed via the
  Publisher API).

### "Which of my pending conversions are likely to be rejected?"

- `POST /conversions/range` filtered to `conversion_status=pending`.
- Inspect each row's offer, currency, and timestamp. Historical rejection
  rates per offer (computed from prior `conversion_status=rejected` data
  via the same endpoint) are the strongest signal.
- Webhook reconciliation: re-query `/conversions/range` over the last 7
  days to detect status flips from pending to rejected.

## Authenticate

`POST https://api.involve.asia/api/authenticate`

Exchange your API key and secret for a bearer token. Tokens expire after 2 hours — cache them and refresh proactively (or on 401).

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `key` | string | yes | Your API key. Manage at https://app.involve.asia/v2/publisher/api-keys (Tools → API). |
| `secret` | string | yes | Your API secret. Treat like a password. |

### Example

```bash
curl -X POST "https://api.involve.asia/api/authenticate" \
  -H "Accept: application/json" \
  --data-urlencode "key=general" \
  --data-urlencode "secret=general_secret_key"
```

### Response

```json
{
  "status": "success",
  "message": "Success",
  "data": {
    "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
  }
}
```

## All conversions

`POST https://api.involve.asia/api/conversions/all`

Paginated dump of every conversion attributed to your account. Useful for cold syncs; for incrementals use /conversions/range or /conversions/data-range.

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `Authorization` | header | yes | Bearer {token} |
| `page` | integer | no | Page number (default 1). |
| `limit` | integer | no | Page size, default 100, max 100. |
| `filters[conversion_id]` | string | no | Pipe-separated list, e.g. `899191|7008168`. |
| `filters[offer_id]` | integer | no | Restrict to a single offer. |
| `filters[offer_name]` | string | no | Substring match on offer name. |
| `filters[conversion_status]` | string | no | Pipe-separated: `pending|approved|rejected`. |
| `filters[preferred_currency]` | string | no | ISO 4217 (`USD` or `MYR`). Defaults to the conversion's local currency. |

### Example

```bash
curl -X POST "https://api.involve.asia/api/conversions/all" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "page=1" \
  --data-urlencode "limit=100" \
  --data-urlencode "filters[conversion_status]=pending|approved"
```

### Response

```json
{
  "status": "success",
  "message": "Success",
  "data": {
    "page": 1,
    "limit": 100,
    "count": 246,
    "nextPage": 2,
    "data": [
      {
        "conversion_id": 899191,
        "offer_id": 44,
        "offer_name": "Lazada (PH)",
        "sale_amount": "345.00",
        "currency": "PHP",
        "payout": "14.49",
        "base_payout": "14.49",
        "bonus_payout": "0.00",
        "aff_sub1": null,
        "aff_sub2": null,
        "aff_sub3": null,
        "aff_sub4": null,
        "aff_sub5": null,
        "adv_sub1": "348666368",
        "adv_sub2": "Health & Beauty",
        "adv_sub3": "false",
        "adv_sub4": "AD109HBAK1YRANPH-611385",
        "adv_sub5": "6.00",
        "datetime_conversion": "2017-02-22 21:16:47",
        "conversion_status": "pending",
        "affiliate_remarks": "rejected by advertiser"
      }
    ]
  }
}
```

## Conversions by range

`POST https://api.involve.asia/api/conversions/range`

Pull conversions whose conversion datetime falls within a date range. Exclude today unless it's the 1st of the month — partial data trickles in for ~24 hours after the click.

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `Authorization` | header | yes | Bearer {token} |
| `start_date` | string | yes | Inclusive start, `YYYY-MM-DD`. |
| `end_date` | string | yes | Inclusive end, `YYYY-MM-DD`. |
| `page` | integer | no | Page number (default 1). |
| `limit` | integer | no | Page size, default 100, max 100. |
| `filters[conversion_id]` | string | no | Pipe-separated list of conversion IDs. |
| `filters[offer_id]` | integer | no | Restrict to a single offer. |
| `filters[conversion_status]` | string | no | Pipe-separated: `pending|approved|rejected`. |
| `filters[preferred_currency]` | string | no | `USD` or `MYR`. Defaults to local currency. |

### Example

```bash
curl -X POST "https://api.involve.asia/api/conversions/range" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "start_date=2026-05-01" \
  --data-urlencode "end_date=2026-05-22" \
  --data-urlencode "filters[preferred_currency]=USD"
```

### Response

```json
{
  "status": "success",
  "message": "Success",
  "data": {
    "page": 1,
    "limit": 100,
    "count": 312,
    "nextPage": 2,
    "data": [
      {
        "conversion_id": 899192,
        "offer_id": 5126,
        "offer_name": "Shopee MY - CPS",
        "sale_amount": "89.90",
        "currency": "MYR",
        "payout": "5.26",
        "base_payout": "5.26",
        "bonus_payout": "0.00",
        "aff_sub1": "blog-post-42",
        "aff_sub2": null,
        "aff_sub3": null,
        "aff_sub4": null,
        "aff_sub5": null,
        "datetime_conversion": "2026-05-21 14:22:11",
        "conversion_status": "pending",
        "affiliate_remarks": null
      }
    ]
  }
}
```

## Conversions by date+time range

`POST https://api.involve.asia/api/conversions/data-range`

Same shape as /conversions/range, but accepts second-precision timestamps. Use this when you need sub-day windows (e.g. an hourly sync that overlaps).

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `Authorization` | header | yes | Bearer {token} |
| `start_date` | string | yes | Inclusive start, `YYYY-MM-DD HH:MM:SS`. |
| `end_date` | string | yes | Inclusive end, `YYYY-MM-DD HH:MM:SS`. |
| `page` | integer | no | Page number (default 1). |
| `limit` | integer | no | Page size, default 100, max 100. |
| `filters[conversion_id]` | string | no | Pipe-separated list of conversion IDs. |
| `filters[offer_id]` | integer | no | Restrict to a single offer. |
| `filters[conversion_status]` | string | no | Pipe-separated statuses. |
| `filters[preferred_currency]` | string | no | `USD` or `MYR`. |

### Example

```bash
curl -X POST "https://api.involve.asia/api/conversions/data-range" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "start_date=2026-05-22 13:00:00" \
  --data-urlencode "end_date=2026-05-22 14:30:00"
```

### Response

```json
{
  "status": "success",
  "message": "Success",
  "data": {
    "page": 1,
    "limit": 100,
    "count": 18,
    "nextPage": null,
    "data": [
      {
        "conversion_id": 7008168,
        "offer_id": 25,
        "offer_name": "Lazada (MY)",
        "sale_amount": "120.50",
        "currency": "MYR",
        "payout": "7.23",
        "datetime_conversion": "2026-05-22 14:20:19",
        "conversion_status": "approved"
      }
    ]
  }
}
```

## All offers

`POST https://api.involve.asia/api/offers/all`

Paginated list of offers you have access to. Filter by country, category, application status, and offer status.

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `Authorization` | header | yes | Bearer {token} |
| `page` | integer | no | Page number (default 1). |
| `limit` | integer | no | Page size, default 100, max 100. |
| `sort_by` | string | no | `relevance` (default) · `show_latest_first` · `show_oldest_first` · `a_to_z` · `z_to_a` · `highest_commision_percent` · `lowest_commision_percent` · `highest_flat_payout` · `lowest_flat_payout`. |
| `filters[offer_id]` | string | no | Pipe-separated list, e.g. `25|5126`. |
| `filters[offer_name]` | string | no | Substring match on offer name. |
| `filters[offer_country]` | string | no | Pipe-separated country names (English), e.g. `Malaysia|Indonesia`. |
| `filters[offer_type]` | string | no | Pipe-separated payout types: `cpa|cps|cpa_both|cpc|cpm`. |
| `filters[categories]` | string | no | Pipe-separated, e.g. `Electronics|Fashion|Finance|Health & Beauty|Lifestyle|Marketplace|Other|Services|Travel`. |
| `filters[application_status]` | string | no | Pipe-separated: `Approved|Blocked|Pending|Rejected`. Added 2024-10-01. |
| `filters[offer_status]` | string | no | Pipe-separated: `Active|Paused`. Added 2024-10-01. |

### Example

```bash
curl -X POST "https://api.involve.asia/api/offers/all" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "page=1" \
  --data-urlencode "limit=100" \
  --data-urlencode "filters[offer_country]=Malaysia" \
  --data-urlencode "filters[application_status]=Approved" \
  --data-urlencode "filters[offer_status]=Active"
```

### Response

```json
{
  "status": "success",
  "message": "Success",
  "data": {
    "page": 1,
    "limit": 100,
    "count": 75,
    "nextPage": 2,
    "data": [
      {
        "offer_id": 289,
        "offer_name": "Medical Departures",
        "description": "Offer Description",
        "preview_url": "https://www.medicaldepartures.com/",
        "currency": "USD",
        "logo": "https://img.involve.asia/ia_logo/289.png",
        "lookup_value": "cpa",
        "datetime_created": "2019-04-12 09:00:00",
        "datetime_updated": "2019-05-31 11:17:36",
        "countries": "International",
        "categories": "Health & Beauty",
        "commissions": [
          {
            "Commission": "USD52.50"
          }
        ],
        "special_commissions": [],
        "validation_terms": "30",
        "payment_terms": "60",
        "is_require_approval": "0",
        "commission_tracking": "manual",
        "tracking_type": "redirect",
        "directory_page": "https://app.involve.asia/offer/289",
        "marketplace_store_offer": "0",
        "tracking_link": "https://invl.me/aff_m?offer_id=577&aff_id=13972&source=ia_api_offer"
      }
    ]
  }
}
```

## Last-updated offers

`POST https://api.involve.asia/api/offers/last-updated-range`

Pull only offers whose `datetime_updated` falls within a window. Cheaper than /offers/all for delta sync.

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `Authorization` | header | yes | Bearer {token} |
| `start_date` | string | yes | Inclusive start, `YYYY-MM-DD HH:MM:SS`. |
| `end_date` | string | yes | Inclusive end, `YYYY-MM-DD HH:MM:SS`. |
| `page` | integer | no | Page number (default 1). |
| `limit` | integer | no | Page size, default 100, max 100. |
| `sort_by` | string | no | Same options as /offers/all. |
| `filters[offer_country]` | string | no | Pipe-separated country names. |
| `filters[categories]` | string | no | Pipe-separated categories. |
| `filters[application_status]` | string | no | Pipe-separated: `Approved|Blocked|Pending|Rejected`. |
| `filters[offer_status]` | string | no | Pipe-separated: `Active|Paused`. |

### Example

```bash
curl -X POST "https://api.involve.asia/api/offers/last-updated-range" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "start_date=2026-05-20 00:00:00" \
  --data-urlencode "end_date=2026-05-22 23:59:59"
```

### Response

```json
{
  "status": "success",
  "message": "Success",
  "data": {
    "page": 1,
    "limit": 100,
    "count": 12,
    "nextPage": null,
    "data": [
      {
        "offer_id": 5126,
        "offer_name": "Shopee MY - CPS",
        "datetime_updated": "2026-05-21 09:15:42",
        "countries": "Malaysia",
        "categories": "Marketplace",
        "currency": "MYR",
        "commissions": [
          {
            "New buyer": "6.50%"
          },
          {
            "Existing buyer": "1.20%"
          }
        ]
      }
    ]
  }
}
```

## List campaigns

`POST https://api.involve.asia/api/campaigns/all`

Campaign banners + landing pages your account is allowed to promote. Useful for surfacing seasonal vouchers and pre-built creatives.

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `Authorization` | header | yes | Bearer {token} |
| `page` | integer | no | Page number (default 1). |
| `limit` | integer | no | Page size, default 100, max 100. |
| `filters[campaign_banner_id]` | string | no | Pipe-separated campaign banner IDs. |
| `filters[offer_id]` | string | no | Pipe-separated offer IDs. |
| `filters[offer_name]` | string | no | Substring match on offer name. |
| `filters[start_date]` | string | no | `YYYY-MM-DD` — campaign-start window lower bound. |
| `filters[end_date]` | string | no | `YYYY-MM-DD` — campaign-end window upper bound. |
| `filters[country]` | string | no | Pipe-separated country names (English). |
| `filters[category]` | string | no | Pipe-separated category names. |
| `filters[coupons_only]` | boolean | no | `true` to limit to campaigns with a `voucher_code`. |
| `filters[with_banner]` | boolean | no | `true` to limit to campaigns that include a banner image. |
| `filters[banner_size]` | string | no | Pipe-separated: `300x250|728x90`. |
| `filters[commission_tracking]` | string | no | `manual` or `real-time`. |
| `filters[device_type]` | string | no | Pipe-separated: `desktop|mobile|ios|android`. |

### Example

```bash
curl -X POST "https://api.involve.asia/api/campaigns/all" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "filters[country]=Malaysia" \
  --data-urlencode "filters[coupons_only]=true"
```

### Response

```json
{
  "status": "success",
  "message": "Success",
  "data": {
    "page": 1,
    "limit": 100,
    "count": 71,
    "nextPage": 2,
    "data": [
      {
        "campaign_banner_id": 123,
        "offer_id": 46,
        "merchant_id": 577,
        "offer_name": "Photobook (MY)",
        "campaign_name": "60% OFF storewide!",
        "description": "60% OFF storewide! for all Photobooks",
        "voucher_code": "IARAM60",
        "date_campaign_start": "2026-05-23",
        "date_campaign_end": "2026-06-30",
        "banner_image_url": "https://img.involve.asia/rpss/campaigns_banners/photobook-may26.jpg",
        "tracking_link": "https://invl.me/aff_m?offer_id=577&aff_id=13972&source=ia_api_campaign&url=https%3A%2F%2Fwww.photobook.com.my%2Fprp%2Fmay26-aff-involveasia",
        "categories": "Lifestyle",
        "with_banner": true
      }
    ]
  }
}
```

## Generate deeplink

`POST https://api.involve.asia/api/deeplink/generate`

Convert a destination URL into a trackable affiliate link. Capped at 1,000 unique links per rolling 30-day window per account.

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `Authorization` | header | yes | Bearer {token} |
| `offer_id` | integer | yes | Offer ID from /offers/all. **Resolve first** — an unknown offer_id returns HTTP 500 with a generic 'Something went wrong' message (see Error model). |
| `url` | string | yes | Destination URL — must belong to one of the offer's whitelisted domains. A non-whitelisted host returns the same HTTP 500. |
| `aff_sub` | string | no | Sub ID 1 — note the param name has no number. Surfaces on the conversion as `aff_sub1`. Up to 255 chars. |
| `aff_sub2` | string | no | Sub ID 2. |
| `aff_sub3` | string | no | Sub ID 3. |
| `aff_sub4` | string | no | Sub ID 4. |
| `aff_sub5` | string | no | Sub ID 5. |

### Example

```bash
curl -X POST "https://api.involve.asia/api/deeplink/generate" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "offer_id=5126" \
  --data-urlencode "url=https://shopee.com.my/Apple-iPhone-16-Pro-i.12345678.987654" \
  --data-urlencode "aff_sub=blog-42" \
  --data-urlencode "aff_sub2=newsletter-q3"
```

### Response

```json
{
  "status": "success",
  "message": "Success",
  "data": {
    "offer_name": "Shopee MY - CPS",
    "offer_id": 5126,
    "merchant_id": 103972,
    "tracking_link": "https://invl.me/cl8a2bX9q"
  }
}
```

## Shopee Xtra brands

`POST https://api.involve.asia/api/shopeextra/all`

Boosted-payout brands enrolled in Shopee Commission Xtra. Refresh nightly. Page size cap is 200 (higher than other endpoints).

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `Authorization` | header | yes | Bearer {token} |
| `page` | integer | no | Page number (default 1). |
| `limit` | integer | no | Page size, default 200, **max 200**. |
| `filters[country]` | string | no | One of `Malaysia`, `Singapore`, `Indonesia`, `Thailand`, `Vietnam`, `Philippines`, `Taiwan`. |
| `filters[shop_name]` | string | no | Substring match on shop name (or a Shopee shop URL — the shop_id is parsed out). |
| `filters[shop_type]` | string | no | `mall` or `preferred`. |
| `filters[sort_type]` | string | no | `latest_updated`, `high_commission`, or default (shop_name ASC). |

### Example

```bash
curl -X POST "https://api.involve.asia/api/shopeextra/all" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "limit=200" \
  --data-urlencode "filters[country]=Malaysia" \
  --data-urlencode "filters[shop_type]=mall"
```

### Response

```json
{
  "status": "success",
  "message": "Success",
  "data": {
    "page": 1,
    "limit": 200,
    "count": 312,
    "nextPage": 2,
    "data": [
      {
        "shop_id": 1234567,
        "shop_name": "Apple Authorised Reseller",
        "shop_type": "mall",
        "shop_link": "https://shopee.com.my/apple_my",
        "shop_image": "https://cf.shopee.com.my/file/apple_logo.jpg",
        "shop_banner": [],
        "offer_name": "Shopee Malaysia",
        "country": "Malaysia",
        "period_start_time": "2026-05-01",
        "period_end_time": "2026-06-30",
        "commission_rate": "0.0150",
        "tracking_link": "https://invl.me/aff_m?offer_id=577&aff_id=13972&source=ia_api_shopeextra&url=https%3A%2F%2Fshopee.com.my%2Fapple_my"
      }
    ]
  }
}
```

## Gotchas

- **Data lag.** Conversion data lags ~24h. Exclude today from rollups
  unless you specifically need the most recent partial day.
- **Pagination.** Continue while `page * limit < count`. Default `limit`
  is 100 (Shopee Xtra: 200). Envelope shape:
  `{ status, message, data: { page, limit, count, nextPage, data: [...] } }`.
- **Throttle.** 60 requests / minute / account (NOT 20/min).
  Back off exponentially on 429: 250 → 500 → 1000 ms.
- **Deeplink cap.** `/deeplink/generate` is additionally capped at 1,000
  unique tracking links per rolling 30-day window per account.
- **Sub-IDs.** Publisher path supports five sub-IDs only. Request-param
  naming is asymmetric: sub-ID #1 is `aff_sub` (no number); sub-IDs #2–5
  are `aff_sub2`..`aff_sub5`. All five surface on the conversion record as
  `aff_sub1`..`aff_sub5`. Do not pass `aff_sub6` through `aff_sub10` —
  they are silently ignored.
- **Envelope.** Every endpoint returns `{ status, message, data }`. For
  paginated endpoints, `data` wraps a `{ page, limit, count, nextPage,
  data: [...] }` object — note the nested `data` key.
- **Envelope exceptions.** JWT auth errors do NOT follow the standard
  envelope: they may surface as `{"message":"...","status_code":401}` or
  `{"error":"Unauthenticated."}`. Handle both shapes defensively.
- **Payout independence.** `payout`, `base_payout`, and `bonus_payout`
  are independent fields. `payout` is the final commission amount;
  `base_payout` + `bonus_payout` are reported separately from a
  bonus-details join and do NOT necessarily sum to `payout`.
- **Deeplink 500 = permanent.** `/deeplink/generate` returns HTTP 500
  with "Something went wrong" for bad `offer_id` or non-whitelisted URL.
  Despite the 5xx, this is a permanent client error — do not retry.
- **Tracking link host varies.** The returned `tracking_link` host (e.g.
  `invl.me`) depends on the offer's tracking-domain configuration.
  Treat the full URL as opaque; do not hard-code the host.

## Error model

Error bodies use a flat `message` string with `data: []` (array, NOT an
object with field-level `errors`). The HTTP status is not always
semantically correct — inspect `message` AND status together.

- **401 (auth)** — bad / expired token. Response may NOT follow the
  standard envelope (e.g. `{"message":"Wrong number of segments",`
  `"status_code":401}` from the JWT layer). Re-authenticate **once**,
  then retry the original request.
- **401 (credentials)** on `/authenticate` — wrong key/secret.
  `{"status":"error","message":"Invalid Credentials","data":[]}`.
  Do NOT retry; the credentials are invalid.
- **401 (validation)** — missing required params (e.g. `/conversions/`
  `range` without `start_date`). Returned with status 401 today, not 422.
  `{"status":"error","message":"Incorrect arguments...","data":[]}`.
  **Fix the request — do NOT re-authenticate or retry unchanged.**
- **429 Too Many Requests** — genuine throttle. Back off using the
  ladder 250 → 500 → 1000 ms. On `/deeplink/generate`, the body
  message also covers the rolling 30-day cap.
- **500 "Something went wrong"** on `/deeplink/generate` — returned for
  invalid `offer_id`, non-whitelisted destination URLs, and other
  business-rule violations. **PERMANENT client error despite the 5xx**
  **status code — fix the input, do NOT retry.**
- **5xx (other)** — only retry an unexpected 5xx if the identical
  request previously succeeded (true transient).

Field-level error objects (`data.errors`) are NOT emitted by any
endpoint today. Parse `message` directly.
