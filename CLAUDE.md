# CLAUDE.md — SpaceBlock integration rules

> Binding rules for any project that integrates the SpaceBlock CMS.
> **Copy this folder into every new integration project** (e.g. as
> `docs/spaceblock/`) and add one line to the project's root `CLAUDE.md`:
>
> ```
> SpaceBlock integration: ALWAYS follow docs/spaceblock/CLAUDE.md and consult
> the numbered guides in that folder before writing any CMS-related code.
> ```
>
> The numbered guides (`01-setup.md` … `16-chatbot-widget.md`) are the
> authoritative reference — these rules tell you WHEN a guide is mandatory
> reading and what must never regress. When a rule and site code disagree,
> surface it; don't silently pick one.

## When to read which guide

| Touching | Read first |
| --- | --- |
| SDK init, env vars, app bootstrap | `01-setup.md` |
| Anything inside the visual-editor preview (`?cms-preview=true`) | `02-visual-editor.md`, `13-editor-only-dom.md` |
| Adding/renaming/removing a section component | `03-templates.md` |
| `data-cms-id` markup, editable text/images | `04-content-blocks.md` |
| Routes/slugs, dynamic pages | `05-pages.md` |
| Blog, collections, media, global navbar/footer | `06`–`08`, `10` |
| Fetch helpers, API params | `09-api-reference.md` |
| Numbered slot lists with reordering | `11-slot-templates.md` |
| SEO/OG tags | `12-seo-metadata.md` |
| Chatbot embed or a custom chat UI | `16-chatbot-widget.md` |

## Absolute rules

### R1 — Initialise the SDK exactly as documented, from env vars

`window.SpaceBlock.init({ apiKey, projectId, autoLoad: false, … })` per
`01-setup.md`, with `apiKey`/`projectId` coming from environment variables
(e.g. `VITE_SPACEBLOCK_API_KEY`, `VITE_SPACEBLOCK_PROJECT_ID`) that are **set
in every deploy target**. NEVER ship a build where these resolve to
`undefined` — the site will silently render only hardcoded fallbacks and no
published content. Keep an `.env.example` listing both.

### R2 — Every editable thing gets a stable `data-cms-id`; fallbacks must work

Mark up editable text/images with a unique `data-cms-id`
(`page-section-field` convention) and the right `data-cms-type`. Bake real
fallback content into the markup — and if a fallback references a static
asset (`/images/…`), that file MUST exist in the deploy. A broken fallback
path renders a broken site everywhere the CMS hasn't overridden it yet.

### R3 — Never rename or delete a `data-cms-id` or `templateId` casually

CMS content is keyed to these strings. Renaming a `data-cms-id` orphans the
saved content; renaming a section's `templateId` orphans every element
already placed on pages (they keep content but lose their template link).
If a rename is genuinely needed, flag it so content can be migrated in the
CMS first.

### R4 — Removing a section component from the code removes it from the CMS

Templates are detected from the site's registration (`03-templates.md`).
Deleting or un-registering a section means it disappears from the CMS "Add
Element" library on the next refresh — that is intended. Never leave dead
template registrations around "just in case"; archive/delete deliberately.

### R5 — Page paths must match the CMS

A CMS page with path `/about` must be served by the site at `/about`. When
creating the site's home page, make sure the CMS page path is `/` (not
`/home`) unless the site actually serves `/home`. A mismatch renders a 404
inside the visual editor's preview iframe.

### R6 — Keep the DOM stable between preview and production

The visual editor overlays the real site. Don't branch layout on
`?cms-preview=true` except as documented in `13-editor-only-dom.md`
(editor-only field markers). A DOM that differs between preview and
production makes click-to-edit target the wrong nodes.

### R7 — Only the public API, only the public key

Consume content through the documented public endpoints/SDK
(`09-api-reference.md`). NEVER call the CMS's session-authenticated project
routes from a site, and never embed anything but the project's public API
key client-side. Secrets (webhook signing keys, etc.) are server-only.

### R8 — Don't fight the SDK's content application

The SDK owns filling `data-cms-id` nodes. Don't re-render or overwrite those
nodes after `load()` outside the documented patterns (React fallbacks via
props/state are fine — see `01-setup.md`'s `hideUntilLoaded` notes). If
content "flickers back", the component is re-rendering over applied content;
fix the component, don't poll the API.

### R9 — Chatbot: one embed, documented modes only

One `chatbot-widget.js` script tag per page, configured via `data-*`
attributes (`16-chatbot-widget.md`). For custom looks use the documented
design modes (theme / custom CSS / headless with
`window.SpaceBlockChatbot.createClient()` or the raw SSE contract) — never
scrape or spoof the widget's internals.

### R10 — When unsure, the guide wins over guesswork

Before inventing an endpoint, SDK option, parameter, or field name: grep
these guide files. If the answer isn't in the guide or the code, ask — do
not fabricate a contract. If the guide and observed behavior disagree,
report the discrepancy upstream (the CMS repo mirrors this guide and keeps
both in sync).
