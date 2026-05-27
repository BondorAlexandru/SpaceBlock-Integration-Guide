# SEO & Social Metadata

Per-page and per-post SEO and social-share (Open Graph / Twitter) metadata,
authored in the CMS and injected server-side so social crawlers get rich link
previews.

## Why this needs the CMS (not just the app)

Most CMS-driven sites are client-side SPAs. Social crawlers (Facebook, LinkedIn,
Slack, iMessage, X) **do not run JavaScript** — they read the raw HTML response.
So meta tags injected by the app at runtime are seen only by the browser and
JS-rendering crawlers (Google), **not** by social share scrapers. Rich previews
require the tags to be in the *initial HTML*, which must be produced server-side.
Because the CMS is the server we control, it produces that HTML.

There are three layers, each filling a gap:

| Layer | Where | Covers |
|-------|-------|--------|
| Static defaults | `index.html` (`seo:start`…`seo:end` block) | Fallback preview for any un-injected request |
| **CMS meta-injection** | CMS render endpoint + host rewrite | **Social share previews** (the main goal) |
| Client-side manager | The consumer's client-side SEO module | Browser tab + Google, updated on SPA navigation |

## 1. The `seo` object on the API

Pages and blog posts now expose an optional `seo` object. It appears on the
page object (`GET /api/public/pages/{projectId}/{slug}` → `page`) and the blog
post object (`GET /api/public/blog/{slug}` → `post`):

```json
"seo": {
  "title":       "string | null",   // falls back to page title / post title
  "description": "string | null",   // falls back to page description / post excerpt
  "image":       "string | null",   // ABSOLUTE url, 1200x630 PNG/JPG. Falls back to post.featuredImage, then project default
  "canonical":   "string | null",   // absolute canonical url; defaults to the page url
  "type":        "website | article | null",
  "noindex":     "boolean | null"   // true → robots: noindex,nofollow
}
```

All fields are optional/nullable — the API and the render endpoint fall back to
the existing `title`, `description`, `excerpt`, and `featuredImage`, then to the
project-wide defaults. Shipping the fields is purely additive; the consumer's
client-side SEO module already understands this shape.

### Resolution / fallback rules

Each tag value is resolved through a fallback chain (first non-empty wins):

- **title** → `seo.title` → page/post title → project site name
- **description** → `seo.description` → page description / post excerpt → project default description
- **image** → `seo.image` → `post.featuredImage` → project default image — always made **absolute**
- **url** → `seo.canonical` → site origin + the request path
- **type** → `seo.type` → `article` for posts, `website` for pages
- **noindex** → `seo.noindex === true`

## 2. Authoring SEO

There are three places to set SEO, mirroring the fallback chain:

- **Per blog post** — the SEO panel in the post editor sidebar (title,
  description, social image, canonical, type, noindex).
- **Per page** — the page **Settings** gear → **"SEO & Social"** section.
- **Project-wide defaults** — **Project → SEO** in the dashboard. Any project
  member can edit: **Site name**, **Default description**, **Default share
  image** (1200×630 PNG/JPG), and **Site URL**. The **Site URL** is both the
  origin used to build absolute URLs *and* the place the HTML shell is fetched
  from — so it must point at the live static site.

## 3. The render endpoint

The CMS serves the consumer's `index.html` shell with per-slug tags injected:

```
GET /api/public/render/{projectId}/{slug}        → page by slug
GET /api/public/render/{projectId}/blog/{slug}   → blog post by slug
GET /api/public/render/{projectId}               → home page (empty slug)
```

What it does:

1. Loads the page/post + its `seo`, resolving with the same fallback chain
   described above (per-field → page/post fields → **project SEO defaults**).
2. Fetches the `index.html` shell (from the configured **Site URL**, cached
   ~60s) and **replaces the block between `<!-- seo:start -->` and
   `<!-- seo:end -->`** with the resolved tags.
3. Returns `text/html`. The SPA still boots and hydrates normally for human
   visitors.

Notes:

- **Unknown slug → returns the shell with HTTP status `404` and injects
  `<meta name="robots" content="noindex,nofollow" />`** (still the shell, never
  a 500). This is the only place a *true* 404 status can be produced: a static
  host serves `index.html` with a 200 for every path, and client-side JS cannot
  change an already-sent status. Without this, non-existent routes are
  indexable.
- Unpublished slug (without `cms-preview`) → same treatment as unknown: 404 +
  noindex.
- Title/description are HTML-stripped to plain text and all attribute values are
  escaped.
- `og:image` / `og:url` / canonical are always absolute (resolved against the
  Site URL).
- Pass `?cms-preview=true` to resolve unpublished/draft content (returns
  `no-store`).

The exact tag set emitted into the `seo:start … seo:end` block:

```html
<title>{title}</title>
<meta name="description" content="{description}" />
<meta name="robots" content="{noindex ? 'noindex,nofollow' : 'index,follow'}" />
<meta property="og:site_name" content="{siteName}" />
<meta property="og:type" content="{type}" />
<meta property="og:title" content="{title}" />
<meta property="og:description" content="{description}" />
<meta property="og:image" content="{absolute image}" />
<meta property="og:url" content="{absolute url}" />
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:title" content="{title}" />
<meta name="twitter:description" content="{description}" />
<meta name="twitter:image" content="{absolute image}" />
<link rel="canonical" href="{absolute url}" />
```

## 4. Host rewrite

Route the **document** request (not static assets) to the render endpoint, so
crawlers and first paint get injected HTML. Replace `{projectId}` with your
project's id and `your-cms-host` with the CMS host. The pattern is the same on
every platform; pick the one matching your host.

**Vercel** (`vercel.json` rewrites):

```json
{
  "rewrites": [
    { "source": "/blog/:slug", "destination": "https://your-cms-host/api/public/render/{projectId}/blog/:slug" },
    { "source": "/:path*",     "destination": "https://your-cms-host/api/public/render/{projectId}/:path*" }
  ]
}
```

**Netlify** (`netlify.toml` — proxy redirects run BEFORE the SPA fallback):

```toml
[[redirects]]
  from = "/blog/*"
  to = "https://your-cms-host/api/public/render/{projectId}/blog/:splat"
  status = 200
  force = true

# Real static files (/assets/*, *.js, *.css, /index.html) are still served by
# Netlify; only un-matched paths are proxied.
[[redirects]]
  from = "/*"
  to = "https://your-cms-host/api/public/render/{projectId}/:splat"
  status = 200

# SPA fallback for anything the proxy doesn't handle
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

**Cloudflare** (Workers / `_redirects`, equivalent proxy rule):

```
/blog/*  https://your-cms-host/api/public/render/{projectId}/blog/:splat  200
/*       https://your-cms-host/api/public/render/{projectId}/:splat       200
```

**nginx** (`proxy_pass`, with static files served first):

```nginx
location /blog/ {
  proxy_pass https://your-cms-host/api/public/render/{projectId}/blog/;
}
location / {
  try_files $uri @render;   # serve real files, else proxy the document
}
location @render {
  proxy_pass https://your-cms-host/api/public/render/{projectId}$uri;
}
```

Caveats:

- **Keep `/index.html` reachable on the static host** — the render endpoint
  fetches it as the shell. Don't proxy `/index.html` itself to the endpoint (it
  would recurse).
- The home page (`/`) is usually served as the static `index.html`, so it shows
  the *default* tags. For per-home injection, add an explicit rewrite of `/` to
  `/api/public/render/{projectId}` (forcing the proxy ahead of the static file).

No serverless function is required in the app repo.

## Verification checklist

- In the CMS, open **Project → SEO** and set: Site name, Default description,
  Default share image (1200×630 PNG/JPG), and **Site URL** (the live
  static-site origin).
- Add the host rewrite from §4 (remember to keep `/index.html` reachable).
- Verify a shared URL in:
  - [Facebook Sharing Debugger](https://developers.facebook.com/tools/debug/)
  - [X/Twitter Card Validator](https://cards-dev.twitter.com/validator)
  - [LinkedIn Post Inspector](https://www.linkedin.com/post-inspector/)
- Confirm an unknown slug returns HTTP `404` with `robots: noindex,nofollow`.
