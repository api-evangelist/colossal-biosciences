---
name: Search colossal.com and resolve results
description: Run a cross-content-type search against the colossal.com WordPress REST API and resolve each lightweight result pointer into the full post or page object.
api: openapi/colossal-biosciences-content-openapi.yml
base_url: https://colossal.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getSearch, getPostsId, getPagesId, getTypes, getTypesType]
---

# Search colossal.com and resolve results

The site-wide search endpoint spans every public content type at once, which makes it the
right entry point when you do not know whether an answer lives in a news post or on a
species/glossary page. It returns **pointers, not documents** — resolving them is step 2.

## Before you start

- Base URL: `https://colossal.com/wp-json/wp/v2`
- No authentication. No API key. Honour `Crawl-delay: 10` from robots.txt.
- This searches Colossal's public website content only. It does not search genomic data,
  publications databases or any Colossal research system — none of those has a public API.

## Steps

### 1. Search — `getSearch`

`GET /wp/v2/search?search=<term>&per_page=100&page=1`

Optional narrowing parameters declared on the operation:

- `type=post` — restrict to a top-level search type
- `subtype=post` or `subtype=page` — restrict to a specific content type; omit or use `any`
  to search everything
- `exclude` / `include` — explicit id sets

Each result is deliberately thin:

```json
{ "id": 1234, "title": "…", "url": "https://colossal.com/…", "type": "post", "subtype": "page" }
```

Pagination works exactly as it does elsewhere: `X-WP-Total` and `X-WP-TotalPages` headers
plus an RFC 8288 `Link` header with `rel="next"`.

### 2. Resolve each pointer — `getPostsId` or `getPagesId`

Branch on `subtype`:

- `subtype == "post"` → `GET /wp/v2/posts/{id}?_embed`
- `subtype == "page"` → `GET /wp/v2/pages/{id}?_embed`

Do not guess. Calling `getPostsId` with a page id returns HTTP 404
`{"code":"rest_post_invalid_id"}`.

If you only need to show a link, the `url` already in the search result is enough — skip the
resolve call entirely rather than making a request you do not need.

### 3. Confirm what content types exist — `getTypes`, `getTypesType`

`GET /wp/v2/types` lists every registered content type and
`GET /wp/v2/types/{type}` describes one. Use these to discover valid `subtype` values for
step 1 instead of assuming the site only has posts and pages.

## When to prefer a different approach

- Searching **only** news? Use `getPosts` with `search=<term>` (see the read-newsroom skill) —
  you get full post objects in one call instead of pointers you then have to resolve.
- Enumerating **everything**? Page `getPosts` and `getPages` directly. Search is for lookup,
  not for a full crawl.

## Rules

- **Read-only**, anonymous. Never attempt a write; every one returns HTTP 401 `rest_forbidden`.
- **Decode before use.** `title.rendered` carries HTML entities (`&#038;`, `&#8217;`). Decode
  them and strip markup before presenting text.
- **No idempotency, no rate-limit headers.** Pace requests; do not retry non-GET requests.
- **Errors** use the WordPress envelope `{code, message, data.status}`, not
  `application/problem+json`. Branch on `code`. A malformed parameter returns HTTP 400
  `rest_invalid_param` with the offending names in `data.params`.
- **Cite the `url`** on every result you surface to a user.
