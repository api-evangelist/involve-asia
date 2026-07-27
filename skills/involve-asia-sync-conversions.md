---
name: involve-asia-sync-conversions
description: Incrementally sync Involve Asia conversion (commission) data for a date range and roll it up by sub-ID or offer.
api: openapi/involve-asia-publisher-openapi.yml
operations: [authenticate, conversionsRange, conversionsDataRange]
---

# Sync Involve Asia conversions

Pull attributed conversions for reporting and reconciliation.

## Steps

1. **Authenticate.** `POST /authenticate` with `key` + `secret`; cache
   `data.token` ~110 min; send `Authorization: Bearer <token>`.
2. **Pull the window.** `POST /conversions/range` with `start_date` and
   `end_date` (`YYYY-MM-DD`). For sub-day / hourly overlapping syncs use
   `POST /conversions/data-range` with `YYYY-MM-DD HH:MM:SS` timestamps.
   Optionally filter `filters[conversion_status]=pending|approved|rejected`
   and `filters[preferred_currency]=USD|MYR`.
3. **Paginate.** The envelope is
   `{ status, message, data: { page, limit, count, nextPage, data: [...] } }`.
   Continue while `page * limit < count` (or until `nextPage` is null);
   `limit` max 100.
4. **Roll up.** Group rows by `offer_name` or by `aff_sub1`..`aff_sub5`; sum
   `payout` (final commission). `base_payout` + `bonus_payout` are reported
   independently and do NOT necessarily sum to `payout`.

## Rules

- **Data lag ~24h.** Exclude today from rollups unless you want the partial day.
- Conversions carry no country field — scope a market via `currency` or an
  offer-name substring.
- **Throttle:** 60 req/min/account; back off 250 -> 500 -> 1000 ms on 429.
- A 401 may be a bad token (re-auth once, retry) OR a validation error
  (missing `start_date`/`end_date` — fix the request, do not retry).
