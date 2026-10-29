# 16 · Chatbot Widget

Add an AI chat assistant to your site. The assistant answers **only** from that
site's own content (pages you publish plus files/text added in the CMS). It's an
admin-only feature, configured per project under the **Chatbot** tab.

## Install

Paste this once, just before `</body>` on your site (every page):

```html
<script src="https://YOUR-CMS-DOMAIN/chatbot-widget.js"
        data-website-id="YOUR_WEBSITE_ID"
        async></script>
```

- `YOUR-CMS-DOMAIN` — the origin where the CMS is hosted.
- `YOUR_WEBSITE_ID` — the **Website ID** shown on the project's Chatbot page
  (a GUID; it's the public tenant identifier and safe to expose).

Copy the exact snippet (with both values filled in) from **Chatbot → Preview →
Add to your site**.

## Modes: corner vs. full-page

Two presentations, chosen by the theme's **Position / mode** setting (Chatbot →
Preview) or overridden per embed:

- **Corner** — a floating launcher + panel (the default).
- **Full page** — a page-like takeover (own header + "GO BACK", centred column,
  large composer). The widget **never** opens full-page automatically — put it on
  a dedicated route and open it there (below).

Embed attributes:

| Attribute | Effect |
| --- | --- |
| `data-mode="fullscreen"` \| `"corner"` | Override the theme's default mode for this embed. |
| `data-launcher="false"` | Hide the floating launcher — the chat is then opened only via the JS API. |

### JS API

Once loaded (published chatbot), the widget exposes:

```js
window.SpaceBlockChatbot.open('fullscreen' | 'corner')  // open (defaults to the configured mode)
window.SpaceBlockChatbot.close()
window.SpaceBlockChatbot.toggle('fullscreen' | 'corner')
```

The global is set asynchronously (after the config request resolves), so poll a
few times if you call it very early.

### Full-page chat as its own route (recommended)

Install the widget site-wide with `data-launcher="false"`, then add a dedicated
route (e.g. `/chat`) that opens the full-page chat on mount and closes it on
unmount. The chat's **"← GO BACK"** navigates back and closes it.

```jsx
// /chat page (React example)
useEffect(() => {
  let n = 0
  const go = () => window.SpaceBlockChatbot
    ? window.SpaceBlockChatbot.open('fullscreen')
    : n++ < 40 && setTimeout(go, 100)
  go()
  return () => window.SpaceBlockChatbot?.close()
}, [])
```

Link to it from your nav (`<a href="/chat">Chat</a>`). This keeps the rest of the
site usable — the full-page chat only appears on that route.

## Behaviour

- The script is self-contained: no dependencies, and it renders inside a **Shadow
  DOM**, so your site's CSS can't leak into the widget or vice-versa.
- It renders a floating launcher in the corner; clicking it opens the chat panel.
- **Appearance follows the theme** you set in **Chatbot → Preview** (colors,
  fonts, sizes, corner radius, launcher position, header title, placeholder,
  welcome text). Change the theme and re-save — no code change needed.
- **Nothing renders until the chatbot is Published.** While it's a Draft the
  script loads but shows no launcher, so you can stage it safely. Publish from the
  Chatbot page to go live.
- **Page context.** Every message also sends the address of the page the visitor
  is on (`window.location.href`, read at send time, so client-side route changes
  count) and their language (`navigator.language`). Only the address is sent —
  never page content — and before sending, the `#fragment` and any query
  parameter whose name looks like a credential (`token`, `key`, `code`, `auth`,
  `session`, `email`, …) are removed; product parameters like `?id=` stay. The
  assistant uses the page only when **page awareness** is on.
- **Product cards (e-commerce bots).** When an answer recommends products, the
  widget shows up to 6 cards under it — image, name, price, an "Out of stock"
  badge — each linking to the product page (opens in a new tab, so the chat stays
  open). Only `http(s)` links and images are ever rendered.

## E-commerce bots: website processing & page awareness

Set this up in **Chatbot → Settings → Website & shop**, then **Chatbot → Website**:

1. **Chatbot type** — *General* or *E-commerce*. E-commerce adds a product
   catalogue and lets answers carry product cards. Delivery/returns/about answers
   work for both.
2. **Website address** — your public site (e.g. `https://shop.com`).
3. **Process website** (Website tab) — reads every public page into the
   assistant's knowledge and, for shops, builds the product catalogue. Progress
   shows live; "Last processed" is the moment prices and stock in answers are
   from, so **Reprocess** after catalogue changes. Pages found appear under
   Knowledge as "Website page"; uploaded and pasted sources are never touched.
4. **The bot knows which page the visitor is on** — enables page awareness
   ("Do you have this in blue?" on a product page). It uses the processed pages,
   so process the website first.
5. **Site language / English fallback** — the language is detected during
   processing if left empty; the English fallback message is shown to
   English-speaking visitors when the assistant can't answer.

No code change is needed on your site: the same embed script sends the page
context and renders product cards.

## Fully custom design (advanced)

If the theme controls aren't enough, switch **Chatbot → Preview → Design mode**
to **"Fully custom CSS"** (or go fully **Headless** — see the next section). The widget then IGNORES every visual theme setting
(colors, fonts, radii, sizes) and instead:

1. renders a neutral **structural skeleton** — plain, functional layout rules on
   stable class names, and
2. injects your **Custom CSS** verbatim into its shadow root, *after* the
   skeleton — so plain class selectors override everything, no `!important`
   needed.

Position / mode (corner side, full-page) and the copy fields (header title,
placeholder, welcome text) still apply in custom mode.

### Class reference

| Class | Element |
| --- | --- |
| `.cb-wrap` | fixed corner container (launcher + panel) |
| `.cb-launcher` | floating open button |
| `.cb-panel` | corner chat panel |
| `.cb-header` / `.cb-title` / `.cb-close` | panel header, its title, close button |
| `.cb-list` | scrollable message list (corner and full-page) |
| `.cb-row`, `.cb-row-user`, `.cb-row-assistant` | message row (alignment) |
| `.cb-msg`, `.cb-msg-user`, `.cb-msg-assistant` | message bubble |
| `.cb-note` | small note under a message (e.g. errors) |
| `.cb-form` / `.cb-input` / `.cb-send` | corner composer, its input, send button |
| `.cb-dots` | typing indicator |
| `.cb-fs` | full-page overlay |
| `.cb-fs-nav` / `.cb-fs-bar` / `.cb-fs-logo` / `.cb-goback` | full-page navbar parts |
| `.cb-fs-col` / `.cb-fs-list` | full-page centred column / message list |
| `.cb-fs-form` / `.cb-fs-compose` | full-page composer and its input |
| `.cb-products` | row of product cards under an answer (e-commerce) |
| `.cb-product` (+ `.cb-product-oos` when out of stock) | one product card (an `<a>` when it has a link) |
| `.cb-product-img` / `.cb-product-body` | card image / text area |
| `.cb-product-name` / `.cb-product-price` / `.cb-product-badge` | card name, price, "Out of stock" badge |

### Example

```css
.cb-launcher { background: #0a3d2e; }
.cb-panel { width: 420px; height: 620px; border-radius: 0; border: 2px solid #0a3d2e; }
.cb-header { background: #0a3d2e; color: #fff; font-family: Georgia, serif; }
.cb-msg-user { background: #0a3d2e; color: #fff; border-radius: 2px; }
.cb-msg-assistant { background: #f4f1ea; border-radius: 2px; }
.cb-input { border-radius: 2px; border-color: #0a3d2e; }
.cb-send { background: #0a3d2e; border-radius: 2px; }
```

### Notes

- Your CSS lives **inside the widget's shadow root**: it can't affect the host
  page, and the host page's CSS can't affect the widget. `@import`/`@font-face`
  inside it load relative to your site as usual; fonts already loaded by the
  host page are **not** inherited automatically — declare `font-family` with
  fonts the page provides via `@font-face`.
- The skeleton keeps the widget functional with zero CSS, so build up from a
  working base rather than a blank screen.
- Capped at 20,000 characters.

## Headless — build the chat UI from scratch

Some projects need a design or conversation flow the widget can't express. Switch
**Chatbot → Preview → Design mode** to **"Headless"**: the embed script renders
**nothing** — no launcher, no panel, no styles — and your own code owns the whole
experience. Publishing still gates the API: a Draft chatbot answers nobody.

There are two ways to build on it:

### Option A — the JS client (keep the embed script)

The script exposes a UI-less client that handles streaming, SSE parsing and
conversation history (last 20 turns) for you:

```html
<script src="https://YOUR-CMS/chatbot-widget.js" data-website-id="<uuid>" async></script>
```

```js
const chat = window.SpaceBlockChatbot.createClient({
  onDelta: (text) => render(text),            // full assistant text so far (streams)
  onDone:  (text, history) => finalize(text), // stream finished
  onError: (message) => showError(message),   // rate limit / network / server error
  onProducts: (items) => renderCards(items),  // e-commerce product cards (up to 6, see below)
})

await chat.send("What do you offer?")  // resolves true on success, false on error
chat.stop()                            // abort the in-flight answer
chat.reset()                           // clear conversation history
chat.history()                         // [{ role: "user"|"assistant", content }]
chat.streaming()                       // boolean
```

`createClient()` is available in **every** design mode, so you can also mix it
with the rendered widget (e.g. a separate inline chat on one page). It sends the
page address and language with every message, like the widget.

### Option B — the raw HTTP contract (any stack, no script)

Skip the script entirely and call the two public endpoints yourself:

```
GET  /api/chatbot/{websiteId}/public-config
     → { published: boolean, displayName, theme } — render nothing unless published.

POST /api/chatbot/{websiteId}/chat
     Content-Type: application/json
     {
       "message": "Hi",
       "history": [{ "role": "user"|"assistant", "content": "…" }],
       "pageUrl": "https://shop.com/p/linen-shirt",   // optional: window.location.href
       "language": "en-GB"                            // optional: navigator.language
     }
     → Server-Sent Events stream
```

`pageUrl` is only the address of the visitor's current page (re-read it on
client-side route changes); it's ignored unless page awareness is enabled, and
page content must never be sent. Strip the `#fragment` and any credential-like
query parameters (tokens, codes, session ids, emails) before sending — the
address is passed on to the AI provider. `language` only picks the English fallback
message for failures that never reach the model.

The SSE stream emits events with a JSON `data` payload:

| Event | Payload | Meaning |
| --- | --- | --- |
| `delta` | `{ "text": "…" }` | Append to the current answer (token chunk). |
| `message` | `{ "text": "…" }` | Whole fallback message — the reply when the bot can't answer. |
| `block` | `{ "type": "products", "items": [ProductCard] }` | E-commerce product cards, after the last `delta`. **Ignore unknown `type`s.** |
| `done` | `{ "reason": "Completed" \| … }` | Always the last event. Any reason other than `Completed` means the `message` event carried the reply (`no_credentials`, `budget_exceeded`, `ProviderAuthFailed`, `ProviderUnavailable` — retry the last one). |
| `error` | `{ "error": "not_configured" }` | The chatbot isn't set up; the stream ends. |

A `ProductCard` is:

```json
{
  "id": "guid", "url": "https://shop.com/p/linen-shirt", "name": "Linen Shirt",
  "description": "Light and airy.", "imageUrl": "https://shop.com/img/linen.jpg",
  "price": 49.0, "currency": "EUR", "availability": "InStock",
  "category": "Shirts", "brand": "Brand", "language": "en"
}
```

Up to 6 items, in the order the answer mentions them. `description`, `imageUrl`,
`price`, `currency`, `availability` (`InStock` | `OutOfStock` | `PreOrder`),
`category` and `brand` can each be `null` — render the card without them. The
answer text never contains product links; the cards are the only links. Only
render `http(s)` URLs.

Notes for a from-scratch client:

- Send at most the **last 20 turns** as `history`; the server ignores more.
- A `429` response means the per-site/IP rate limit was hit — back off briefly.
- CORS is open on both endpoints; no API key exists client-side by design.
- Abort the fetch to cancel an answer (the widget uses `AbortController`).

## How it talks to the CMS

The widget never holds any API key. It calls two open (CORS-enabled) CMS
endpoints, keyed by the public Website ID:

| Endpoint | Purpose |
| --- | --- |
| `GET /api/chatbot/{websiteId}/public-config` | Published state + theme (no secrets). |
| `POST /api/chatbot/{websiteId}/chat` | Streams the answer over Server-Sent Events. |

All model credentials and the chatbot service key stay server-side in the CMS
backend. Rate limiting (per website + IP) is enforced upstream.

## Notes

- One snippet covers the whole site; don't add it more than once per page.
- Draft chatbots only respond inside the CMS **Preview** tab (an authenticated
  admin request) — public visitors get nothing until you publish.
- Removing the assistant: switch it back to Draft (instant), or remove the script
  tag from your site.
