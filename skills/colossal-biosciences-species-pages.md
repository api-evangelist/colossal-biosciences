---
name: Browse Colossal species pages and media
description: Retrieve the Colossal Biosciences species, science and glossary pages from the colossal.com WordPress REST API and pull the associated media-library assets with their real source URLs.
api: openapi/colossal-biosciences-content-openapi.yml
base_url: https://colossal.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getPages, getPagesId, getMedia, getMediaId]
---

# Browse Colossal species pages and media

Colossal's substantive explanatory content — the per-species pages (mammoth, thylacine, dodo,
moa, dire wolf, blue buck), the science and technology pages, the glossaries and the ethics
FAQ — lives in the WordPress **pages** collection, not in the newsroom. This skill reads that
collection and the media library attached to it.

## Before you start

- Base URL: `https://colossal.com/wp-json/wp/v2`
- No authentication. No API key. Honour `Crawl-delay: 10` from robots.txt.
- Pages are marketing and educational content authored by Colossal. They are not peer-reviewed
  data and must not be presented as Colossal's scientific record.

## Steps

### 1. List pages — `getPages`

`GET /wp/v2/pages?per_page=100&page=1&_fields=id,slug,title,link,parent,menu_order`

Page through using `X-WP-TotalPages`. Useful filters, all declared on the operation:

- `slug=<slug>` — fetch a known page directly, e.g. `slug=mammoth`, `slug=thylacine`,
  `slug=dodo`, `slug=moa`, `slug=direwolf`, `slug=bluebuck`, `slug=species`,
  `slug=technology`, `slug=labs`, `slug=glossary`, `slug=education`, `slug=company`
- `parent=<id>` — children of a section page; use this to walk the site hierarchy
- `search=<term>` — text search within pages only
- `orderby=menu_order&order=asc` — the site's own intended ordering

Prefer `slug` over `id`. Slugs are stable and human-meaningful; ids are not.

### 2. Fetch one page — `getPagesId`

`GET /wp/v2/pages/{id}?_embed`

- `content.rendered` is full HTML with entities — decode and strip before use.
- `featured_media` is a media id; `_embed` inlines it under `_embedded["wp:featuredmedia"]`.
- Some pages carry an `acf` object (Advanced Custom Fields). Its shape is site-specific and
  undocumented — read it defensively and never assume a field is present.

### 3. Pull media — `getMedia`, `getMediaId`

`GET /wp/v2/media?parent=<page_id>&per_page=100` returns the attachments for one page, and
`GET /wp/v2/media/{id}` returns one asset.

Key fields:

- `source_url` — the real file URL. Use this, not a URL you construct.
- `media_type` / `mime_type` — `image`, `video`, `file`
- `media_details.sizes` — the pre-rendered size variants; pick the smallest that meets your
  need rather than always taking the full-size original
- `alt_text` — carry it through; do not invent alt text for a Colossal asset

Filter to images with `?media_type=image`.

## Rules

- **Read-only**, anonymous. `createMedia`, `updateMediaId`, `deleteMediaId`, `createPages`
  and every other write require a WordPress Application Password over HTTP Basic and return
  HTTP 401 `rest_forbidden` for you. Do not attempt them.
- **Media is copyrighted.** `source_url` points at Colossal Biosciences' own assets. Link to
  them; do not rehost, and do not present a Colossal image as your own or as generic stock.
- **No idempotency, no documented rate limits, no sandbox.** Pace requests; do not retry
  non-GET requests.
- **Errors** use `{code, message, data.status}` — not RFC 9457. A bad page id returns HTTP 404
  `rest_post_invalid_id`; an unregistered route returns `rest_no_route`.
- **Cite the page `link`** when you surface content from it.
