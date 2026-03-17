# SpaceBlock Integration Guide

Welcome to the SpaceBlock CMS integration guide. This documentation covers integrating SpaceBlock into a React + Vite + TypeScript + Tailwind application.

## Recommended Project Structure

```
src/
  components/
    layout/
      Layout.tsx        ← page shell: Navbar + children + Footer + TemplateRegistry
      Navbar.tsx        ← CMS-driven nav with live preview
      Footer.tsx        ← CMS-driven footer with live preview
      index.ts          ← re-exports
    sections/
      HeroBanner.tsx    ← one file per template
      FeatureSection.tsx
      SimpleText.tsx
      RichText.tsx
  pages/
    DynamicPage.tsx     ← catch-all CMS page renderer
    BlogPostPage.tsx
    BlogListPage.tsx
    BlogPreviewPage.tsx
    HomePage.tsx
  lib/
    api.ts              ← all SpaceBlock fetch helpers
    types.ts            ← TypeScript interfaces
  App.tsx               ← SDK init + routing (wrapped in Layout)
  main.tsx
  index.css
index.html              ← SDK script loader
.env.example
```

## Table of Contents

1. [Setup Guide](./01-setup.md) — SDK installation and initial configuration
2. [Visual Editor](./02-visual-editor.md) — Integrate the visual editing experience
3. [Templates](./03-templates.md) — TemplateRegistry, section components, triple-detection
4. [Content Blocks](./04-content-blocks.md) — Simple `data-cms-id` editing
5. [Pages & Elements](./05-pages.md) — DynamicPage pattern, `find()` helper, caching
6. [Blog Integration](./06-blog.md) — Blog posts and block rendering
7. [Collections](./07-collections.md) — Structured data (team, products, testimonials)
8. [Media Library](./08-media.md) — Images and file management
9. [API Reference](./09-api-reference.md) — Complete API documentation
10. [Global Elements](./10-global-elements.md) — Navbar, footer, Layout component

## Quick Start

```html
<!-- index.html — load the SDK -->
<script>
  (function() {
    var isLocalhost = window.location.hostname === 'localhost';
    var script = document.createElement('script');
    script.src = isLocalhost
      ? 'http://localhost:3000/spaceblock-sdk.js'
      : 'https://www.spaceblock.app/spaceblock-sdk.js';
    script.defer = true;
    document.head.appendChild(script);
  })();
</script>
```

```tsx
// src/App.tsx — initialise once React has mounted
let sdkInitialised = false

export default function App() {
  useEffect(() => {
    if (sdkInitialised) return
    sdkInitialised = true

    function init() {
      if (window.SpaceBlock) {
        window.SpaceBlock.init({
          apiKey: import.meta.env.VITE_SPACEBLOCK_API_KEY,
          projectId: import.meta.env.VITE_SPACEBLOCK_PROJECT_ID,
          autoLoad: false,
          cacheEnabled: true,
          hideUntilLoaded: false,
        })
        requestAnimationFrame(() => requestAnimationFrame(() =>
          setTimeout(() => window.SpaceBlock?.load(), 200)
        ))
      } else {
        setTimeout(init, 50)
      }
    }
    init()
  }, [])

  return (
    <Layout>
      <Routes>...</Routes>
    </Layout>
  )
}
```

```
# .env.local
VITE_SPACEBLOCK_API_BASE=https://www.spaceblock.app
VITE_SPACEBLOCK_API_KEY=your-api-key
VITE_SPACEBLOCK_PROJECT_ID=your-project-id
```

## Getting Your Credentials

1. Log in to [SpaceBlock](https://www.spaceblock.app)
2. Create a new project or select an existing one
3. Go to **Project Settings** to find your API Key and Project ID
4. In **Project Settings → Domains**, set your Production and Development domains

## Support

For questions or issues, visit the [SpaceBlock Dashboard](https://www.spaceblock.app/dashboard).
