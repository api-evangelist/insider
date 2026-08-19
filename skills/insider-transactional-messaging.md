---
name: insider-transactional-messaging
description: Send a transactional email, SMS or WhatsApp message through Insider One and reconcile the outcome, remembering that a 2xx means queued, not delivered.
api: Insider One Mail / SMS / WhatsApp / Gateway APIs
spec: openapi/insider-mail-openapi.yml
base_url: https://mail.useinsider.com
operations:
  - sendTransactionalEmails
  - sendTransactionalSingleSms
  - sendTransactionalBulkSms
  - sendTransactionalWhatsappTemplateMessage
  - sendConversationalWhatsappMessageTemplate
  - sendTransactionalWhatsappMessagesWithOauth20
  - updateTransactionalWhatsappWebhookSettingsWithOauth20
generated: '2026-08-13'
method: generated
source: openapi/insider-mail-openapi.yml + openapi/insider-sms-openapi.yml + openapi/insider-whatsapp-openapi.yml + openapi/insider-gateway-openapi.yml
---

# Send transactional messages with Insider One

## The rule that governs everything here

**A 2xx on a send endpoint means accepted and queued — not delivered.** Actual throughput is governed
by Messaging Per Second (MPS), a separate limit from the request rate limit, and over-rate traffic is
queued for the next interval rather than dropped. Never report "sent" to a user from the send
response alone.

## Pick the host and the header

Each channel is a different host with a different key header:

| Channel | Operation | Endpoint | Limit |
| --- | --- | --- | --- |
| Email | `sendTransactionalEmails` | `POST https://mail.useinsider.com/mail/v1/send` | 9,000/sec |
| SMS (one) | `sendTransactionalSingleSms` | `POST https://sms.useinsider.com/v1/send` | 200/sec |
| SMS (bulk) | `sendTransactionalBulkSms` | `POST https://sms.useinsider.com/v1/sendMultipleMessage` | 5/sec × 50 messages |
| WhatsApp | `sendTransactionalWhatsappTemplateMessage` | `POST https://whatsapp.useinsider.com/v1/send` | 1,000/sec |
| WhatsApp (OAuth v2) | `sendTransactionalWhatsappMessagesWithOauth20` | `POST https://gw.useinsider.com/api/wa/v2/transactional/messages/send` | not published |

The messaging hosts use `X-INS-AUTH-KEY`; the gateway uses an OAuth 2.0 `Authorization: Bearer`.
Check `authentication/insider-authentication.yml` before assuming a header.

## Email — `sendTransactionalEmails`

Body: `subject`, `tos[]`, `from`, `content[]` (`{type, value}`), with optional `cc`, `bcc`,
`reply_to`, `dynamic_fields` (substituted into `{{placeholders}}` in the content), `unique_args`
(your own tagging metadata) and `attachments`.

Missing-field failures come back as
`400 {"errors": ["Missing 'subject' parameter"], "message": "bad-request", "status": 400}` — a
different envelope from every other Insider host.

If your account has Transactional Email Domain Authentication enabled, an unauthenticated sender
domain returns 401 and needs a support ticket, not a code change.

Use `unique_args` as your correlation key: there is no request id on the response.

## WhatsApp — track the `key`

The send response returns a `key` (`"whatsapp-***********"`). That tracking key appears in **every**
webhook event for that message, and it is the only end-to-end correlation identifier Insider One
publishes. Store it against your own message record at send time.

## Reconcile via webhooks, not polling

Register a webhook URL — in the panel (WhatsApp Settings > Account APIs) or via
`updateTransactionalWhatsappWebhookSettingsWithOauth20` — and subscribe to `sent`, `delivered`,
`read`, `failed`. Notes that matter:

- A single message can produce both `delivered` and `failed` when the user is on multiple devices.
- Meta errors are forwarded verbatim: `{"error": {"message": "(#130429) Rate limit hit", "type":
  "OAuthException", "code": 130429}, "key": "..."}`. Branch on `error.code`; treat `error_data` as
  optional.
- "Delivery Report Missing" messages never produce an event at all — age them out yourself.
- Changing the webhook URL immediately breaks the previous binding. There is no dual-delivery window.
- Max 20 webhook URLs per account (10 bearer + 10 OAuth 2.0), no duplicates, and the API type
  (Transactional / Conversational / Verify) is fixed at creation.

Email, SMS, web push and app push have **no** webhook surface — for those, poll the analytics APIs
(`getEmailCampaignAnalytics`, `getTransactionalSmsAnalytics`) and remember analytics history is
capped at one year.

Full detail: `asyncapi/insider-whatsapp-webhooks.yml`.

## Retry safely

There is no idempotency key. A 504 note in the docs says identical content to the same user inside
15 minutes may be de-duplicated, but do not rely on that as a guarantee — hold your own sent-state
and do not blind-retry a send that may have been accepted.
