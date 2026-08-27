---
name: Curate a Pexafy collection
description: Create a saved set of photos, add and remove members, and understand exactly which of those steps can be taken back.
api: openapi/pexafy-api-openapi.json
operations:
  - list_collections_api_v1_collections_get
  - create_collection_api_v1_collections_post
  - get_collection_api_v1_collections__collection_id__get
  - add_photo_to_collection_api_v1_collections__collection_id__photos_post
  - remove_photo_from_collection_api_v1_collections__collection_id__photos__photo_id__delete
  - delete_collection_api_v1_collections__collection_id__delete
generated: '2026-08-27'
method: generated
source: openapi/pexafy-api-openapi.json + conventions/pexafy-conventions.yml
---

# Curate a Pexafy collection

Collections are the only writable objects in the Pexafy API, and the only place an agent
can do something it cannot undo. Read the reversibility section before the first `DELETE`.

## 0. Check you hold write scope

A read-scoped key gets `403` on everything under `/collections`. Keys created from the
dashboard carry read and write; an OAuth token issued to the MCP connector carries `read`
only, so **an agent reaching Pexafy over MCP cannot run this skill at all** — it needs a
REST key.

Collections are also plan-capped: 1 on Free, 3 on Starter, unlimited from Pro. Exceeding
the cap returns `403 COLLECTIONS_LIMIT_REACHED`.

## 1. Look before creating

`list_collections_api_v1_collections_get` (`GET /api/v1/collections`) first. There is **no
idempotency key on this API** — retrying `create_collection_api_v1_collections_post`
(`POST /api/v1/collections`) after a timeout creates a second collection with the same
name. Listing first is the only guard.

`create_collection_api_v1_collections_post` returns `201` with the new collection's `id`.
Collection ids are **integers**, not UUIDs — unlike `photo_id`, which is a UUID.

## 2. Add and remove members

- `add_photo_to_collection_api_v1_collections__collection_id__photos_post` —
  `POST /api/v1/collections/{collection_id}/photos`, `201`. Adding a photo that is already
  in the collection returns `409 PHOTO_ALREADY_IN_COLLECTION` rather than duplicating it,
  so this one operation is safe to retry.
- `remove_photo_from_collection_api_v1_collections__collection_id__photos__photo_id__delete`
  — `DELETE /api/v1/collections/{collection_id}/photos/{photo_id}`.

These two invert each other exactly. Anything you add, you can remove; anything you
remove, you can add back — provided you still hold the `photo_id`.

## 3. Reversibility — read this before deleting

| action | reversal | window |
| --- | --- | --- |
| create a collection | delete it | none stated |
| add a photo | remove the photo | none stated |
| remove a photo | add it back | none stated |
| **delete a collection** | **none** | **—** |

`delete_collection_api_v1_collections__collection_id__delete` is **irreversible**. There is
no restore, undelete or trash operation in the contract and the docs describe none. The
membership list is gone.

Before deleting, call `get_collection_api_v1_collections__collection_id__get`
(`GET /api/v1/collections/{collection_id}`) and keep the returned `data.photos[].photo_id`
values. There is no separate list-the-photos operation — `POST` is the only method on
`/api/v1/collections/{collection_id}/photos` — so this single read is the whole backup, and
it is the only thing that makes the delete recoverable: recreate the collection and re-add
each id.

The contract says so itself. The `delete` operation's own description reads: *"Permanently
delete a collection and remove every photo saved in it. This cannot be undone (the original
photos in the library are not affected)."*

No published retention window applies to any of these operations; do not assume one.

## 4. Errors

`403 FORBIDDEN` (scope or plan), `403 COLLECTIONS_LIMIT_REACHED`, `404 NOT_FOUND`
(collection gone), `409 PHOTO_ALREADY_IN_COLLECTION`, `422 VALIDATION_ERROR` (read
`error.fields`). Log `meta.request_id` on every failure.
