---
name: involve-asia-generate-deeplinks
description: Generate trackable Involve Asia affiliate deeplinks for a batch of destination URLs, tagging each with a campaign sub-ID.
api: openapi/involve-asia-publisher-openapi.yml
operations: [authenticate, offersAll, deeplink]
---

# Generate Involve Asia affiliate deeplinks

Turn a list of destination product URLs into trackable affiliate links.

## Steps

1. **Authenticate.** `POST /authenticate` with form fields `key` and `secret`.
   Read `data.token`; cache it ~110 minutes (2h TTL). Send
   `Authorization: Bearer <token>` on every later call.
2. **Resolve the offer.** `POST /offers/all` (optionally
   `filters[offer_name]` or `filters[offer_country]`) to find the `offer_id`
   whose whitelisted domain matches your destination URL. The destination URL
   MUST belong to one of the offer's whitelisted domains.
3. **Generate each link.** For every URL, `POST /deeplink/generate` with
   `offer_id`, `url`, and `aff_sub=<your-slug>` (sub-ID #1 is `aff_sub` with
   NO number; use `aff_sub2`..`aff_sub5` for the rest). Read
   `data.tracking_link` from each response.

## Rules

- **Resolve offer_id first.** An unknown `offer_id` or a non-whitelisted `url`
  returns HTTP 500 "Something went wrong" — a PERMANENT client error; fix the
  input, do not retry (errors/involve-asia-problem-types.yml).
- **Throttle:** 60 req/min/account. Back off 250 -> 500 -> 1000 ms on 429.
- **Deeplink cap:** 1,000 unique tracking links per rolling 30-day window per
  account.
- Treat the returned `tracking_link` host (e.g. `invl.me`) as opaque.
