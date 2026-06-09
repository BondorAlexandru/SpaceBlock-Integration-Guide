# 14. CDN cache invalidation

This is a **design document**, not a how-to. It specifies how ContentLite caches
public API responses at the CDN edge, and how it notifies your site to purge
that cache when an editor publishes new content. Implementation lands in a
single coordinated change across the CMS, your site, and the project settings UI.

The goal: **your site loads ~10× faster, ContentLite carries ~100× less
traffic, and editor publishes still appear within a few seconds.**

---

## Status

| Item | State |
|---|---|
| Date | 2026-06-09 |
| Status | Approved design, pending implementation |
| Affected repos | `CMS`, `SpaceBlock-Integration-Guide`, every consumer site |
| Risk class | Medium — changes the default caching contract for every project |
| Rollout | All four parts in one coordinated PR set |

---

## Why this exists

Today, every public ContentLite endpoint emits:

```
Cache-Control: no-store, no-cache, must-revalidate, max-age=0
CDN-Cache-Control: no-store
Vercel-CDN-Cache-Control: no-store
```

That guarantees fresh data — but it also guarantees that every visitor to every
page hits the origin function, which queries Postgres on every request.

For a project receiving 10k visitors/day at 5 page views each, that's ~250,000
Vercel function invocations per project per day, none of which can be cached.
Multiply across every project on the platform.

The cache *wiring* is already correct: every route handler computes an ETag
from a content hash and passes `maxAge` into `getPublicApiHeaders`. The helper
just ignores those parameters and forcibly emits `no-store`. That was a
deliberate choice — without an invalidation channel, any non-zero `max-age`
delays editor publishes from showing up.

This design adds the invalidation channel, then re-enables the caching that
the route handlers were already prepared for.

---

## Architecture

Three caching layers, each independently useful, stacked:

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 3 — Consumer SPA: in-memory request coalescing        │
│   Concurrent fetches of the same URL in one SPA session     │
│   share a single Promise. No HTTP for duplicates.           │
└─────────────────────────────────────────────────────────────┘
                              ↓ on first request
┌─────────────────────────────────────────────────────────────┐
│ Layer 2 — Browser HTTP cache + ETag                         │
│   First visit hits Layer 1. Repeat visits send              │
│   `If-None-Match`; server returns 304 (~200 bytes).         │
└─────────────────────────────────────────────────────────────┘
                              ↓ on miss / revalidation
┌─────────────────────────────────────────────────────────────┐
│ Layer 1 — CDN edge (Vercel, Cloudflare, etc.)               │
│   `s-maxage` fresh window — serve from edge instantly.      │
│   `stale-while-revalidate` window — serve stale while       │
│   revalidating origin in background.                        │
└─────────────────────────────────────────────────────────────┘
                              ↓ on cold cache or webhook purge
┌─────────────────────────────────────────────────────────────┐
│ Origin — ContentLite serverless function + Postgres         │
└─────────────────────────────────────────────────────────────┘
```

**Invalidation path** (independent of the read path):

```
Editor clicks Publish
        ↓
CMS mutation handler succeeds
        ↓
firePurgeWebhook({ projectId, paths }) — fire-and-forget
        ↓
HMAC-signed POST to project's configured webhookUrl
        ↓
Consumer site verifies signature, purges affected paths from its CDN
        ↓
Within seconds, the next visitor to those paths sees fresh content
```

If the webhook fails for any reason, the SWR window in `Cache-Control` is the
self-healing backstop: content still updates within `s-maxage`, and visitors
see stale-but-revalidating responses during the SWR window.

---

## TTL strategy

```ts
// lib/utils.ts
export const HTTP_CACHE_TTL = {
  SHORT:  3600,     //  1h — frequently-edited content
  MEDIUM: 21600,    //  6h — pages, collection lists
  LONG:   86400,    // 24h — global elements, blog post bodies, SEO
  STATIC: 31536000, // 1y  — hashed assets (unchanged)
} as const
```

| Tier | Used by | Why this TTL |
|---|---|---|
| `SHORT` (1h) | Collection items (frequently reordered, content added) | Editors touch these often; short window minimizes staleness if webhook fails |
| `MEDIUM` (6h) | Page bodies, collection lists, individual blog posts | Edited occasionally; 6h fits a typical editorial workflow |
| `LONG` (24h) | Global elements (navbar, footer), SEO settings, render endpoint | Rarely changed; long window maximizes cache hit rate |

**Stale-while-revalidate window: 7 days** for all tiers. If the origin is
unreachable during background revalidation, visitors continue to see the
stale-but-still-recent response.

**Rationale for these specific values:** they balance two failure modes:

1. **Webhook fails to fire** (network, timeout, misconfiguration). Worst case
   for the editor: their publish takes up to `s-maxage` to appear to visitors
   who hit a cold edge. With these values that's 1h, 6h, or 24h depending on
   what changed.
2. **Webhook fires correctly.** The editor's publish appears to visitors within
   1–3 seconds regardless of TTL. So picking long TTLs has no downside in the
   happy path.

Editors mid-flow always see fresh content because the CMS preview mode
(`?cms-preview=true` query param or `X-CMS-Preview: true` header) bypasses the
cache via `getNoCacheHeaders()`. That path is already in `lib/utils.ts:240`
and doesn't change.

---

## Schema changes

### `Project` model — one new column

```prisma
// prisma/schema.prisma
model Project {
  // ... existing fields ...
  cdnSettings  Json  @default("{}")
}
```

Stored shape:

```ts
type CdnSettings = {
  /**
   * URL on the consumer site that ContentLite POSTs to on publish events.
   * Empty / null means the project has not opted into webhook invalidation —
   * it still benefits from the SWR window, just with longer lag.
   */
  purgeWebhookUrl?: string

  /**
   * Shared secret used to sign webhook payloads (HMAC-SHA256).
   * 32+ random bytes encoded as hex.
   * Generated on first save; rotatable via the settings UI.
   */
  purgeSecret?: string

  /**
   * Optional opt-out: if true, public endpoints emit no-store regardless
   * of TTL config. For clients who explicitly don't want caching.
   * Default: false (caching enabled).
   */
  disableCdnCache?: boolean
}
```

**Why a JSON column rather than dedicated columns:** matches the existing
pattern (`seoSettings`, `featureFlags`); avoids a structural migration each
time we add a CDN-related field; keeps settings discoverable in one place.

**Why not reuse `seoSettings`:** CDN settings have nothing to do with SEO
semantically, and reusing the column would couple unrelated UI surfaces.

**No migration data needed.** `@default("{}")` means existing projects get
empty settings; they continue working exactly as before until an editor opts
in via the settings UI.

---

## API: header changes

### `getPublicApiHeaders` — re-enable conditional caching

```ts
// lib/utils.ts
export function getPublicApiHeaders(
  request?: Request,
  options?: {
    maxAge?: number
    staleWhileRevalidate?: number
    etag?: string
  }
): Record<string, string> {
  const corsHeaders = getCorsHeaders(request)
  const headers: Record<string, string> = {
    ...corsHeaders,
    'Vary': 'Origin, x-api-key, Accept-Encoding',
  }
  if (options?.etag) headers['ETag'] = `"${options.etag}"`

  // CMS preview mode → always no-store. Editors must see fresh content.
  if (request && isCmsPreviewMode(request)) {
    headers['Cache-Control'] = 'no-store, no-cache, must-revalidate'
    headers['Pragma'] = 'no-cache'
    return headers
  }

  const maxAge = options?.maxAge ?? 0
  if (maxAge <= 0) {
    headers['Cache-Control'] = 'no-store, no-cache, must-revalidate, max-age=0'
    return headers
  }

  const swr = options?.staleWhileRevalidate ?? 7 * 24 * 3600 // 7 days
  headers['Cache-Control'] = `public, max-age=0, s-maxage=${maxAge}, stale-while-revalidate=${swr}`
  headers['Vercel-CDN-Cache-Control'] = `public, max-age=${maxAge}, stale-while-revalidate=${swr}`
  // NOTE: do NOT emit `CDN-Cache-Control: no-store` — that was the kill switch.

  return headers
}
```

**Per-project opt-out:** route handlers check `project.cdnSettings.disableCdnCache`
before computing maxAge:

```ts
const effectiveMaxAge = project.cdnSettings?.disableCdnCache ? 0 : HTTP_CACHE_TTL.MEDIUM
return Response.json(data, {
  headers: getPublicApiHeaders(req, { maxAge: effectiveMaxAge, etag }),
})
```

### 304 Not Modified support

Add at the top of each public route handler, before the database query:

```ts
const etag = generateEtag(data) // moved up — needs computing before headers
const ifNoneMatch = req.headers.get('If-None-Match')
if (ifNoneMatch === `"${etag}"`) {
  return new Response(null, {
    status: 304,
    headers: getPublicApiHeaders(req, { maxAge, etag }),
  })
}
```

For this to work without an extra database query, route handlers need to be
restructured to compute the ETag from the version timestamp / row hash before
serializing the full response. For endpoints already loading the data anyway,
the savings are bandwidth-only, which is still worthwhile.

---

## API: the webhook contract

### Outbound (CMS → consumer site)

When an editor publishes content, the CMS POSTs to `cdnSettings.purgeWebhookUrl`:

**Request:**

```http
POST /api/revalidate-cdn HTTP/1.1
Host: cosmos-institute.org
Content-Type: application/json
X-ContentLite-Signature: sha256=4ab1c8...
X-ContentLite-Event: content.published
X-ContentLite-Delivery: 7f8e3a... (uuid for log correlation)
User-Agent: ContentLite-Webhook/1.0

{
  "projectId": "100744a7-0909-464b-a151-731f18fd0b63",
  "event": "content.published",
  "paths": ["/fast-grants", "/"],
  "collections": ["team"],
  "timestamp": 1717920000000
}
```

**Payload fields:**

| Field | Type | Meaning |
|---|---|---|
| `projectId` | string | Lets receivers reject events from other tenants if they're multi-tenant |
| `event` | string | `content.published` \| `content.unpublished` \| `content.deleted` \| `global.updated` |
| `paths` | string[] | Site paths to purge. `["*"]` means full-site purge (used when global elements change) |
| `collections` | string[]? | Collection slugs affected, for sites that fetch collection lists separately |
| `timestamp` | number | Unix ms; receivers should reject events more than 5 min old to prevent replay |

**Signature:** `X-ContentLite-Signature: sha256=<hmac>` where `<hmac>` is
HMAC-SHA256 of the raw request body using the project's `purgeSecret` as the key.

**Expected response:** `200 OK` within 3 seconds. The CMS does not retry on
failure — the SWR window is the backstop. Receivers that need retry semantics
should queue purges internally (e.g. via SQS, Cloudflare Queues).

### Inbound (CMS receives nothing)

The CMS does not consume webhooks. This is a one-way notification from CMS
to consumer site.

---

## API: receiver reference implementations

### Vercel Edge Function (Cosmos and most React/Vue SPAs)

```ts
// api/revalidate-cdn.ts
import crypto from 'node:crypto'

export const config = { runtime: 'edge' }

export default async function handler(req: Request): Promise<Response> {
  if (req.method !== 'POST') return new Response('Method Not Allowed', { status: 405 })

  const secret = process.env.CDN_PURGE_SECRET
  if (!secret) return new Response('Not configured', { status: 503 })

  // Verify signature before parsing body — avoids parsing untrusted JSON
  // from a request we can't authenticate.
  const body = await req.text()
  const signature = req.headers.get('X-ContentLite-Signature') ?? ''
  if (!signature.startsWith('sha256=')) {
    return new Response('Missing signature', { status: 401 })
  }
  const expected = 'sha256=' + crypto.createHmac('sha256', secret).update(body).digest('hex')
  // Timing-safe comparison to prevent length-extension / timing attacks
  const sigBuf = Buffer.from(signature)
  const expBuf = Buffer.from(expected)
  if (sigBuf.length !== expBuf.length || !crypto.timingSafeEqual(sigBuf, expBuf)) {
    return new Response('Invalid signature', { status: 401 })
  }

  const payload = JSON.parse(body) as { paths: string[]; timestamp: number }

  // Replay protection: reject events more than 5 minutes old.
  if (Math.abs(Date.now() - payload.timestamp) > 5 * 60 * 1000) {
    return new Response('Stale event', { status: 401 })
  }

  // Purge each path. On Vercel, the supported mechanism for a static SPA is
  // re-fetching with a cache-bypass query string to evict the edge cache.
  // (For Next.js App Router sites, use `revalidatePath` instead.)
  const origin = `https://${process.env.VERCEL_URL}`
  await Promise.all(
    payload.paths.map(p =>
      fetch(`${origin}${p}?_purge=${payload.timestamp}`, { cache: 'no-store' })
        .catch(() => {/* best-effort */})
    )
  )

  return new Response('OK', { status: 200 })
}
```

**Required env var on the consumer site:** `CDN_PURGE_SECRET` matching the
project's `purgeSecret` in CMS.

### Next.js App Router site (better off-the-shelf invalidation)

```ts
// app/api/revalidate-cdn/route.ts
import { revalidatePath } from 'next/cache'

export async function POST(req: Request) {
  // ... same signature verification ...
  for (const path of payload.paths) revalidatePath(path)
  return Response.json({ ok: true })
}
```

### Cloudflare Workers / Pages

```ts
export default {
  async fetch(req: Request, env: Env): Promise<Response> {
    // ... same signature verification ...
    // Cloudflare Cache API:
    const cache = caches.default
    await Promise.all(
      payload.paths.map(p => cache.delete(`https://${url.host}${p}`))
    )
    return new Response('OK')
  },
}
```

### Netlify Functions

```ts
// netlify/functions/revalidate-cdn.ts
import { purgeCache } from '@netlify/functions'
export default async (req: Request) => {
  // ... same signature verification ...
  await purgeCache({ tags: payload.paths })
  return new Response('OK')
}
```

---

## CMS implementation

### `lib/webhooks/cdn-purge.ts` (new)

```ts
import crypto from 'node:crypto'
import { prisma } from '@/lib/prisma'

interface PurgePayload {
  projectId: string
  event: 'content.published' | 'content.unpublished' | 'content.deleted' | 'global.updated'
  paths: string[]
  collections?: string[]
  timestamp: number
}

/**
 * Fire-and-forget HMAC-signed POST to a project's configured purge webhook.
 *
 * Never throws. The SWR window in Cache-Control is the backstop — content
 * still propagates within HTTP_CACHE_TTL if this fails. We log warnings so
 * persistent failures are visible in monitoring without blocking mutations.
 */
export async function firePurgeWebhook(payload: Omit<PurgePayload, 'timestamp'>): Promise<void> {
  const project = await prisma.project.findUnique({
    where: { id: payload.projectId },
    select: { cdnSettings: true },
  })
  const settings = (project?.cdnSettings ?? {}) as {
    purgeWebhookUrl?: string
    purgeSecret?: string
  }
  if (!settings.purgeWebhookUrl || !settings.purgeSecret) return

  const fullPayload: PurgePayload = { ...payload, timestamp: Date.now() }
  const body = JSON.stringify(fullPayload)
  const signature = crypto
    .createHmac('sha256', settings.purgeSecret)
    .update(body)
    .digest('hex')

  try {
    await fetch(settings.purgeWebhookUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'User-Agent': 'ContentLite-Webhook/1.0',
        'X-ContentLite-Signature': `sha256=${signature}`,
        'X-ContentLite-Event': payload.event,
        'X-ContentLite-Delivery': crypto.randomUUID(),
      },
      body,
      signal: AbortSignal.timeout(3000),
    })
  } catch (err) {
    console.warn(`[cdn-purge] ${payload.projectId} webhook to ${settings.purgeWebhookUrl} failed:`, err)
  }
}
```

### `lib/webhooks/paths.ts` (new)

Centralized path-computation helpers so each mutation handler is one line:

```ts
export function pathsForPageUpdate(page: { slug: string }): string[] {
  // The page itself + home (in case home links to it).
  // Receivers MAY normalize trailing slashes.
  return [`/${page.slug}`, '/']
}

export function pathsForBlogUpdate(post: { slug: string }): string[] {
  return [`/blog/${post.slug}`, '/blog', '/']
}

export function pathsForCollectionItemUpdate(collectionSlug: string): string[] {
  // Any page can embed any collection. Without a reverse index from collection
  // → pages that reference it, full-site purge is the safe default.
  // (Future optimization: a `PageElement → Collection` lookup table that lets
  // us purge only pages actually referencing the changed collection.)
  return ['*']
}

export function pathsForGlobalElementUpdate(): string[] {
  // Navbar/footer affects every page.
  return ['*']
}

export function pathsForSeoUpdate(): string[] {
  // SEO defaults bleed into every page's meta tags.
  return ['*']
}
```

### Mutation handler integration

Each publish-touching endpoint adds one line at the end of the success path:

```ts
// app/api/projects/[projectId]/pages/[pageId]/route.ts (PUT)
import { firePurgeWebhook } from '@/lib/webhooks/cdn-purge'
import { pathsForPageUpdate } from '@/lib/webhooks/paths'

// ... after updatedPage = await prisma.page.update(...) ...
if (updatedPage.isPublished) {
  // Fire-and-forget. Do not await — editor save must stay snappy.
  void firePurgeWebhook({
    projectId,
    event: 'content.published',
    paths: pathsForPageUpdate(updatedPage),
  })
}
```

**All mutation surfaces that need this hook:**

| Endpoint | Method | Event | Path helper |
|---|---|---|---|
| `app/api/projects/[projectId]/pages/[pageId]/route.ts` | PUT, DELETE | published / deleted | `pathsForPageUpdate` |
| `app/api/projects/[projectId]/blog/[postId]/route.ts` | PUT, DELETE | published / deleted | `pathsForBlogUpdate` |
| `app/api/projects/[projectId]/collections/[collectionId]/route.ts` | PUT, DELETE | published / deleted | `pathsForCollectionItemUpdate` |
| `app/api/projects/[projectId]/collections/[collectionId]/items/[itemId]/route.ts` | PUT, DELETE, POST | published / deleted | `pathsForCollectionItemUpdate` |
| `app/api/projects/[projectId]/global-elements/[elementId]/route.ts` | PUT | global.updated | `pathsForGlobalElementUpdate` |
| `app/api/projects/[projectId]/seo/route.ts` | PUT | global.updated | `pathsForSeoUpdate` |

---

## Settings UI

New section in `app/projects/[projectId]/settings/page.tsx`:

```
CDN cache invalidation
══════════════════════

When you publish content, ContentLite notifies your hosting platform to
refresh its cache. Without this, changes take up to 24 hours to fully
propagate (your site still serves stale content gracefully in the meantime).

  Purge webhook URL
  ┌────────────────────────────────────────────────────────┐
  │ https://your-site.com/api/revalidate-cdn               │
  └────────────────────────────────────────────────────────┘
  POST endpoint on your site that triggers CDN purge.
  Leave empty to skip — site will still benefit from the SWR window.

  Webhook secret                          [ Rotate ]  [ Reveal ]
  ┌────────────────────────────────────────────────────────┐
  │ •••••••••••••••••••••••••••••••••                      │
  └────────────────────────────────────────────────────────┘
  Used to sign webhook payloads (HMAC-SHA256). Copy this into
  your site's CDN_PURGE_SECRET env var.

  [ Test webhook ]  Sends a no-op POST to verify your endpoint
                    is reachable and signature verification works.

  ☐ Disable CDN caching entirely
    Public API responses will emit `no-store`. Useful for projects
    where the SWR window's 24h max staleness is too long and you
    can't configure a webhook.
```

**"Test webhook" button:** POSTs a `content.test` event with empty paths. The
receiver should respond 200 OK. Successful test → green check; failure →
shows the error text and the response status returned.

**Rotate secret:** generates a new 32-byte hex secret, displays it once, never
shows it again unless rotated again. Standard "secret was here" pattern.

---

## Consumer-side cleanup (Cosmos as the reference example)

### Drop cache-bust hacks

Every existing `fetch(`...?_t=${Date.now()}`, { cache: 'no-store' })` becomes
a normal `fetch(url)`. 15 call sites in Cosmos as of writing — they all collapse
to one `cmsFetch` wrapper:

```ts
// src/lib/cmsFetch.ts
const inFlight = new Map<string, Promise<unknown>>()

/**
 * Fetch a CMS URL with in-session request coalescing.
 *
 * Concurrent calls to the same URL within one SPA session share a single
 * Promise — no duplicate HTTP. Browser HTTP cache (via standard ETag /
 * If-None-Match) handles cross-session caching; the CDN handles cross-visitor
 * caching.
 *
 * This is intentionally NOT a persistent cache. It just deduplicates
 * concurrent in-flight requests. For longer-lived caching, configure a
 * service worker or use a library like SWR/React Query.
 */
export async function cmsFetch<T>(url: string, init?: RequestInit): Promise<T> {
  const cached = inFlight.get(url) as Promise<T> | undefined
  if (cached) return cached
  const promise = fetch(url, init).then(async r => {
    if (!r.ok) throw new Error(`${r.status} ${r.statusText} for ${url}`)
    return r.json() as Promise<T>
  })
  inFlight.set(url, promise)
  promise.finally(() => inFlight.delete(url))
  return promise
}
```

### Effect

For Cosmos's 10 routes after this change:

| Scenario | Today | After |
|---|---|---|
| First visit, cold CDN | ~250ms TTFB | ~250ms TTFB (no change — cache miss either way) |
| First visit, warm CDN | ~250ms TTFB | ~30ms TTFB (CDN edge hit) |
| Repeat visit, same session | ~250ms TTFB | 0ms (Layer 3 dedupes) |
| Repeat visit, different session | ~250ms TTFB | ~50ms TTFB (Layer 2: 304 from CDN) |
| Editor publish appears | immediate | 1–3 seconds (webhook) or ≤24h (SWR backstop) |

---

## Failure modes and recovery

| Failure | Symptom | Recovery |
|---|---|---|
| Webhook receiver is down | Editors publish, content takes up to `s-maxage` to appear | Self-healing — SWR refreshes within window. Receiver comes back online, next publish purges normally. |
| Webhook receiver returns 500 | Same as above | Same — no retry, SWR backstop. CMS logs warning. |
| Webhook signature mismatch | Editor publish doesn't appear; nothing in CMS logs (CMS only sees fire-and-forget result) | Receiver logs 401. Operator checks `CDN_PURGE_SECRET` env var matches the project's secret. |
| Editor publishes during CDN provider outage | Content updates queued at edge, may serve stale | Out of our control; SWR ensures visitors see *something* |
| Editor publishes thousands of items rapidly | Webhook fires for each, ratelimit on receiver | Recommend receivers debounce / batch. Future: CMS-side debounce per-project (~250ms coalescing window) |
| Editor *unpublishes* content | Cached responses still serve old content until purged | The publish flow fires `content.unpublished` event — receiver purges. Public API also returns 404 for now-unpublished slugs (already implemented in render endpoint). |
| Webhook URL points to wrong host (typo) | 401/404/connection refused | Editor uses "Test webhook" button before going live; CMS dashboard surfaces last 10 webhook results with status. |

---

## Security

1. **HMAC signing prevents forged purges.** Without the secret, attackers
   can't trigger arbitrary cache purges on the consumer site (which would be
   a low-impact DoS but still unwanted).

2. **Timing-safe signature comparison** in the receiver reference
   implementations prevents timing-attack secret recovery.

3. **Replay protection via 5-minute timestamp window.** Captured webhook
   requests can't be replayed indefinitely.

4. **Secret rotation** via the settings UI without code change. Compromised
   secret? Rotate, update the env var, done.

5. **Secret transmission:** the secret is never sent over the wire after
   initial display. The CMS stores it; the receiver stores it. Only HMAC
   signatures cross the wire.

6. **Per-project isolation:** webhooks are project-scoped. A leaked secret
   for project A cannot purge project B's cache.

---

## Migration

This is a **backward-compatible additive change** for any project that has not
configured a webhook URL:

- Existing projects get `cdnSettings = {}` on rollout.
- With empty settings: `Cache-Control` now emits `s-maxage=21600,
  stale-while-revalidate=604800` for the medium tier (was `no-store`). Sites
  start benefiting from CDN caching immediately. Worst-case staleness: 6h for
  most content, 24h for global elements.
- Sites that *need* zero-cache behavior set `cdnSettings.disableCdnCache = true`.

**Existing consumer sites do not need any code change to benefit from Part 1**
(header changes). They just stop sending `?_t=` cache-busters at their own pace.

**Sites that want sub-second invalidation** add the receiver function and
configure the URL + secret in CMS settings.

---

## Rollout plan

All four parts ship as a coordinated set (per project decision 2026-06-09):

| Order | Repo | Change | Test |
|---|---|---|---|
| 1 | CMS | Schema migration: add `cdnSettings Json @default("{}")` | `prisma migrate dev` locally; verify existing projects unaffected |
| 2 | CMS | `lib/utils.ts` — TTL constants + `getPublicApiHeaders` rewrite | Curl a public endpoint; verify `Cache-Control: public, s-maxage=…` on non-preview requests; verify `no-store` on `?cms-preview=true` |
| 3 | CMS | Route handlers — add 304 If-None-Match handling | `curl -H 'If-None-Match: "abc"'` against an endpoint; first request returns 200 + ETag; second with matching ETag returns 304 |
| 4 | CMS | `lib/webhooks/{cdn-purge,paths}.ts` + hook into 6 mutation handlers | Configure a test receiver (Cosmos preview); publish a page; observe POST arrive with valid signature |
| 5 | CMS | Settings UI for purge URL + secret + Test button | Round-trip through the form; verify secret persists; click Test → 200 |
| 6 | Cosmos | `api/revalidate-cdn.ts` + `CDN_PURGE_SECRET` env var | Receive the test POST from CMS dashboard |
| 7 | Cosmos | `src/lib/cmsFetch.ts` + replace 15 cache-bust call sites | Page loads work identically; DevTools network shows no `_t=` query strings |
| 8 | Integration guide | This doc + receiver examples for non-Vercel hosts | Docs render; example snippets compile |
| 9 | Production cutover | Merge CMS PR → deploy → merge Cosmos PR → deploy → configure webhook in CMS dashboard | Publish a test page; observe propagation in 1–3 seconds |

**Rollback plan:** if Part 1 (header changes) causes problems, set
`disableCdnCache: true` on the affected project — emits `no-store` again
for that project only. The webhook stops firing as a side effect since there's
nothing to purge.

---

## Open questions

These don't block implementation but should be revisited after rollout:

1. **Should we add a CMS-side debounce per project?** Publishing 10 collection
   items in a row currently fires 10 webhooks. A 250ms debounce per project
   would coalesce them into one. Trade-off: adds latency to the first publish.
2. **Reverse index from Collection → referencing Pages.** Today, any collection
   change triggers a full-site purge. With a reverse index, only the affected
   pages get purged. ~2 hours of work, meaningful CDN hit-rate improvement
   for content-heavy sites.
3. **Webhook delivery log in CMS dashboard.** "Last 50 events with status."
   Useful for debugging silent failures.
4. **Multi-receiver fan-out.** Some projects might want to notify multiple
   hosts (e.g. staging + prod). Currently one URL per project.
5. **Native integrations for popular hosts.** Click "Connect to Vercel" → OAuth
   flow → automatic webhook URL + secret provisioning. Removes the setup
   friction. Considered future work.

---

## Cross-references

- `integration-guide/09-api-reference.md` — public API endpoints (will gain a
  "Caching" subsection in each endpoint after this lands)
- `integration-guide/13-editor-only-dom.md` — `?cms-preview=true` mode, which
  bypasses the cache via `isCmsPreviewMode()`
- `integration-guide/12-seo-metadata.md` — SEO settings update fires
  `pathsForSeoUpdate()` which is a full-site purge
- `lib/utils.ts` — `HTTP_CACHE_TTL`, `getPublicApiHeaders`, `isCmsPreviewMode`,
  `generateEtag`
- `prisma/schema.prisma` — `Project.cdnSettings` (new)
