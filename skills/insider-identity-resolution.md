---
name: insider-identity-resolution
description: Change or remove the identifiers that address an Insider One user profile without orphaning or merging the wrong records.
api: Insider One Unification API
spec: openapi/insider-unification-openapi.yml
base_url: https://unification.useinsider.com
operations:
  - updateIdentifiers
  - deleteIdentifiers
  - getUserProfiles
generated: '2026-08-13'
method: generated
source: openapi/insider-unification-openapi.yml
---

# Manage Insider One identifiers

Insider One has no server-issued user ID in its public API. A profile is addressed only by the
identifiers you own — email, phone number, uuid, and your own custom identifiers. That makes this the
highest-risk surface in the estate: a bad write here silently merges or strands customer records.

Headers are the partner pair (`X-PARTNER-NAME` + `X-REQUEST-TOKEN`) as in
`skills/insider-ingest-user-data.md`.

## Change an identifier value — `updateIdentifiers`

`PATCH https://unification.useinsider.com/api/user/v1/identity`

```json
{"old_identifier": {"email": "sample@mail.com"},
 "new_identifier": {"email": "sample2@mail.com"}}
```

Exactly one entry in each object. The four documented failures are all constraint violations, and all
return 400:

| Response | Meaning |
| --- | --- |
| `identifier values are the same: bad request` | old and new are identical — nothing to do |
| `there must be exactly 1 entry for both new and old identifiers: bad request` | you sent zero or several |
| `new identifier already has a user assigned to it: bad request` | identifiers are globally unique; free the value first |
| `no valid identifier: bad request` | the identifier type is not enabled in Identity Resolution Management Settings |

Rate limit: 2,000 requests/minute.

## Remove an identifier — `deleteIdentifiers`

`DELETE https://unification.useinsider.com/api/user/v1/identity`

A profile must keep at least one identifier. Deleting the last one returns
`400 {"error": "you cannot delete the sole identifier specified for a user: bad request"}`.
Rate limit: 1,000 requests/minute — half the update budget.

## Verify before and after

Call `getUserProfiles` with the old identifier before the change and the new one after. A `404
{"error":"no such user for these identifiers: no data"}` on the new identifier after a reported
success means you have a problem, not an empty result.

## Sequencing rule

To move an identifier from profile A to profile B: read both with `getUserProfiles`, delete it from A
with `deleteIdentifiers` (checking A still has another identifier), then set it on B with
`updateIdentifiers`. There is no transaction and no idempotency key, so record your own progress —
a retry of a partially applied move will hit the uniqueness error, not re-apply cleanly.
