---
name: Audit an AdMob app and ad-unit inventory
description: Walk an AdMob publisher account's apps and ad units to produce an inventory audit — which apps are approved, which ad units exist per format, and which have mediation enabled.
api: openapi/admob-api-v1-openapi.yml
operations: [accounts_list, accounts_apps_list, accounts_adUnits_list]
generated: '2026-08-12'
method: generated
source: openapi/admob-api-v1-openapi.yml, data-model/admob-data-model.yml, rate-limits/admob-rate-limits.yml
---

# Audit an AdMob app and ad-unit inventory

Base URL `https://admob.googleapis.com`. OAuth 2.0 user access token required;
`https://www.googleapis.com/auth/admob.readonly` is sufficient for this whole flow.

## Steps

1. **Resolve the account.** `accounts_list` (`GET /v1/accounts`) → `account[0].name`, of the form
   `accounts/pub-XXXXXXXXXXXXXXXX`.
2. **List apps.** `accounts_apps_list` (`GET /v1/accounts/{accountsId}/apps`). Page with
   `pageSize` + `pageToken`; keep calling while `nextPageToken` is non-empty. For each `App` record:
   - `appId` — the AdMob app id, the join key to ad units.
   - `platform` — IOS or ANDROID.
   - `appApprovalState` — flag anything not approved; unapproved apps do not serve.
   - `linkedAppInfo` vs `manualAppInfo` — linked apps are matched to a store listing, manual apps
     are not. A manual app that has been live for a long time is usually an integration gap.
3. **List ad units.** `accounts_adUnits_list` (`GET /v1/accounts/{accountsId}/adUnits`), paged the
   same way. For each `AdUnit`:
   - `appId` — group ad units under the app from step 2.
   - `adFormat` and `adTypes` — the format mix per app.
   - `rewardSettings` — present only on rewarded units.
   - `adUnitId` — the public `ca-app-pub-…/…` id the mobile SDK requests.
4. **Report the gaps.** Apps with zero ad units; approved apps missing a high-value format
   (app open, rewarded, native); ad units with no matching app record.

## Paging and limits

- **Inventory quota is the tight one: 120 read requests per minute and 172,800 per day, per
  project** — far tighter than the 900/min for account and reporting calls. Both list calls in this
  skill count against it. Use the largest `pageSize` the API accepts rather than many small pages.
- Paging is `pageSize` / `pageToken` in, `nextPageToken` out. Stop when `nextPageToken` is absent.

## Rules

- Ad units are addressed by hierarchical resource name
  (`accounts/{publisherId}/adUnits/{adUnitId}`), not by a bare id. Do not construct these by hand —
  use the `name` returned by the list call.
- `v1` is **read-only**. Creating apps and ad units
  (`accounts_apps_create`, `accounts_adUnits_create`) exists only on `v1beta`
  (`openapi/admob-api-v1beta-openapi.yml`), whose surface may change before promotion.
- On `429 RESOURCE_EXHAUSTED`, back off client side; there is no `Retry-After` header.
