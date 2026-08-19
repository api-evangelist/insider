---
name: insider-campaign-analytics
description: Pull Insider One campaign and journey analytics, over REST or through the Insider One MCP server, without tripping the date-format and retention traps.
api: Insider One Analytics / Architect Analytics / SMS Analytics APIs + Insider One MCP
spec: openapi/insider-analytics-openapi.yml
base_url: https://analytics.api.useinsider.com
operations:
  - getEmailCampaignListV2
  - getEmailCampaignAnalyticsV2
  - getOverallAnalyticsV2
  - getOnsiteOverallAnalytics
  - getSmsCampaignList
  - getSmsCampaignAnalytics
  - getWebPushCampaignMetricsAnalytics
  - getAppPushAnalytics
  - getArchitectOverallAnalytics
  - getArchitectJourneyAnalytics
  - getArchitectChannelAnalytics
  - getConversionGoalsAnalytics
  - exportJourneyList
generated: '2026-08-13'
method: generated
source: openapi/insider-analytics-openapi.yml + openapi/insider-architect-analytics-openapi.yml + openapi/insider-sms-openapi.yml + mcp/insider-tool-crosswalk.yml
---

# Read Insider One analytics

## Two ways in — pick deliberately

**MCP** (`https://mcp.insiderone.com/mcp`, OAuth 2.0): 21 of the 35 tools are analytics reads, and
they map onto the REST operations below — see `mcp/insider-tool-crosswalk.yml`. Best for ad-hoc
questions from an agent. It does **not** cover OnSite, and it inherits the same REST rate limits.

**REST**: everything, including OnSite, and the only option for scheduled extraction.

## The three date formats

This is the trap. Three analytics hosts, three formats:

| Host | Parameter | Format |
| --- | --- | --- |
| `analytics.api.useinsider.com` | `startTime` / `endTime` | int64 epoch seconds |
| `architect-analytics.api.useinsider.com` | `statDate` | `dd/MM/yyyy - dd/MM/yyyy` range string |
| `mobile.api.useinsider.com` (InApp details) | `from` / `to` | `YYYY-MM-DD`, window ≤ 90 days |

A wrong format returns `400 {"error": "invalid date format for 'from'; expected format is YYYY-MM-DD"}`
on the mobile host and a generic bad request elsewhere.

## Retention

Analytics data is available for the **last 1 year only**. Requests reaching further back return
`406 Not Acceptable`. If you need history, extract on a schedule and keep it yourself.

## Email — prefer v2

`getEmailCampaignListV2` (`GET /email/v2/campaign/list?page=&perPage=`) then
`getEmailCampaignAnalyticsV2` (`GET /email/v2/campaign/statistics?campaignId=&startTime=`), plus
`getOverallAnalyticsV2` (`GET /email/v2/overall`). The v1 equivalents still run; v2 is the current
line. `page`/`perPage` are the only pagination in the estate — everything else returns whole sets.

Limit: 100 requests/minute per key on every analytics endpoint.

## Architect journeys

`exportJourneyList` (`GET /v1/journeys`) → `getArchitectJourneyAnalytics`
(`GET /v1/journey/{journeyId}`) → `getArchitectChannelAnalytics` (`GET /v1/element/{campaignId}`) for
the per-element drop-off. `getConversionGoalsInformation` lists configured goals;
`getConversionGoalsAnalytics` returns their performance. All 200 requests/minute.

## Other channels

- SMS: `getSmsCampaignList` (`GET /analytics/v1/list`), then `getSmsCampaignAnalytics`,
  `getOverallSmsCampaignAnalytics`, `getTransactionalSmsAnalytics`, `getOtpVerifySmsAnalytics` — all
  on `sms.useinsider.com`, all POST despite being reads.
- Web push: three POST statistics endpoints on `web-push.api.useinsider.com`, sharing 30/minute
  **in aggregate** — the tightest analytics budget in the estate.
- App push: `getAppPushAnalytics` on `mobile.useinsider.com`; raw per-user results come from
  `exportRawUserData`, which is limited to one call per UTC day.

## Failure handling

429 is documented, but the Web Push statistics endpoints signal exhaustion as
`403 {"error": {"message": "Too many attempts, please slow down the request. 30 Requests/minute is
allowed."}, "status": 403}`. Treat both as throttling. No quota headers exist on success.
