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
