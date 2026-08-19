---
name: insider-consent-and-erasure
description: Honour a GDPR/CCPA request or a channel opt-out in Insider One — consent, unsubscribe, anonymize and delete, across three different hosts.
api: Insider One Unification API + Contact API + Mobile API
spec: openapi/insider-unification-openapi.yml
base_url: https://unification.useinsider.com
operations:
  - setDataProcessingConsentForAppUsers
  - deleteUserProfile
  - deleteUserPiiDataUsingIdentifier
  - deleteUserPiiDataUsingProfileId
  - unsubscribeEmailUsersFromDatabase
  - unsubscribeEmailUsersForEmailGlobalUnsubscription
  - unsubscribeSmsUsersFromDatabase
  - unsubscribeWhatsappUsersFromDatabase
  - resubscribeEmailUsers
generated: '2026-08-13'
method: generated
source: openapi/insider-unification-openapi.yml + openapi/insider-contact-openapi.yml + openapi/insider-mobile-openapi.yml
---

# Consent, opt-out and erasure in Insider One

These four things are **different operations on three different hosts**, and doing the wrong one is
the classic compliance failure here. Decide which you actually need before you call anything.

| Intent | Operation | Host |
| --- | --- | --- |
| Stop messaging on one channel | `unsubscribe*` | `contact.useinsider.com` |
| Withdraw data-processing consent (app) | `setDataProcessingConsentForAppUsers` | `mobile.useinsider.com` |
| Strip PII, keep the behavioural record | `deleteUserPiiDataUsingIdentifier` / `...UsingProfileId` | `unification.useinsider.com` |
| Erase the profile | `deleteUserProfile` | `unification.useinsider.com` |

## 1. Channel opt-out — `contact.useinsider.com`

v1 and v2 both run. **v2 is not a replacement, it is a different scope**: v1 unsubscribes from the
channel database; v2 distinguishes *global* unsubscription from *group* unsubscription and supports
bulk ingestion. Pick deliberately — `unsubscribeEmailUsersForEmailGlobalUnsubscription` (v2) and
`unsubscribeEmailUsersFromDatabase` (v1) are not interchangeable.

Resubscribe operations exist for every channel (`resubscribeEmailUsers`, `resubscribeSmsUsers`,
`resubscribeWhatsappUsers`) and set an unreachable address back to reachable. Only call these with
recorded, evidenced consent.

Rate limit: 600 requests/minute per key on the v1 subscribe endpoints.

## 2. App data-processing consent — `setDataProcessingConsentForAppUsers`

`POST https://mobile.useinsider.com/api/v1/privacy/gdpr/consent/set`. This is separate from channel
subscription state; setting one does not set the other.

## 3. Anonymize — keep the record, remove the person

`POST https://unification.useinsider.com/api/user/v1/anonymize` (by identifier) or
`/api/contact/v1/anonymize` (by profile id). Rate limit: **500 requests/minute**, the tightest budget
in the Unification API — plan a large erasure backlog around it.

## 4. Delete the profile — `deleteUserProfile`

`POST https://unification.useinsider.com/api/user/v1/delete` with `{"identifiers": {...}}`.
10,000 requests/minute. This removes the profile; anything referencing it by identifier will
subsequently 404.

## Order of operations for a subject request

1. `getUserProfiles` — capture what exists, for your own audit record.
2. Unsubscribe on every channel the subject is reachable on.
3. `setDataProcessingConsentForAppUsers` if they are an app user.
4. Anonymize or delete, per your retention policy.
5. `getUserProfiles` again — a `404 no such user for these identifiers` is your completion evidence.

There is no idempotency key and no bulk erasure job. Log every call yourself: Insider One returns no
request id, so your own log is the only trace.
