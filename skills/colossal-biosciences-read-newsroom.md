---
name: Read the Colossal Biosciences newsroom
description: Page through published Colossal Biosciences news posts from the colossal.com WordPress REST API, with authors, featured images and taxonomy resolved in a single request.
api: openapi/colossal-biosciences-content-openapi.yml
base_url: https://colossal.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getPosts, getPostsId, getCategories, getTags, getUsers, getUsersId]
---

# Read the Colossal Biosciences newsroom

Colossal Biosciences publishes no product API. This skill reads the **content** API that
colossal.com's WordPress install exposes — the company newsroom. Use it to retrieve what
Colossal has publicly said, not to reach any scientific or laboratory system.

## Before you start

- Base URL: `https://colossal.com/wp-json/wp/v2`
- **No authentication is required** for any step below. Every read operation is anonymous.
- **No API key exists.** Do not attempt to obtain one; there is no developer signup.
- Respect `Crawl-delay: 10` from `https://colossal.com/robots.txt`. There is no documented
  rate limit and no `X-RateLimit-*` or `Retry-After` header, so pace yourself deliberately.

## Steps

### 1. List posts, newest first — `getPosts`

`GET /wp/v2/posts?per_page=100&page=1&orderby=date&order=desc&_embed`

- `per_page` maximum is **100**; the default is 10.
- `_embed` inlines the author, featured media and terms into an `_embedded` object, which
  saves you the follow-up calls in step 3.
- Add `_fields=id,date,slug,link,title,excerpt,categories,tags,author,featured_media` to trim
  the payload when you do not need the full rendered `content`.

Read the total from the response headers, not the body:

- `X-WP-Total` — total posts in the collection (238 at harvest, 2026-08-02)
- `X-WP-TotalPages` — total pages at the current `per_page`

Loop `page` from 1 to `X-WP-TotalPages`. A `page` beyond the last returns HTTP 400
`rest_post_invalid_page_number` — stop on that rather than treating it as a failure.

### 2. Filter to what you need — `getPosts`

Combine these query parameters, all declared on the operation:

- `search=<term>` — full-text search within posts
- `categories=<id>` / `tags=<id>` — filter by term id (resolve ids in step 4)
- `after=<ISO8601>` / `before=<ISO8601>` — publication-date window
- `modified_after` / `modified_before` — change-detection window; use these to poll for
  updates instead of re-reading the whole collection
- `slug=<slug>` — fetch a known post without knowing its id
- `include` / `exclude` — explicit id sets

### 3. Fetch one post — `getPostsId`

`GET /wp/v2/posts/{id}?_embed`

The `title`, `content` and `excerpt` fields are objects with a `rendered` string containing
HTML entities (`&#038;` for `&`, `&#8217;` for `'`). Decode the entities and strip the HTML
before handing text to a model or a user.

If the id does not exist or is not publicly visible, you get HTTP 404 with
`{"code":"rest_post_invalid_id"}`.

### 4. Resolve taxonomy and authors — `getCategories`, `getTags`, `getUsers`, `getUsersId`

Only needed when you did **not** use `_embed`:

- `GET /wp/v2/categories?per_page=100` — id, name, slug, count
- `GET /wp/v2/tags?per_page=100`
- `GET /wp/v2/users?per_page=100` or `GET /wp/v2/users/{id}` — post authors

`GET /wp/v2/users/me` (`getUsersMe`) requires authentication and will return 401 for you.
Do not call it.

## Rules

- **Read-only.** Every write operation in this spec (`createPosts`, `updatePostsId`,
  `deletePostsId`, and the equivalents on every other resource) requires a WordPress
  Application Password over HTTP Basic that you do not have and must not attempt to acquire.
  A write attempt returns HTTP 401 `rest_forbidden`.
- **No idempotency contract exists.** There is no idempotency key header or parameter on any
  route. Do not retry a non-GET request.
- **Errors are not RFC 9457.** The envelope is
  `{"code": "<slug>", "message": "<human string>", "data": {"status": <int>}}` with
  `Content-Type: application/json`. Branch on `code`, not on the message text. See
  `errors/colossal-biosciences-problem-types.yml`.
- **Attribute the source.** Content returned here is Colossal Biosciences' own published
  communications. Cite the post `link` when you surface it.
