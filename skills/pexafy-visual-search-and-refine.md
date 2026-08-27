---
name: Search Pexafy by image and refine
description: Find photos that look like a reference image, blend an image with a text nudge, and walk outward with "more like this".
api: openapi/pexafy-api-openapi.json
operations:
  - search_photos_by_image_api_v1_search_photos_post
  - photo_similar_api_v1_photos__photo_id__similar_get
  - get_photo_api_v1_photos__photo_id__get
generated: '2026-08-27'
method: generated
source: openapi/pexafy-api-openapi.json + https://docs.pexafy.com/why-pexafy
---

# Search Pexafy by image and refine

Two operations, one loop: start from an image, then walk the neighbourhood.

## 1. Post the reference image

`search_photos_by_image_api_v1_search_photos_post` — `POST /api/v1/search/photos`,
multipart. Send the image file plus any of the same filters the text search takes
(`orientation`, `source`, `color_name`, `license_type`, `photographer`, `after_date`).

Supported formats are JPEG, PNG, WebP and AVIF. Anything else, or anything oversized,
returns `422 INVALID_IMAGE`. Calling this operation with no image at all returns
`422 MISSING_IMAGE`.

> The versioned description at github.com/Pexafy/pexafy-openapi places this operation at
> `POST /search/photos/by-image`. That path does not exist on the live host. Use
> `POST /api/v1/search/photos`, which is what `https://api.pexafy.com/openapi.json` serves.

## 2. Blend text with the image

Send `q` alongside the image to steer it — "like this, but at night" — and `text_alpha`
to weight the sentence against the picture. This is the capability a plain reverse-image
lookup does not have.

## 3. Page without re-uploading

Read `pagination.next_cursor` and send the next request with `cursor=` **only**. The cursor
already carries the image and the filters, so the upload happens once. Cursors last about
5 minutes.

## 4. Walk outward from a result

Once you have a `photo_id`, `photo_similar_api_v1_photos__photo_id__similar_get`
(`GET /api/v1/photos/{photo_id}/similar`) is cheaper than re-uploading — no encode, no
transfer. It paginates the same way, but on some plans the similar set is capped to a
single page, so `has_more` can be false immediately. That is a plan limit, not an empty
result.

`get_photo_api_v1_photos__photo_id__get` fetches one photo by id when you need to re-read
it later. Ids are UUIDs; a malformed one returns `400 INVALID_ID`, an unknown one
`404 PHOTO_NOT_FOUND`.

## 5. Judge the results

Every photo carries `relevance_score`. Since description 1.3.0 the search operations also
accept `score_threshold`, because semantic search always returns something — there is no
natural zero-result case to detect. If you need "nothing good enough", you have to set the
bar yourself.
