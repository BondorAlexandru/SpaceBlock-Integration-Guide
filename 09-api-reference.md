# API Reference

Complete API documentation for SpaceBlock CMS.

## Authentication

### API Key Authentication (Public Endpoints)

Public endpoints require your project's API key. Pass it via:

```javascript
// Query parameter
fetch('/api/public/content?apiKey=YOUR_API_KEY')

// OR request header (recommended)
fetch('/api/public/content', {
  headers: {
    'x-api-key': 'YOUR_API_KEY'
  }
})
```

### Session Authentication (Dashboard Endpoints)

Dashboard endpoints require a valid session from NextAuth. These are used internally by the SpaceBlock dashboard.

## Response Format

### Success Response

```json
{
  "data": { ... }
}
```

### Error Response

```json
{
  "error": "Error message description"
}
```

### HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 304 | Not Modified (cached response valid) |
| 400 | Bad Request - Invalid parameters |
| 401 | Unauthorized - Missing or invalid API key |
| 404 | Not Found - Resource doesn't exist |
| 500 | Internal Server Error |

## Caching & ETags

Public APIs support conditional requests for efficient caching:

```javascript
// First request - returns data with ETag
const response = await fetch('/api/public/content?apiKey=YOUR_KEY')
const etag = response.headers.get('ETag')

// Subsequent requests - include If-None-Match
const cachedResponse = await fetch('/api/public/content?apiKey=YOUR_KEY', {
  headers: {
    'If-None-Match': etag
  }
})

if (cachedResponse.status === 304) {
  // Use cached data - no body returned
}
```

### Preview Mode

Add `preview=true` to bypass caching in the Visual Editor:

```javascript
fetch('/api/public/content?apiKey=YOUR_KEY&preview=true')
```

---

## Public API Endpoints

These endpoints are accessible with your API key and are designed for frontend use.

### Content

#### Get All Content

Fetches all content blocks and page elements for your project.

```
GET /api/public/content
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| apiKey | string | Yes | Your project API key |
| cmsId | string | No | Filter by specific content ID |

**Response:**

```json
{
  "content": [
    {
      "cmsId": "hero-title",
      "type": "text",
      "content": { "text": "Welcome to Our Site" },
      "updatedAt": "2024-01-15T10:30:00Z"
    },
    {
      "cmsId": "hero-image",
      "type": "image",
      "content": { "url": "https://cdn.example.com/hero.jpg" },
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  ]
}
```

**Example:**

```javascript
// Fetch all content
const response = await fetch(
  `/api/public/content?apiKey=${API_KEY}`
)
const { content } = await response.json()

// Create lookup map
const contentMap = {}
content.forEach(item => {
  contentMap[item.cmsId] = item.content
})

// Use in your app
const heroTitle = contentMap['hero-title']?.text
```

#### Get Specific Content

```javascript
const response = await fetch(
  `/api/public/content?apiKey=${API_KEY}&cmsId=hero-title`
)
const data = await response.json()
// Returns single content object instead of array
```

---

### Blog

#### List Blog Posts

Fetches published blog posts with pagination.

```
GET /api/public/blog
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| apiKey | string | Yes | Your project API key |
| limit | number | No | Max posts to return (default: 10, max: 100) |
| offset | number | No | Number of posts to skip (default: 0) |
| tag | string | No | Filter by tag |

**Response:**

```json
{
  "posts": [
    {
      "id": "post-123",
      "title": "Getting Started with SpaceBlock",
      "slug": "getting-started",
      "excerpt": "Learn how to integrate SpaceBlock...",
      "content": "<p>Full HTML content...</p>",
      "coverImage": "https://cdn.example.com/cover.jpg",
      "author": {
        "name": "John Doe",
        "avatar": "https://cdn.example.com/avatar.jpg"
      },
      "tags": ["tutorial", "getting-started"],
      "publishedAt": "2024-01-15T10:30:00Z",
      "updatedAt": "2024-01-15T10:30:00Z",
      "seoTitle": "Getting Started with SpaceBlock",
      "seoDescription": "A step-by-step guide to integrating SpaceBlock into your site.",
      "ogImage": "https://cdn.example.com/og/getting-started.png",
      "canonicalUrl": "https://mysite.com/blog/getting-started",
      "noindex": false,
      "seo": {
        "title": "Getting Started with SpaceBlock",
        "description": "A step-by-step guide to integrating SpaceBlock into your site.",
        "image": "https://cdn.example.com/og/getting-started.png",
        "canonical": "https://mysite.com/blog/getting-started",
        "type": "article",
        "noindex": false
      }
    }
  ],
  "total": 25,
  "hasMore": true
}
```

The flat `seoTitle`, `seoDescription`, `ogImage`, `canonicalUrl`, and `noindex`
fields are additive convenience mirrors of the nested `seo` object; the server
returns the raw override or `null` / `false` (no server-side fallback). See
[SEO & Social Metadata](./12-seo-metadata.md) for the recommended consumer
fallback chain.

**Example:**

```javascript
// Fetch latest 5 posts
const response = await fetch(
  `/api/public/blog?apiKey=${API_KEY}&limit=5`
)
const { posts, total, hasMore } = await response.json()

// Fetch posts by tag
const tutorialPosts = await fetch(
  `/api/public/blog?apiKey=${API_KEY}&tag=tutorial`
)

// Pagination
const page2 = await fetch(
  `/api/public/blog?apiKey=${API_KEY}&limit=10&offset=10`
)
```

#### Get Single Blog Post

Fetches a single blog post by slug.

```
GET /api/public/blog/{slug}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| apiKey | string | Yes | Your project API key |
| slug | path | Yes | Post URL slug |

**Response:**

```json
{
  "post": {
    "id": "post-123",
    "title": "Getting Started with SpaceBlock",
    "slug": "getting-started",
    "content": "<p>Full HTML content...</p>",
    "coverImage": "https://cdn.example.com/cover.jpg",
    "author": {
      "name": "John Doe",
      "avatar": "https://cdn.example.com/avatar.jpg"
    },
    "tags": ["tutorial"],
    "publishedAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-15T10:30:00Z",
    "seoTitle": "Getting Started with SpaceBlock",
    "seoDescription": "A step-by-step guide to integrating SpaceBlock into your site.",
    "ogImage": "https://cdn.example.com/og/getting-started.png",
    "canonicalUrl": "https://mysite.com/blog/getting-started",
    "noindex": false,
    "seo": {
      "title": "Getting Started with SpaceBlock",
      "description": "A step-by-step guide to integrating SpaceBlock into your site.",
      "image": "https://cdn.example.com/og/getting-started.png",
      "canonical": "https://mysite.com/blog/getting-started",
      "type": "article",
      "noindex": false
    }
  }
}
```

**Example:**

```javascript
const response = await fetch(
  `/api/public/blog/getting-started?apiKey=${API_KEY}`
)
const { post } = await response.json()
```

---

### Collections

#### List Collection Items

Fetches all items in a collection.

```
GET /api/public/collections/{collectionSlug}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| apiKey | string | Yes | Your project API key |
| collectionSlug | path | Yes | Collection URL slug |

**Response:**

```json
{
  "collection": {
    "name": "Team Members",
    "slug": "team",
    "description": "Our amazing team",
    "schema": [
      { "key": "role", "label": "Role", "type": "text" },
      { "key": "bio", "label": "Bio", "type": "richtext" }
    ]
  },
  "items": [
    {
      "id": "item-123",
      "title": "Jane Smith",
      "slug": "jane-smith",
      "image": "https://cdn.example.com/jane.jpg",
      "description": "CEO & Founder",
      "tags": ["leadership"],
      "data": {
        "role": "CEO",
        "bio": "<p>Jane founded the company in 2020...</p>"
      },
      "order": 0,
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  ]
}
```

**Example:**

```javascript
// Fetch team members
const response = await fetch(
  `/api/public/collections/team?apiKey=${API_KEY}`
)
const { collection, items } = await response.json()

items.forEach(member => {
  console.log(member.title, member.data.role)
})
```

#### Get Single Collection Item

```
GET /api/public/collections/{collectionSlug}/{itemSlug}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| apiKey | string | Yes | Your project API key |
| collectionSlug | path | Yes | Collection URL slug |
| itemSlug | path | Yes | Item URL slug |

**Response:**

```json
{
  "item": {
    "id": "item-123",
    "title": "Jane Smith",
    "slug": "jane-smith",
    "image": "https://cdn.example.com/jane.jpg",
    "description": "CEO & Founder",
    "data": {
      "role": "CEO",
      "bio": "<p>Jane founded the company in 2020...</p>"
    },
    "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```

---

### Pages

#### List All Pages

Fetches all published pages for a project.

```
GET /api/public/pages/{projectId}/list
```

**Response:**

```json
{
  "pages": [
    {
      "id": "page-123",
      "name": "Home",
      "slug": "home",
      "path": "/",
      "title": "Welcome - My Site",
      "description": "Homepage meta description",
      "createdAt": "2024-01-15T10:30:00Z",
      "updatedAt": "2024-01-16T09:00:00Z",
      "seoTitle": "Welcome - My Site",
      "seoDescription": "The official home of My Site.",
      "ogImage": "https://cdn.example.com/og/home.png",
      "canonicalUrl": "https://mysite.com/",
      "noindex": false
    },
    {
      "id": "page-456",
      "name": "About",
      "slug": "about",
      "path": "/about",
      "title": "About Us - My Site",
      "description": "Learn about our company",
      "createdAt": "2024-01-15T10:30:00Z",
      "updatedAt": "2024-01-15T10:30:00Z",
      "seoTitle": null,
      "seoDescription": null,
      "ogImage": null,
      "canonicalUrl": null,
      "noindex": false
    }
  ]
}
```

Each page in the list carries the flat `seoTitle`, `seoDescription`, `ogImage`,
`canonicalUrl`, `noindex`, and `updatedAt` fields (additive; all existing fields
unchanged). The list does **not** include the nested `seo` object — use the
detail endpoint below for that. Flat fields return the raw override or
`null` / `false`; the server applies no fallback.

#### Get Page with Elements

Fetches a page and all its elements.

```
GET /api/public/pages/{projectId}/{pageSlug}
```

**Preview mode:** Unpublished (draft) pages return `404` on this endpoint.
To fetch a draft (e.g. inside the Visual Editor), put the request in preview
mode using any of:

- `?cms-preview=true` query parameter (recommended — forward this from your
  `publicUrl()` builder when the current page URL has `cms-preview=true`)
- `X-CMS-Preview: true` or `X-Spaceblock-Preview: true` request header
- A same-origin `Referer` containing `cms-preview=true`

Preview responses bypass the cache and are returned with no-cache headers.

**Response:**

```json
{
  "page": {
    "id": "page-123",
    "name": "Home",
    "slug": "home",
    "path": "/",
    "title": "Welcome - My Site",
    "description": "Homepage meta description",
    "updatedAt": "2024-01-16T09:00:00Z",
    "seoTitle": "Welcome - My Site",
    "seoDescription": "The official home of My Site.",
    "ogImage": "https://cdn.example.com/og/home.png",
    "canonicalUrl": "https://mysite.com/",
    "noindex": false,
    "seo": {
      "title": "Welcome - My Site",
      "description": "The official home of My Site.",
      "image": "https://cdn.example.com/og/home.png",
      "canonical": "https://mysite.com/",
      "type": "website",
      "noindex": false
    }
  },
  "elements": [
    {
      "id": "elem-1",
      "elementId": "hero-section-abc123",
      "type": "hero",
      "templateName": "Hero Section",
      "zone": "main",
      "order": 0,
      "content": {
        "hero-section-abc123-title": "Welcome",
        "hero-section-abc123-subtitle": "Start building today"
      }
    }
  ]
}
```

#### Server-side Render (SEO meta injection)

Returns the consumer's `index.html` shell with per-slug SEO/social meta tags
injected, so social crawlers (which don't run JavaScript) get rich link
previews. This is the only endpoint that returns `text/html` rather than JSON.

```
GET /api/public/render/{projectId}/{slug}        → page by slug
GET /api/public/render/{projectId}/blog/{slug}   → blog post by slug
GET /api/public/render/{projectId}               → home page (empty slug)
```

- Replaces the block between `<!-- seo:start -->` and `<!-- seo:end -->` in the
  shell with the resolved `<title>`, `<meta>`, and `<link rel="canonical">`
  tags. The SPA still boots normally for human visitors.
- **Unknown or unpublished slug → HTTP `404`** with the shell and
  `<meta name="robots" content="noindex,nofollow">` injected (never a 500).
- `?cms-preview=true` resolves unpublished/draft content and responds with
  `no-store`.
- `og:image` / `og:url` / canonical are always absolute. The shell is fetched
  from the project's configured **Site URL** and cached ~60s.

See [SEO & Social Metadata](./12-seo-metadata.md) for the full guide, the exact
tag block, fallback rules, and host-rewrite setup.

#### Sitemap

Returns a flat list of every published page and blog post so you can render your
own `sitemap.xml`. It's a convenience bundle of the two list endpoints.

```
GET /api/public/pages/{projectId}/sitemap
```

Returns **JSON, not XML**: the CMS doesn't know your site's absolute origin, so
the consumer joins each `path` to its own origin and renders the XML.

**Response:**

```json
{
  "entries": [
    { "type": "page", "path": "/", "slug": "home", "updatedAt": "2024-01-16T09:00:00Z", "noindex": false },
    { "type": "blog", "path": "/blog/getting-started", "slug": "getting-started", "updatedAt": "2024-01-15T10:30:00Z", "noindex": false }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| type | `"page"` \| `"blog"` | Whether the entry is a page or a blog post |
| path | string | Path relative to your site origin |
| slug | string | The page/post slug |
| updatedAt | string | ISO 8601 last-modified timestamp (use for `<lastmod>`) |
| noindex | boolean | `true` → exclude from the sitemap / mark noindex |

---

### Global Elements

#### List Global Elements

Fetches all active global elements (navbar, footer, etc.) for a project.

```
GET /api/public/global-elements/{projectId}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| apiKey | string | Yes | Your project API key |
| type | string | No | Filter by type: "navbar", "footer", "sidebar", "banner" |

**Response:**

```json
{
  "elements": [
    {
      "id": "elem-123",
      "elementId": "navbar",
      "type": "navbar",
      "content": {
        "logo": "/media/logo.svg",
        "logoAlt": "My Site",
        "items": [
          {
            "label": "Programs",
            "children": [
              { "label": "Fellowship", "path": "/programs/fellowship" },
              { "label": "Academic", "path": "/programs/academic" }
            ]
          },
          { "label": "About", "path": "/about" }
        ]
      },
      "order": 0,
      "isActive": true
    },
    {
      "id": "elem-456",
      "elementId": "footer",
      "type": "footer",
      "content": {
        "columns": [
          {
            "title": "Company",
            "links": [
              { "label": "About", "path": "/about" },
              { "label": "Careers", "path": "/careers" }
            ]
          }
        ],
        "copyright": "© 2026 My Company",
        "socialLinks": [
          { "label": "Twitter", "url": "https://twitter.com/mycompany" }
        ]
      },
      "order": 1,
      "isActive": true
    }
  ]
}
```

**Example:**

```javascript
// Fetch all global elements
const response = await fetch(
  `/api/public/global-elements/${PROJECT_ID}?apiKey=${API_KEY}`
)
const { elements } = await response.json()

// Get navbar content
const navbar = elements.find(el => el.type === 'navbar')
const navItems = navbar?.content?.items || []

// Items can be simple links or dropdowns with children
navItems.forEach(item => {
  if (item.children) {
    // Dropdown menu with sub-items
    console.log(`${item.label} (dropdown):`, item.children)
  } else {
    // Simple link
    console.log(`${item.label}: ${item.path}`)
  }
})

// Get footer content
const footer = elements.find(el => el.type === 'footer')
const footerColumns = footer?.content?.columns || []
```

#### Filter by Type

```javascript
// Fetch only navbar
const response = await fetch(
  `/api/public/global-elements/${PROJECT_ID}?apiKey=${API_KEY}&type=navbar`
)
const { elements } = await response.json()
const navbarContent = elements[0]?.content
```

---

## Authenticated API Endpoints

These endpoints require session authentication and are used by the SpaceBlock dashboard.

### Projects

#### List Projects

```
GET /api/projects
```

**Response:**

```json
{
  "projects": [
    {
      "id": "proj-123",
      "name": "My Website",
      "domain": "https://mysite.com",
      "devDomain": "http://localhost:5173",
      "apiKey": "sb_xxxx...",
      "_count": {
        "BlogPost": 10,
        "Collection": 3,
        "Media": 50,
        "Page": 5
      }
    }
  ]
}
```

#### Get Project

```
GET /api/projects/{projectId}
```

#### Create Project

```
POST /api/projects
```

**Body:**

```json
{
  "name": "My Website",
  "domain": "https://mysite.com"
}
```

#### Update Project

```
PATCH /api/projects/{projectId}
```

**Body:**

```json
{
  "name": "Updated Name",
  "domain": "https://newdomain.com",
  "devDomain": "http://localhost:3000",
  "bunnyStorageZone": "my-zone",
  "bunnyApiKey": "xxxx",
  "bunnyCdnUrl": "https://my-zone.b-cdn.net"
}
```

#### Delete Project

```
DELETE /api/projects/{projectId}
```

---

### Media

#### List Media

```
GET /api/projects/{projectId}/media/list
```

**Query Parameters:**

| Name | Type | Description |
|------|------|-------------|
| type | string | Filter by type: "all", "image", "video", "document" |

**Response:**

```json
{
  "files": [
    {
      "name": "hero-image.jpg",
      "url": "https://cdn.example.com/proj-123/hero-image.jpg",
      "size": 245000,
      "type": "image",
      "lastModified": "2024-01-15T10:30:00Z"
    }
  ],
  "total": 50
}
```

#### Upload Media

```
POST /api/projects/{projectId}/media
```

**Body:** `multipart/form-data` with `file` field

**Response:**

```json
{
  "media": {
    "id": "media-123",
    "filename": "uploaded-image.jpg",
    "url": "https://cdn.example.com/proj-123/uploaded-image.jpg",
    "size": 150000,
    "mimeType": "image/jpeg"
  }
}
```

#### Delete Media

```
DELETE /api/projects/{projectId}/media/{mediaId}
```

---

### Blog Management

#### List All Posts (including drafts)

```
GET /api/projects/{projectId}/blog
```

#### Create Post

```
POST /api/projects/{projectId}/blog
```

**Body:**

```json
{
  "title": "My New Post",
  "slug": "my-new-post",
  "content": "<p>Post content...</p>",
  "excerpt": "Brief summary",
  "coverImage": "https://cdn.example.com/cover.jpg",
  "tags": ["news", "update"],
  "status": "published"
}
```

#### Update Post

```
PATCH /api/projects/{projectId}/blog/{postId}
```

#### Delete Post

```
DELETE /api/projects/{projectId}/blog/{postId}
```

---

### Collections Management

#### List Collections

```
GET /api/projects/{projectId}/collections
```

#### Create Collection

```
POST /api/projects/{projectId}/collections
```

**Body:**

```json
{
  "name": "Products",
  "slug": "products",
  "description": "Our product catalog",
  "schema": [
    { "key": "price", "label": "Price", "type": "text" },
    { "key": "features", "label": "Features", "type": "richtext" }
  ]
}
```

#### Update Collection

```
PATCH /api/projects/{projectId}/collections/{collectionId}
```

#### Delete Collection

```
DELETE /api/projects/{projectId}/collections/{collectionId}
```

#### Collection Items

```
GET    /api/projects/{projectId}/collections/{collectionId}/items
POST   /api/projects/{projectId}/collections/{collectionId}/items
PATCH  /api/projects/{projectId}/collections/{collectionId}/items/{itemId}
DELETE /api/projects/{projectId}/collections/{collectionId}/items/{itemId}
```

---

### Pages Management

#### List Pages

```
GET /api/projects/{projectId}/pages
```

#### Create Page

```
POST /api/projects/{projectId}/pages
```

**Body:**

```json
{
  "name": "Contact",
  "slug": "contact",
  "path": "/contact",
  "title": "Contact Us - My Site",
  "description": "Get in touch with our team"
}
```

#### Update Page

```
PATCH /api/projects/{projectId}/pages/{pageId}
```

#### Delete Page

```
DELETE /api/projects/{projectId}/pages/{pageId}
```

#### Page Elements

```
GET    /api/projects/{projectId}/pages/{pageId}/elements
POST   /api/projects/{projectId}/pages/{pageId}/elements
PATCH  /api/projects/{projectId}/pages/{pageId}/elements/{elementId}
DELETE /api/projects/{projectId}/pages/{pageId}/elements/{elementId}
```

#### Reorder Elements

```
POST /api/projects/{projectId}/pages/{pageId}/elements/reorder
```

**Body:**

```json
{
  "elementIds": ["elem-3", "elem-1", "elem-2"]
}
```

---

### Global Elements Management

#### List Global Elements

```
GET /api/projects/{projectId}/global-elements
```

**Response:**

```json
{
  "elements": [
    {
      "id": "elem-123",
      "elementId": "navbar",
      "type": "navbar",
      "content": { ... },
      "isActive": true
    }
  ]
}
```

#### Create Global Element

```
POST /api/projects/{projectId}/global-elements
```

**Body:**

```json
{
  "elementId": "navbar",
  "type": "navbar",
  "content": {
    "logo": "/media/logo.svg",
    "logoAlt": "My Site",
    "items": [
      { "label": "Home", "path": "/" },
      {
        "label": "Programs",
        "path": "/programs",
        "children": [
          { "label": "Fellowship", "path": "/programs/fellowship" },
          { "label": "Grants", "path": "/programs/grants" }
        ]
      },
      { "label": "About", "path": "/about" }
    ]
  }
}
```

#### Get Single Global Element

```
GET /api/projects/{projectId}/global-elements/{elementId}
```

#### Update Global Element

```
PATCH /api/projects/{projectId}/global-elements/{elementId}
```

**Body:**

```json
{
  "content": {
    "logo": "/media/new-logo.svg",
    "logoAlt": "Updated Site Name",
    "items": [
      { "label": "Home", "path": "/" },
      {
        "label": "Services",
        "path": "/services",
        "children": [
          { "label": "Consulting", "path": "/services/consulting" }
        ]
      }
    ]
  },
  "isActive": true
}
```

#### Delete Global Element

```
DELETE /api/projects/{projectId}/global-elements/{elementId}
```

---

## SDK Methods

The SpaceBlock SDK provides convenient methods for common operations:

```javascript
// Initialize
SpaceBlock.init({
  apiKey: 'YOUR_API_KEY',
  projectId: 'YOUR_PROJECT_ID'
})

// Fetch all content
const content = await SpaceBlock.getContent()

// Fetch specific content
const heroTitle = await SpaceBlock.getContent('hero-title')

// Apply content to DOM
SpaceBlock.loadContent() // Auto-applies to data-cms-id elements
```

## Rate Limits

- Public API: 1000 requests per minute per API key
- Authenticated API: 100 requests per minute per user

## CORS

Public API endpoints support CORS from any origin. Include the appropriate headers:

```javascript
fetch('/api/public/content?apiKey=YOUR_KEY', {
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'YOUR_KEY'
  }
})
```

## Next Steps

- [Setup Guide](./01-setup.md) - Get started with SDK installation
- [Visual Editor](./02-visual-editor.md) - Enable visual editing
- [Global Elements](./10-global-elements.md) - Navbar, footer, and site-wide components
