---
name: Find a usable photo with Pexafy
description: Search nine free stock-photo libraries in one semantic request, page through results, and emit the credit line the licence expects.
api: openapi/pexafy-api-openapi.json
operations:
  - search_photos_api_v1_search_photos_get
  - facet_sources_api_v1_facets_sources_get
  - facet_colors_api_v1_facets_colors_get
  - get_photo_api_v1_photos__photo_id__get
generated: '2026-08-27'
method: generated
source: openapi/pexafy-api-openapi.json + https://docs.pexafy.com/
---

# Find a usable photo with Pexafy

One request searches nine free-licence libraries and returns every photo in one schema.
Base URL `https://api.pexafy.com/api/v1`. Authenticate with `x-api-key: <key>` on every
call (a bearer token on the same routes works too).

## 1. Write the query as a sentence, not keywords

`search_photos_api_v1_search_photos_get` (`GET /api/v1/search/photos`) ranks by meaning.
`a tired developer at a desk, warm lamp light` beats `developer desk lamp`. Up to 500
characters. Queries work in any of 100+ languages.

`q` is optional if you send at least one filter — but sending neither returns
`400 MISSING_PARAMS`.

## 2. Filter, but read the vocabulary at runtime

`color_name`, `color_hex` (+ `color_tolerance`), `orientation`, `source`, `license_type`,
`photographer`, `after_date`. `orientation`, `source` and `license_type` are repeated
query parameters — repeat the parameter, do not comma-join, or the filter silently matches
nothing.

`color_name` and `color_hex` are mutually exclusive; sending both returns
`400 INVALID_PARAMS`.

Do not hardcode the accepted values. Call
`facet_sources_api_v1_facets_sources_get`, `facet_colors_api_v1_facets_colors_get`,
`facet_orientations_api_v1_facets_orientations_get` or
`facet_licenses_api_v1_facets_licenses_get` — the source list grows (`wikimedia` was added
in description 1.3.0) and a pinned list will quietly go stale.

## 3. Page with the cursor

The response carries `pagination.{per_page, has_more, next_cursor}`. Repeat the same
request adding `cursor=<next_cursor>`; stop when `has_more` is false. Set `per_page` (1-100,
default 20) on the first request only. Cursors expire in about 5 minutes — on expiry,
restart from page one rather than retrying the cursor.

## 4. Use the attribution the API hands you

Every photo carries `attribution.plain` and `attribution.html`, plus `license_type` and
`source_image_url`. Emit the credit line as given. Do not compose your own from
`photographer_full_name` — the licence terms differ per source and the API has already
resolved them.

## 5. Handle the errors that actually happen

Read `error.code`, never `error.message` (documented as subject to change). Log
`meta.request_id` — it is also returned as the `x-request-id` header and is what support
asks for.

| code | status | what to do |
| --- | --- | --- |
| `MISSING_PARAMS` | 400 | add `q` or a filter |
| `INVALID_PARAMS` | 400 | drop one of `color_name` / `color_hex` |
| `UNAUTHORIZED` | 401 | key missing, malformed or revoked |
| `PLAN_RESTRICTION` | 403 | the enriched field you asked for is not in this plan |
| `RATE_LIMITED` | 429 | back off; watch `x-ratelimit-remaining` |
| `SEARCH_TIMEOUT` | 504 | retry once, shortly |

Watch `x-ratelimit-remaining` and `x-ratelimit-reset` on every response rather than waiting
for the 429.
