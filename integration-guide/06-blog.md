# Blog Integration Guide

SpaceBlock includes a full-featured blog system with posts, tags, authors, and structured content.

## Features

- Rich text content with TipTap editor
- Tags for categorization
- Author information
- Featured images
- Custom date fields
- Draft/Published status
- SEO metadata (title, description)

## Managing Blog Posts

### In the Dashboard

1. Go to your project in SpaceBlock
2. Click the "Blog" tab
3. Create, edit, or delete posts
4. Set status to "Published" when ready

### Post Structure

```typescript
interface BlogPost {
  id: string;
  title: string;
  slug: string;
  content: string;         // HTML content
  excerpt: string | null;  // Short description
  featuredImage: string | null;
  tags: string[];
  author: string | null;
  dateWritten: string | null;  // Custom date
  status: 'draft' | 'published';
  publishedAt: string | null;
  createdAt: string;
  updatedAt: string;
}
```

## Content Block Types

Blog posts use a block-based content system. Each block has a `type` and type-specific fields.

### Block Structure

```typescript
interface ContentBlock {
  id: string;
  type: BlockType;
  content?: string;      // Main content (HTML for richtext, URL for media)
  notes?: string;        // Captions, alt text, etc.
  level?: number;        // Heading level (1-6)
  url?: string;          // Link URL for CTA, download blocks
  image?: string;        // Image URL for author-details
  column1?: string;      // HTML for column text blocks
  column2?: string;      // HTML for column text blocks
  column3?: string;      // HTML for 3-column text blocks
  marginBottom?: number; // Bottom margin in pixels
}

type BlockType =
  | 'richtext'           // Rich text content (HTML)
  | 'heading'            // Heading (H1-H6)
  | 'image'              // Image with optional caption
  | 'video'              // Embedded video (YouTube, Vimeo, etc.)
  | 'audio'              // Audio file
  | 'quote'              // Blockquote
  | 'table'              // HTML table
  | 'download'           // Downloadable file link
  | 'cta'                // Call-to-action button
  | 'divider'            // Horizontal rule
  | 'two-column-text'    // Two columns of rich text
  | 'three-column-text'  // Three columns of rich text
  | 'intro-summary'      // Introduction/summary box
  | 'executive-summary'  // Executive summary box
  | 'author-details'     // Author bio with optional image
  | 'acknowledgements'   // Acknowledgements section
  | 'footnotes'          // Footnotes section
  | 'subblog';           // Embedded sub-article
```

### Block Type Details

#### Rich Text (`richtext`)
Standard rich text content with formatting.
```json
{
  "type": "richtext",
  "content": "<p>Paragraph with <strong>bold</strong> and <em>italic</em> text.</p>"
}
```

#### Heading (`heading`)
Section headings with configurable level.
```json
{
  "type": "heading",
  "content": "Section Title",
  "level": 2
}
```

#### Two Column Text (`two-column-text`)
Side-by-side columns of rich text content.
```json
{
  "type": "two-column-text",
  "column1": "<p>Left column content with <strong>formatting</strong>.</p>",
  "column2": "<p>Right column content.</p>"
}
```

#### Three Column Text (`three-column-text`)
Three columns of rich text content.
```json
{
  "type": "three-column-text",
  "column1": "<p>First column.</p>",
  "column2": "<p>Second column.</p>",
  "column3": "<p>Third column.</p>"
}
```

#### Image (`image`)
Image with optional caption.
```json
{
  "type": "image",
  "content": "https://example.com/image.jpg",
  "notes": "Image caption text"
}
```

#### Video (`video`)
Embedded video (supports YouTube, Vimeo embed URLs).
```json
{
  "type": "video",
  "content": "https://www.youtube.com/embed/VIDEO_ID"
}
```

#### Audio (`audio`)
Audio file player.
```json
{
  "type": "audio",
  "content": "https://example.com/audio.mp3"
}
```

#### Quote (`quote`)
Blockquote for citations or callouts.
```json
{
  "type": "quote",
  "content": "This is a quoted passage."
}
```

#### Call-to-Action (`cta`)
Button link for calls to action.
```json
{
  "type": "cta",
  "content": "Learn More",
  "url": "/programs/fellowship"
}
```

#### Download (`download`)
Downloadable file link.
```json
{
  "type": "download",
  "content": "Download PDF",
  "url": "https://example.com/file.pdf"
}
```

#### Divider (`divider`)
Horizontal rule separator.
```json
{
  "type": "divider"
}
```

#### Intro/Executive Summary (`intro-summary`, `executive-summary`)
Highlighted summary boxes.
```json
{
  "type": "intro-summary",
  "content": "<p>Key takeaways from this article...</p>"
}
```

#### Author Details (`author-details`)
Author bio section with optional image.
```json
{
  "type": "author-details",
  "content": "<p><strong>Jane Smith</strong> is a researcher at...</p>",
  "image": "https://example.com/author.jpg"
}
```

#### Acknowledgements (`acknowledgements`)
Acknowledgements section.
```json
{
  "type": "acknowledgements",
  "content": "<p>The author thanks...</p>"
}
```

#### Footnotes (`footnotes`)
Footnotes section.
```json
{
  "type": "footnotes",
  "content": "<p><sup>1</sup> Reference details...</p>"
}
```

#### Table (`table`)
HTML table content.
```json
{
  "type": "table",
  "content": "<table><tr><th>Header</th></tr><tr><td>Cell</td></tr></table>"
}
```

## Public API

### List All Posts

```javascript
const response = await fetch(
  'https://www.spaceblock.app/api/public/blog?projectId=YOUR_PROJECT_ID&apiKey=YOUR_API_KEY'
);
const data = await response.json();

// Returns published posts only
data.posts.forEach(post => {
  console.log(post.title, post.slug);
});
```

### Query Parameters

| Parameter | Description |
|-----------|-------------|
| `projectId` | Your project ID (required) |
| `apiKey` | Your API key (required) |
| `tag` | Filter by tag |
| `limit` | Number of posts (default: 50) |
| `offset` | Pagination offset |

### Filter by Tag

```javascript
const response = await fetch(
  'https://www.spaceblock.app/api/public/blog?projectId=PROJECT_ID&apiKey=API_KEY&tag=technology'
);
```

### Get Single Post

```javascript
const response = await fetch(
  'https://www.spaceblock.app/api/public/blog/my-post-slug?projectId=PROJECT_ID&apiKey=API_KEY'
);
const { post } = await response.json();

console.log(post.title);
console.log(post.content);  // HTML
console.log(post.author);
console.log(post.dateWritten);
```

## React Integration (Vite + React Router)

These examples match the patterns used in this project. API helpers live in `src/lib/api.ts` and are imported directly into page components.

### Blog List Page

```tsx
// src/pages/BlogListPage.tsx
import { useEffect, useState } from 'react'
import { Link, useSearchParams } from 'react-router-dom'
import { fetchBlogPosts } from '@/lib/api'
import type { BlogPost } from '@/lib/types'

export function BlogListPage() {
  const [posts, setPosts] = useState<BlogPost[]>([])
  const [loading, setLoading] = useState(true)
  const [searchParams, setSearchParams] = useSearchParams()
  const activeTag = searchParams.get('tag') ?? undefined

  useEffect(() => {
    let cancelled = false
    setLoading(true)

    fetchBlogPosts({ tag: activeTag })
      .then(data => { if (!cancelled) setPosts(data) })
      .catch(() => {})
      .finally(() => { if (!cancelled) setLoading(false) })

    return () => { cancelled = true }
  }, [activeTag])

  const allTags = [...new Set(posts.flatMap(p => p.tags))].sort()

  return (
    <main className="max-w-6xl mx-auto px-8 py-16">
      <h1 className="text-4xl font-bold mb-4">Blog</h1>

      {/* Tag filter */}
      {allTags.length > 0 && (
        <div className="flex flex-wrap gap-2 mb-10">
          <button onClick={() => setSearchParams({})}>All</button>
          {allTags.map(tag => (
            <button key={tag} onClick={() => setSearchParams({ tag })}>{tag}</button>
          ))}
        </div>
      )}

      {loading ? (
        <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
          {[1, 2, 3].map(i => (
            <div key={i} className="animate-pulse bg-gray-100 rounded-xl h-64" />
          ))}
        </div>
      ) : (
        <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
          {posts.map(post => (
            <article key={post.id} className="border rounded-xl overflow-hidden">
              {post.featuredImage && (
                <img src={post.featuredImage} alt={post.title} className="w-full h-48 object-cover" />
              )}
              <div className="p-6">
                <h2 className="font-semibold mb-2">
                  <Link to={`/blog/${post.slug}`}>{post.title}</Link>
                </h2>
                {post.excerpt && <p className="text-gray-600 text-sm mb-4">{post.excerpt}</p>}
                <div className="text-xs text-gray-400">
                  {post.author && <span>{post.author}</span>}
                  {(post.dateWritten ?? post.createdAt) && (
                    <span> · {new Date(post.dateWritten ?? post.createdAt).toLocaleDateString()}</span>
                  )}
                </div>
              </div>
            </article>
          ))}
        </div>
      )}
    </main>
  )
}
```

### Single Post Page

```tsx
// src/pages/BlogPostPage.tsx
import { useEffect, useState } from 'react'
import { Link, useParams } from 'react-router-dom'
import { fetchBlogPost } from '@/lib/api'
import { BlogContentBlock } from '@/components/BlogContentBlock'
import type { BlogBlock, BlogPost } from '@/lib/types'

export function BlogPostPage() {
  const { slug } = useParams<{ slug: string }>()
  const [post, setPost] = useState<BlogPost | null>(null)
  const [loading, setLoading] = useState(true)
  const [notFound, setNotFound] = useState(false)

  useEffect(() => {
    if (!slug) return
    let cancelled = false
    setLoading(true)

    fetchBlogPost(slug)
      .then(data => { if (!cancelled) setPost(data) })
      .catch(() => { if (!cancelled) setNotFound(true) })
      .finally(() => { if (!cancelled) setLoading(false) })

    return () => { cancelled = true }
  }, [slug])

  if (loading) return <div className="flex items-center justify-center min-h-[60vh]">Loading…</div>
  if (notFound || !post) return (
    <div className="flex flex-col items-center justify-center min-h-[60vh] gap-4 text-gray-500">
      <p>Post not found.</p>
      <Link to="/blog" className="text-blue-600 hover:underline text-sm">Back to blog</Link>
    </div>
  )

  const displayDate = post.dateWritten ?? post.createdAt
  const blocks = Array.isArray(post.content) ? post.content as BlogBlock[] : null
  const htmlContent = typeof post.content === 'string' ? post.content : null

  return (
    <article className="max-w-4xl mx-auto px-8 py-16">
      {post.featuredImage && (
        <img src={post.featuredImage} alt={post.title} className="w-full h-72 object-cover rounded-2xl mb-10" />
      )}

      <header className="mb-10">
        <h1 className="text-4xl font-bold mb-4">{post.title}</h1>
        {post.author && <p className="text-gray-500 text-sm">By {post.author}</p>}
        {displayDate && (
          <time className="text-gray-400 text-sm">
            {new Date(displayDate).toLocaleDateString(undefined, { dateStyle: 'long' })}
          </time>
        )}
        <div className="flex flex-wrap gap-2 mt-4">
          {post.tags.map(tag => (
            <Link key={tag} to={`/blog?tag=${encodeURIComponent(tag)}`}
              className="px-3 py-1 bg-gray-100 text-gray-700 rounded-full text-sm">{tag}</Link>
          ))}
        </div>
      </header>

      {/* Block-based or raw HTML content */}
      {blocks ? (
        <div>{blocks.map(block => <BlogContentBlock key={block.id} block={block} />)}</div>
      ) : htmlContent ? (
        <div className="prose prose-lg max-w-none" dangerouslySetInnerHTML={{ __html: htmlContent }} />
      ) : null}

      <div className="mt-16 pt-8 border-t">
        <Link to="/blog" className="text-blue-600 hover:underline text-sm">← Back to blog</Link>
      </div>
    </article>
  )
}
```

## Styling Blog Content

The blog content is HTML from a rich text editor. Add styles:

```css
/* Prose/blog content styles */
.prose {
  max-width: 65ch;
  line-height: 1.7;
}

.prose h1 { font-size: 2em; font-weight: bold; margin: 1em 0 0.5em; }
.prose h2 { font-size: 1.5em; font-weight: bold; margin: 1em 0 0.5em; }
.prose h3 { font-size: 1.25em; font-weight: bold; margin: 1em 0 0.5em; }

.prose p { margin: 1em 0; }

.prose ul, .prose ol { padding-left: 1.5em; margin: 1em 0; }
.prose li { margin: 0.5em 0; }

.prose a { color: #0066cc; text-decoration: underline; }

.prose blockquote {
  border-left: 3px solid #ddd;
  padding-left: 1em;
  margin: 1em 0;
  font-style: italic;
}

.prose img { max-width: 100%; height: auto; }

.prose pre {
  background: #f5f5f5;
  padding: 1em;
  overflow-x: auto;
  border-radius: 4px;
}

.prose code {
  background: #f5f5f5;
  padding: 0.2em 0.4em;
  border-radius: 2px;
  font-family: monospace;
}

/* Column text blocks */
.prose .two-column-text {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2rem;
}

.prose .three-column-text {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 1.5rem;
}

/* Responsive: stack columns on mobile */
@media (max-width: 768px) {
  .prose .two-column-text,
  .prose .three-column-text {
    grid-template-columns: 1fr;
    gap: 1rem;
  }
}

/* Summary boxes */
.prose .intro-summary,
.prose .executive-summary {
  background: #f8fafc;
  border-left: 4px solid #3b82f6;
  padding: 1.5rem;
  margin: 2rem 0;
}

/* Author details */
.prose .author-details {
  display: flex;
  align-items: flex-start;
  gap: 1rem;
  padding: 1rem;
  background: #f8fafc;
  border-radius: 8px;
}

.prose .author-details img {
  width: 64px;
  height: 64px;
  border-radius: 50%;
  object-fit: cover;
}

/* Footnotes & Acknowledgements */
.prose .footnotes,
.prose .acknowledgements {
  font-size: 0.875rem;
  color: #64748b;
  border-top: 1px solid #e2e8f0;
  padding-top: 1rem;
  margin-top: 2rem;
}
```

## Tags

### Fetch Posts by Tag

```javascript
const techPosts = await fetch(
  `/api/public/blog?projectId=...&apiKey=...&tag=technology`
);
```

### Get All Tags

To get all unique tags, fetch all posts and extract tags:

```javascript
const { posts } = await fetch('/api/public/blog?...').then(r => r.json());

const allTags = [...new Set(posts.flatMap(post => post.tags))];
```

## Featured Posts

Filter posts that have a featured image for featured sections:

```javascript
const featuredPosts = posts.filter(p => p.featuredImage);
```

## Date Handling

Posts have two date fields:
- `createdAt`: When the post was created (automatic)
- `dateWritten`: Custom date set by the author

Use `dateWritten` for display if available:

```javascript
const displayDate = post.dateWritten || post.createdAt;
```

## Preview Links

Preview links allow you to share unpublished blog posts with team members for review before publishing.

### Generating Preview Links

1. Open a blog post in the editor
2. Click the **Preview** button (purple) in the header
3. Click **Generate Preview Link**
4. Copy the generated link and share it with your team

### Features

- **Shareable**: Anyone with the preview link can view the post
- **Works for drafts**: Preview links work for unpublished posts
- **Secure**: Each preview link uses a unique, cryptographically secure token
- **Revocable**: You can regenerate or revoke preview links at any time

### Preview API

Fetch a post by preview token:

```javascript
const response = await fetch(
  'https://www.spaceblock.app/api/public/blog/preview/PREVIEW_TOKEN'
);
const { post, isPreview } = await response.json();

// isPreview will be true for preview requests
```

**Note:** Preview links do not require an API key. The preview token itself serves as secure authentication.

### Implementing a Preview Page

Create a route at `/blog/preview/:token` handled by `BlogPreviewPage.tsx`:

```tsx
// src/pages/BlogPreviewPage.tsx
import { useEffect, useState } from 'react'
import { useParams } from 'react-router-dom'
import { fetchBlogPreview } from '@/lib/api'
import { BlogContentBlock } from '@/components/BlogContentBlock'
import type { BlogBlock, BlogPost } from '@/lib/types'

export function BlogPreviewPage() {
  const { token } = useParams<{ token: string }>()
  const [post, setPost] = useState<BlogPost | null>(null)
  const [isPreview, setIsPreview] = useState(false)
  const [loading, setLoading] = useState(true)
  const [notFound, setNotFound] = useState(false)

  useEffect(() => {
    if (!token) return
    let cancelled = false

    fetchBlogPreview(token)
      .then(({ post: p, isPreview: ip }) => {
        if (cancelled) return
        setPost(p)
        setIsPreview(ip)
      })
      .catch(() => { if (!cancelled) setNotFound(true) })
      .finally(() => { if (!cancelled) setLoading(false) })

    return () => { cancelled = true }
  }, [token])

  if (loading) return <div className="flex items-center justify-center min-h-[60vh]">Loading preview…</div>
  if (notFound || !post) return (
    <div className="flex items-center justify-center min-h-[60vh] text-gray-500">
      Preview link is invalid or has expired.
    </div>
  )

  const blocks = Array.isArray(post.content) ? post.content as BlogBlock[] : null
  const htmlContent = typeof post.content === 'string' ? post.content : null

  return (
    <div>
      {isPreview && (
        <div className="sticky top-0 z-50 bg-amber-100 border-b border-amber-200 px-4 py-2 text-amber-800 text-center text-sm">
          <strong>Preview Mode</strong> — This post is not yet published.
          Only people with this link can see it.
        </div>
      )}

      <article className="max-w-4xl mx-auto px-8 py-16">
        {post.featuredImage && (
          <img src={post.featuredImage} alt={post.title} className="w-full h-72 object-cover rounded-2xl mb-10" />
        )}
        <h1 className="text-4xl font-bold mb-4">{post.title}</h1>

        {blocks ? (
          <div>{blocks.map(block => <BlogContentBlock key={block.id} block={block} />)}</div>
        ) : htmlContent ? (
          <div className="prose prose-lg max-w-none" dangerouslySetInnerHTML={{ __html: htmlContent }} />
        ) : null}
      </article>
    </div>
  )
}
```

Register the route in `App.tsx` **before** the blog post route:

```tsx
<Route path="/blog/preview/:token" element={<BlogPreviewPage />} />
<Route path="/blog/:slug" element={<BlogPostPage />} />
```

### `BlogContentBlock` Component

All block types are rendered by `src/components/BlogContentBlock.tsx`. Import React for the JSX type (required in React 19):

```tsx
import React from 'react'
import type { BlogBlock } from '@/lib/types'

export function BlogContentBlock({ block }: { block: BlogBlock }) {
  switch (block.type) {
    case 'richtext':
      return <div dangerouslySetInnerHTML={{ __html: block.content ?? '' }} />
    case 'heading': {
      const Tag = `h${block.level ?? 2}` as keyof React.JSX.IntrinsicElements
      return <Tag>{block.content}</Tag>
    }
    case 'image':
      return (
        <figure>
          <img src={block.content} alt={block.notes ?? ''} />
          {block.notes && <figcaption className="text-center text-gray-500 text-sm mt-2">{block.notes}</figcaption>}
        </figure>
      )
    case 'video':
      return <div className="aspect-video"><iframe src={block.content} className="w-full h-full" allowFullScreen /></div>
    case 'audio':
      return <audio src={block.content} controls className="w-full" />
    case 'quote':
      return <blockquote>{block.content}</blockquote>
    case 'divider':
      return <hr />
    case 'two-column-text':
      return (
        <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
          <div dangerouslySetInnerHTML={{ __html: block.column1 ?? '' }} />
          <div dangerouslySetInnerHTML={{ __html: block.column2 ?? '' }} />
        </div>
      )
    case 'three-column-text':
      return (
        <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
          <div dangerouslySetInnerHTML={{ __html: block.column1 ?? '' }} />
          <div dangerouslySetInnerHTML={{ __html: block.column2 ?? '' }} />
          <div dangerouslySetInnerHTML={{ __html: block.column3 ?? '' }} />
        </div>
      )
    case 'cta':
      return (
        <a href={block.url ?? '#'} className="inline-block px-6 py-3 bg-blue-600 text-white rounded-lg">
          {block.content}
        </a>
      )
    case 'download':
      return (
        <a href={block.url ?? '#'} download className="inline-flex items-center gap-2 px-4 py-2 border rounded-lg">
          {block.content ?? 'Download'}
        </a>
      )
    case 'intro-summary':
    case 'executive-summary':
      return <div className="bg-gray-50 border-l-4 border-blue-500 p-6 my-8"><div dangerouslySetInnerHTML={{ __html: block.content ?? '' }} /></div>
    case 'author-details':
      return (
        <div className="flex items-start gap-4 p-4 bg-gray-50 rounded-lg">
          {block.image && <img src={block.image} alt="" className="w-16 h-16 rounded-full" />}
          <div dangerouslySetInnerHTML={{ __html: block.content ?? '' }} />
        </div>
      )
    case 'acknowledgements':
    case 'footnotes':
      return <div className="text-sm text-gray-600 border-t pt-4 mt-8"><div dangerouslySetInnerHTML={{ __html: block.content ?? '' }} /></div>
    case 'table':
      return <div className="overflow-x-auto"><div dangerouslySetInnerHTML={{ __html: block.content ?? '' }} /></div>
    default:
      return null
  }
}
```

### Security Notes

- Preview tokens are unique per post
- Regenerating a token invalidates the previous one
- Revoking a token permanently disables the preview link
- Preview tokens are cryptographically secure (32 bytes, base64url encoded)
- No API key is required - the token itself serves as authentication

## Next Steps

- [Collections](./07-collections.md) - For other structured data like team members
- [Media Library](./08-media.md) - Managing images
