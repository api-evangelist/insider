---
name: insider-ingest-user-data
description: Send user profiles, attributes and behavioural events into the Insider One Unified Customer Database, and read them back.
api: Insider One Unification API
spec: openapi/insider-unification-openapi.yml
base_url: https://unification.useinsider.com
operations:
  - upsertUserData
  - getUserProfiles
  - deleteUserAttribute
  - exportRawUserData
generated: '2026-08-13'
method: generated
source: openapi/insider-unification-openapi.yml
---

# Ingest user data into Insider One

## Authenticate

Every request carries **two** headers. This host uses the partner/token pair, not a bearer token:

```
X-PARTNER-NAME: <your partner name, lowercase, no spaces>
X-REQUEST-TOKEN: <API key from InOne > Settings > Integration Settings>
Content-Type: application/json
```

A wrong partner name returns `400 {"success": false, "message": "Partner is invalid."}`, not a 401.
See `authentication/insider-authentication.yml` — the header names differ on every other Insider host.

## 1. Upsert profiles, attributes and events — `upsertUserData`

`POST https://unification.useinsider.com/api/user/v1/upsert`

The body is `{"users": [...]}`. Each user has three sections:

- `identifiers` — `email`, `uuid`, `phone_number`, plus anything you define under `identifiers.custom`
- `attributes` — Insider's default attributes at the root, **your own attributes nested under `attributes.custom`**
- `events` — `event_name` + `timestamp` + `event_params`, again with your own params under `event_params.custom`

**The single most consequential mistake is attribute placement.** A custom attribute written at the
root instead of inside `custom` is silently mismapped — no error is returned. Check the field against
the Attributes and Events page in the panel before sending.

Batch aggressively: the endpoint allows 25,000 requests/minute (shared with `deleteUserAttribute`)
and a 5 MB body. Over 5 MB you get
`400 {"error": "request body is bigger than maximum allowed size 5,000,000 bytes"}`.

Omitting `users` returns `400 {"error":"users must be defined: bad request"}`.

## 2. Read a profile back — `getUserProfiles`

`POST https://unification.useinsider.com/api/user/v1/profile`

Send `identifiers` plus an `attributes` array selecting the fields you want, and optionally an
`events` block with `start_date`/`end_date` (epoch seconds) and a `wanted` list of
`{event_name, params}`. This is the sparse-fieldset mechanism — ask only for what you need.

`404 {"error":"no such user for these identifiers: no data"}` means the identifier is unknown, not
that the request was malformed.

## 3. Remove an attribute — `deleteUserAttribute`

`POST https://unification.useinsider.com/api/user/v1/attribute/delete`. Shares the 25,000/minute
budget with the upsert, so a bulk cleanup will throttle your ingestion.

## 4. Bulk export — `exportRawUserData`

`POST https://unification.useinsider.com/api/raw/v1/export` with a `segment.segment_id`, an
`attributes` list (`["*"]` for everything), an `events` window, a `format` (e.g. `parquet`) and a
`hook` to receive the result.

**This endpoint allows one request per UTC day.** The counter resets at 00:00 UTC and failed
requests do not count. Do not put it behind a retry loop.

## Handle failure

There is no `Idempotency-Key` on this API. Upserts are idempotent by identity — the same identifiers
address the same profile — so a safe retry means re-sending the identical body, not a new attempt id.

On `429 {"error": "rejected: too many requests"}` back off exponentially and honour `Retry-After`
when it is present. There are no quota headers on success, so you cannot see the remaining budget:
track your own send rate. Full table in `rate-limits/insider-rate-limits.yml`.

Error envelopes vary across Insider hosts; on this one expect `{"error": "<text>"}` or
`{"success": false, "message": "<text>"}`. See `errors/insider-problem-types.yml`.
