---
name: Pace a Pexafy integration against its quota
description: Read the rate-limit headers and the usage endpoints so an agent slows down before it is throttled instead of after.
api: openapi/pexafy-api-openapi.json
operations:
  - get_usage_api_v1_usage_get
  - get_daily_usage_api_v1_usage_daily_get
  - get_monthly_usage_api_v1_usage_monthly_get
  - get_usage_by_key_api_v1_usage_by_key_get
generated: '2026-08-27'
method: generated
source: openapi/pexafy-api-openapi.json + rate-limits/pexafy-rate-limits.yml
---

# Pace a Pexafy integration against its quota

Pexafy meters two independent things: a **rate** (requests per minute or hour, by plan) and
a **monthly quota** (total requests per month, by plan). You can see both before you hit
either.

## 1. Read the headers on every response

Every response carries:

- `x-ratelimit-limit` — the ceiling for the current window
- `x-ratelimit-remaining` — what is left
- `x-ratelimit-reset` — Unix epoch seconds when the window resets
- `x-request-id` — the correlation id, mirrored into `meta.request_id`

Observed live on 2026-08-27: 100 on `/health`, 60 on an unauthenticated API route, with
independent counters — the limiter is per-route-class, not one global bucket.

Throttle off `x-ratelimit-remaining`. Do not wait for the `429`.

## 2. Read the monthly quota with the usage endpoints

- `get_usage_api_v1_usage_get` — `GET /api/v1/usage`, current month
- `get_daily_usage_api_v1_usage_daily_get` — `GET /api/v1/usage/daily`
- `get_monthly_usage_api_v1_usage_monthly_get` — `GET /api/v1/usage/monthly`
- `get_usage_by_key_api_v1_usage_by_key_get` — `GET /api/v1/usage/by-key`, per key

The provider's own `api-onboarding` descriptor makes `GET /api/v1/usage` the third step of
its first-run flow, in its words, "so an agent can pace itself". Call it at the start of a
long run and again periodically, not only after a failure.

## 3. Know your ceiling

| plan | requests/month | rate limit |
| --- | --- | --- |
| Free | 5,000 | 1,200/hour |
| Starter | 20,000 | 1,800/hour |
| Pro | 70,000 | 3,600/hour |
| Expert | 150,000 | 100/min |
| Team | 300,000 | 200/min |
| Business | 1,000,000 | 300/min |
| Enterprise | 2,000,000 | 500/min |

What happens when the monthly quota runs out is **not published** — the pricing page does
not say whether it throttles, blocks or bills. Treat exhaustion as a hard stop and alert.

## 4. When you do get a 429

Status `429`, `error.code` `RATE_LIMITED`. No `Retry-After` header was observed on
unauthenticated probes; the official Python SDK honours one when sent, so read it if
present and otherwise back off to `x-ratelimit-reset`.

## 5. Over MCP there is no equivalent

The hosted MCP server at `https://mcp.pexafy.com/mcp` exposes no usage tool — it surfaces
exhaustion conversationally instead. An agent that needs to pace itself must go through
REST.
