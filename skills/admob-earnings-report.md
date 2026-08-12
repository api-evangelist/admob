---
name: Pull an AdMob earnings report
description: Authorize against the AdMob API and generate a network report of estimated earnings, impressions and eCPM broken down by date, app or country.
api: openapi/admob-api-v1-openapi.yml
operations: [accounts_list, accounts_get, accounts_networkReport_generate]
generated: '2026-08-12'
method: generated
source: openapi/admob-api-v1-openapi.yml, conventions/admob-conventions.yml, rate-limits/admob-rate-limits.yml
---

# Pull an AdMob earnings report

Base URL `https://admob.googleapis.com`. Every call needs a Google OAuth 2.0 **user** access token
in `Authorization: Bearer <token>`. Service accounts are **not** supported — a human must consent.

## Before you start

- Scope required: `https://www.googleapis.com/auth/admob.report` (the `admob.readonly` scope alone
  does not cover report generation).
- Enable `admob.googleapis.com` on the Google Cloud project that owns the OAuth client.
- Reporting quota is **900 read requests per minute per project**. See
  `rate-limits/admob-rate-limits.yml`.

## Steps

1. **Resolve the publisher account.** Call `accounts_list` (`GET /v1/accounts`). It returns the AdMob
   publisher account most recently signed in to from the AdMob UI. Read `account[0].name` — the
   hierarchical resource name `accounts/pub-XXXXXXXXXXXXXXXX`. Read `currencyCode` and
   `reportingTimeZone` too: every monetary figure and every date boundary in the report is expressed
   in those, not in your locale.
   - If you already know the publisher id, call `accounts_get`
     (`GET /v1/accounts/{accountsId}`) instead and skip the list.
2. **Build the report spec.** Call `accounts_networkReport_generate`
   (`POST /v1/accounts/{accountsId}/networkReport:generate`) with a `reportSpec`:
   - `dateRange` — `{startDate: {year, month, day}, endDate: {year, month, day}}`. These are
     `Date` objects with separate integer fields, not ISO strings.
   - `metrics` — pick from `ESTIMATED_EARNINGS`, `IMPRESSIONS`, `IMPRESSION_RPM`, `CLICKS`,
     `IMPRESSION_CTR`, `AD_REQUESTS`, `MATCHED_REQUESTS`, `MATCH_RATE`, `SHOW_RATE`.
   - `dimensions` — pick from `DATE`, `WEEK`, `MONTH`, `APP`, `AD_UNIT`, `COUNTRY`, `FORMAT`,
     `PLATFORM`, `AD_TYPE`, `MOBILE_OS_VERSION`, `GMA_SDK_VERSION`, `APP_VERSION_NAME`,
     `SERVING_RESTRICTION`.
   - Optional: `dimensionFilters`, `sortConditions`, `maxReportRows`, `localizationSettings`,
     `timeZone`.
3. **Read the streamed response.** The response is a sequence of `GenerateNetworkReportResponse`
   objects, not a single array: one `header` (echoing the date range, localization settings and
   reporting time zone), then one object per `row`, then one `footer`. Accumulate the `row` values;
   always check `footer.warnings` — AdMob signals partial or delayed data there rather than failing
   the call.
4. **For mediation revenue**, run the same shape against `accounts_mediationReport_generate`
   (`POST /v1/accounts/{accountsId}/mediationReport:generate`), which adds ad-source dimensions.

## Rules

- **Do not retry a 429.** A `429` carries `error.status: RESOURCE_EXHAUSTED`. AdMob publishes no
  `Retry-After` header. The documented fix is to narrow the request — shorter date range, fewer
  dimensions — and to rate-limit client side.
- **Errors are `google.rpc.Status`, not RFC 9457.** Read `error.code`, `error.message`,
  `error.status`. See `errors/admob-problem-types.yml`.
- **`403 PERMISSION_DENIED`** almost always means the token was minted with `admob.readonly` only,
  or the AdMob API is not enabled on the project — not that the account lacks data.
- Reports are generated live; there is no report id to poll and nothing to cache server side.
- There is **no sandbox** for this API. Calls read the real account. See `sandbox/admob-sandbox.yml`.
