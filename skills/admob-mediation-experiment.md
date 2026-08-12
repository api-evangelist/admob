---
name: Run a mediation A/B experiment (v1beta)
description: Use the AdMob v1beta API to inspect mediation groups, create an A/B experiment over a mediation waterfall, and stop it once a variant leads.
api: openapi/admob-api-v1beta-openapi.yml
operations: [accounts_list, accounts_adSources_list, accounts_adSources_adapters_list, accounts_mediationGroups_list, accounts_mediationGroups_patch, accounts_mediationGroups_mediationAbExperiments_create, accounts_mediationGroups_mediationAbExperiments_stop]
generated: '2026-08-12'
method: generated
source: openapi/admob-api-v1beta-openapi.yml, errors/admob-problem-types.yml, conventions/admob-conventions.yml
---

# Run a mediation A/B experiment (v1beta)

Base URL `https://admob.googleapis.com`, path prefix `/v1beta`. **This is a beta surface** — Google
may change it before promotion to `v1`. It is also where every *write* operation on the AdMob API
lives; `v1` is read-only.

Scope: `https://www.googleapis.com/auth/admob.readonly` for the reads, and the same user-consented
OAuth token for the writes (there is no separate write scope; service accounts are not supported).

## Steps

1. **Resolve the account.** `accounts_list` (`GET /v1beta/accounts`) → `account[0].name`.
2. **Inventory the mediation surface.**
   - `accounts_adSources_list` (`GET /v1beta/accounts/{accountsId}/adSources`) — the third-party
     networks available to this account.
   - `accounts_adSources_adapters_list`
     (`GET /v1beta/accounts/{accountsId}/adSources/{adSourcesId}/adapters`) — per ad source, the
     adapters and, critically, `adapterConfigMetadata`: the exact configuration keys that ad source
     requires. Read these before building any mediation line; they differ per network.
3. **Find the waterfall.** `accounts_mediationGroups_list`
   (`GET /v1beta/accounts/{accountsId}/mediationGroups`). Each `MediationGroup` carries
   `mediationGroupId`, `displayName`, `state`, `targeting` (format, platform, ad units, geo) and
   `mediationGroupLines`. Check `mediationAbExperimentState` — a group already running an experiment
   must not be given another.
4. **Create the experiment.** `accounts_mediationGroups_mediationAbExperiments_create`
   (`POST /v1beta/accounts/{accountsId}/mediationGroups/{mediationGroupsId}/mediationAbExperiments`)
   with a `MediationAbExperiment`: `displayName`, `treatmentMediationLines` (the variant waterfall),
   and `treatmentTrafficPercentage` (share of traffic sent to the treatment).
   `controlMediationLines` reflect the group's current lines.
5. **Read the result.** Re-read the group via `accounts_mediationGroups_list` and inspect the
   experiment's `state`, `startTime`/`endTime` and `variantLeader`. Attribute revenue with
   `accounts_mediationReport_generate`
   (`POST /v1beta/accounts/{accountsId}/mediationReport:generate`) over the experiment window.
6. **Stop it.** `accounts_mediationGroups_mediationAbExperiments_stop`
   (`POST /v1beta/accounts/{accountsId}/mediationGroups/{mediationGroupsId}/mediationAbExperiments:stop`).
   To adopt the winning waterfall permanently, apply it with `accounts_mediationGroups_patch`
   (`PATCH /v1beta/accounts/{accountsId}/mediationGroups/{mediationGroupsId}`) using an update mask.

## Rules — read these before writing

- **There is no idempotency contract.** The AdMob API documents no `Idempotency-Key` header and the
  spec declares no equivalent parameter. A retried `create` on a network timeout can produce a
  second experiment or a duplicate mediation group. Before retrying any write, re-read the parent
  collection and confirm the resource was not already created.
- **There is no sandbox.** Every call here mutates the real, revenue-serving AdMob account.
  `sandbox/admob-sandbox.yml` covers demo ad units for the mobile SDK only — they do not apply to
  this API. Treat step 4 and step 6 as production changes and require human confirmation.
- **Stopping an experiment is consequential and irreversible in place.** Confirm intent before
  calling `:stop`; there is no resume.
- Errors are `google.rpc.Status` (`error.code` / `error.message` / `error.status`), not RFC 9457.
  `400 INVALID_ARGUMENT` on a create is almost always a mediation line whose config keys do not match
  that adapter's `adapterConfigMetadata` from step 2.
- Inventory-class reads (steps 2 and 3) share the 120 requests/minute, 172,800/day per-project quota.
