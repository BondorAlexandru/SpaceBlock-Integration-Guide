# 17 · Visitor Insights (beacon, heat map, suggestions)

Every project has an **Insights** dashboard in the CMS (Overview, Traffic,
Pages, Behaviour, Suggestions, SEO, Content). The basic traffic numbers come
from the CMS's own API log and need **no setup**. The detailed metrics — which
pages visitors open inside your app, time on page, scroll depth, which
elements they click, dead/rage clicks and the click **heat map** — come from a
small first-party script that runs in the visitor's browser:
`spaceblock-insights.js` (about 6 KB, no third parties, no cookies).

## Install

**SDK sites (3.5+): nothing to do.** `SpaceBlock.init()` injects the script
automatically from the same origin as the SDK. Opt out per site with:

```js
window.SpaceBlock.init({ apiKey, projectId, insights: false })
```

**Sites that route through the render endpoint** (`/api/public/render/…`) get
the tag injected into the HTML shell by the CMS. Nothing to do either.

**Any other site** (no SDK, no render endpoint): add one tag to every page,
anywhere in `<head>` or before `</body>`:

```html
<script async src="https://YOUR-CMS-DOMAIN/spaceblock-insights.js"
        data-project="YOUR_PROJECT_ID"></script>
```

`data-project` is the project id (not the API key — the script runs in public
browsers). `data-api` overrides the CMS origin; by default it is derived from
the script's own `src`.

## What it records

| Event | When | Fields |
| --- | --- | --- |
| Pageview | page load + every client-side route change (`pushState`, `replaceState`, `popstate`) | path (no query/hash), referrer (external only, first page of the session), viewport size |
| Engagement | page hidden, unloaded, or navigated away | active (visible) milliseconds, max scroll depth % |
| Click | on `a`, `button`, inputs, `[role=button|link|tab|menuitem]`, `[onclick]`, `[data-track]` | element selector, visible label (max 80 chars), href, external flag, position inside the element (0-1) and in the document (0-1) |
| Dead click | click on an inert element with no DOM change and no navigation within 700 ms | same as click, `dead: true` |
| Rage click | 3+ clicks within 1 s on the same spot | same as click, `rage: true` |

Caps: 60 clicks per pageview, 25 events per batch, 32 KB per request.

**Never recorded:** cookies, form values, typed text, page content, IP
addresses. Visitor identity is a daily-rotating server-side hash (see
`lib/analytics/visitor.ts`), the same one the API log uses.

**Silent when:** inside the CMS Visual Editor (any iframe), `?cms-preview=true`,
Do-Not-Track / Global Privacy Control, automation (`navigator.webdriver`, which
covers Playwright prerendering), and on `localhost` / `127.0.0.1`. To test
locally add `data-debug` to the script tag (or run the site on another
hostname).

## Stable selectors — make your clicks readable

Clicks are grouped by a CSS selector built from the clicked element. The
script prefers, in order: `[data-cms-id="…"]`, `[data-track="…"]`, `#id`,
then a `tag:nth-of-type` path (max 6 levels). Tailwind class soup is ignored
on purpose. For buttons that matter, add a stable hook so renames and
reorders don't split the numbers:

```html
<a href="/contact" data-track="hero-cta">Book a call</a>
```

The label shown in the dashboard is `aria-label` → `title` → inner text →
`img[alt]`, trimmed to 80 characters.

## Heat map mode

The Behaviour tab opens your live page in an iframe with `?sb-heatmap=1`. In
that mode the script records nothing; it answers `postMessage` requests from
the CMS with the current bounding boxes of the clicked elements and
`preventDefault`s every click so the viewer can't navigate. Each stored click
is re-projected onto the element's **current** position, so the overlay stays
correct for responsive layouts and for a phone-width frame.

Requirements:

- the project's **Domain** (Settings) must point at the live site, and the
  site must allow being framed by the CMS (the Visual Editor already needs
  this);
- the page must load the script (SDK 3.5+, render endpoint, or the tag above).
  If it doesn't, the viewer shows "The page didn't answer" and falls back to
  approximate document-relative positions.
- the SDK treats `?sb-heatmap=1` like a normal public visit (no editor mode),
  so pages render exactly as visitors see them.

## Ingest endpoint

`POST /api/public/analytics/events` — body is JSON sent as `text/plain`
(so `sendBeacon` works without a preflight):

```json
{ "p": "<projectId>", "e": [
  { "t": "pv", "id": "<pageviewId>", "s": "<sessionId>", "path": "/pricing", "ref": "https://…", "vw": 1440, "vh": 900 },
  { "t": "en", "id": "<pageviewId>", "dur": 12400, "sd": 85 },
  { "t": "ck", "id": "<pageviewId>", "path": "/pricing", "sel": "[data-track=\"hero-cta\"]", "tag": "a", "txt": "Book a call", "href": "https://…", "ext": false, "x": 0.4, "y": 0.5, "px": 0.5, "py": 0.12, "vw": 1440, "vh": 900 }
] }
```

Responses: `204` on success (also for DNT), `400` with zod details for a
malformed batch, `404` unknown project, `413` oversized, `429` rate-limited.
Bots are dropped server-side. You normally never call this yourself — the
script does.

## Consent, ad blockers and CSP

The script sets no cookies and stores nothing identifying (a per-tab
`sessionStorage` id links the pages of one visit and is gone when the tab
closes). Whether that needs a consent banner depends on your jurisdiction and
your own reading of it: if you gate analytics behind consent, initialise the
SDK with `insights: false` and append the script tag only after consent.

Ad blockers that block analytics may block `/spaceblock-insights.js`; those
visitors are simply not counted. If your site sets a Content-Security-Policy,
allow the CMS origin in `script-src` and `connect-src`.

## Once the beacon is live

Once a project receives a steady stream of beacon pageviews (20 or more in
24 hours), the CMS **stops** logging that project's content-API hits as
pageviews (they would double count). Earlier data is kept. Expect the "/" row
from the old API log to fade out and real per-page rows to take over within a
day. The two methods count slightly differently (the API log saw ad-blocked
visitors, the beacon sees client-side navigations), so a period-over-period
comparison that straddles the switch is not like-for-like.

## Suggestions tab

Rule-based, deterministic, no AI. Each suggestion is a threshold over the data
above (traffic drops, quick exits, low scroll, dead/rage click hotspots,
popular pages missing SEO fields, unvisited content, missing pages requested
by visitors, mobile engagement gap …) with a plain "what to do". Rules stay
silent on small samples (≥ 50 views per page, ≥ 200 views per site).
