# Collections Guide

Collections are groups of structured items, perfect for team members, products, testimonials, portfolio items, and any repeatable content.

## Use Cases

- **Team Members**: Name, role, bio, photo
- **Products**: Name, price, description, image
- **Testimonials**: Quote, author, company
- **Portfolio Projects**: Title, description, images, link
- **FAQ Items**: Question, answer
- **Services**: Name, description, icon

## Creating Collections

### In the Dashboard

1. Go to your project
2. Click the "Collections" tab
3. Click "New Collection"
4. Enter collection name (e.g., "Team", "Products")
5. Add items to the collection

### Collection Structure

```typescript
interface Collection {
  id: string;
  name: string;
  slug: string;        // URL-friendly identifier
  description: string | null;
  itemCount: number;
}

interface CollectionItem {
  id: string;
  title: string;       // Defaults to "Untitled" if not provided
  slug: string;        // Auto-generated from title
  description: string | null;
  image: string | null;
  tags: string[];              // Categorization tags
  links: CollectionItemLink[]; // External links
  data: Record<string, any>;   // Custom fields (includes optional url)
  order: number;
}

interface CollectionItemLink {
  name: string;  // Display name for the link
  url: string;   // Full URL (must be valid)
}
```

## Item Content Fields

Each item has built-in fields and custom content. **All fields are optional** — you only need to fill in what's relevant for your use case.

**Built-in Fields:**
- `title` - Item name/title (defaults to "Untitled" if not provided)
- `slug` - URL-friendly identifier (auto-generated from title, with unique suffix if needed)
- `description` - Short description
- `image` - Featured image URL
- `tags` - Array of categorization tags
- `links` - Array of external links (name + url pairs)

**Custom Content:**
Store any additional data in the `data` field, including the optional `url` field:

```json
{
  "title": "John Doe",
  "slug": "john-doe",
  "description": "Lead Developer",
  "image": "https://cdn.example.com/john.jpg",
  "tags": ["Engineering", "Leadership", "Frontend"],
  "links": [
    { "name": "LinkedIn", "url": "https://linkedin.com/in/johndoe" },
    { "name": "GitHub", "url": "https://github.com/johndoe" },
    { "name": "Personal Website", "url": "https://johndoe.dev" }
  ],
  "data": {
    "url": "https://johndoe.dev",
    "role": "Lead Developer",
    "bio": "<p>John has 10 years of experience...</p>",
    "skills": ["React", "Node.js", "TypeScript"]
  }
}
```

## Tags

Tags allow you to categorize collection items for filtering and organization.

**Adding Tags in the Dashboard:**
1. Open a collection item
2. Click "Edit" to enter edit mode
3. In the sidebar, find the "Tags" section
4. Type a tag and press Enter or click the + button
5. Click "Save" to persist changes

**Using Tags in Your Frontend:**
```jsx
// Filter items by tag
const filteredItems = items.filter(item =>
  item.tags.includes('Featured')
);

// Display tags on an item
function ItemCard({ item }) {
  return (
    <div>
      <h3>{item.title}</h3>
      <div className="flex gap-2">
        {item.tags.map(tag => (
          <span key={tag} className="px-2 py-1 bg-blue-100 text-blue-700 rounded text-sm">
            {tag}
          </span>
        ))}
      </div>
    </div>
  );
}
```

## Links

Links allow you to associate external URLs with collection items - useful for social profiles, portfolios, documentation, or any external resources.

**Adding Links in the Dashboard:**
1. Open a collection item
2. Click "Edit" to enter edit mode
3. In the sidebar, find the "Links" section
4. Enter a display name (e.g., "LinkedIn")
5. Enter the full URL (e.g., "https://linkedin.com/in/username")
6. Click the + button to add the link
7. Click "Save" to persist changes

**Link Structure:**
```typescript
interface CollectionItemLink {
  name: string;  // Display name (e.g., "LinkedIn", "Website", "GitHub")
  url: string;   // Full URL (must be a valid URL)
}
```

**Using Links in Your Frontend:**
```jsx
function TeamMemberCard({ member }) {
  return (
    <div className="p-4 border rounded-lg">
      <img src={member.image} alt={member.title} className="w-24 h-24 rounded-full" />
      <h3 className="text-xl font-semibold">{member.title}</h3>
      <p className="text-gray-600">{member.description}</p>

      {/* Display links */}
      {member.links && member.links.length > 0 && (
        <div className="flex gap-3 mt-4">
          {member.links.map((link, index) => (
            <a
              key={index}
              href={link.url}
              target="_blank"
              rel="noopener noreferrer"
              className="text-blue-600 hover:underline text-sm"
            >
              {link.name}
            </a>
          ))}
        </div>
      )}
    </div>
  );
}
```

**Common Link Use Cases:**
- Team members: LinkedIn, Twitter, GitHub, personal website
- Products: Buy link, documentation, support page
- Portfolio items: Live demo, source code, case study
- Resources: Download link, external documentation

## Public API

### List Collections

```javascript
const response = await fetch(
  'https://www.spaceblock.app/api/public/project/{projectId}/collections/{collectionName}?apiKey=YOUR_API_KEY'
);
const { collection, items } = await response.json();
```

### Get Single Item

```javascript
const response = await fetch(
  'https://www.spaceblock.app/api/public/project/{projectId}/collections/{collectionName}/{itemSlug}?apiKey=YOUR_API_KEY'
);
const { item } = await response.json();
```

### Alternative Routes

```javascript
// Using collection slug
GET /api/public/collections/{collectionSlug}?projectId=...&apiKey=...

// Get specific item
GET /api/public/collections/{collectionSlug}/{itemSlug}?projectId=...&apiKey=...
```

## React Integration

### Team Section Example

```tsx
// src/components/sections/TeamSection.tsx
import { useEffect, useState } from 'react'
import { fetchCollection } from '@/lib/api'
import type { CollectionItem } from '@/lib/types'

export function TeamSection() {
  const [teamMembers, setTeamMembers] = useState<CollectionItem[]>([])

  useEffect(() => {
    fetchCollection('team')
      .then(({ items }) => setTeamMembers(items))
      .catch(() => {})
  }, [])

    <section className="py-16">
      <h2 className="text-3xl font-bold text-center mb-12">Our Team</h2>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
        {teamMembers.map(member => (
          <div key={member.id} className="text-center">
            {member.image && (
              <img
                src={member.image}
                alt={member.title}
                className="w-32 h-32 rounded-full mx-auto mb-4 object-cover"
              />
            )}
            <h3 className="text-xl font-semibold">{member.title}</h3>
            <p className="text-gray-600">{member.data?.role || member.description}</p>

            {/* Display item URL if set */}
            {member.data?.url && (
              <a href={member.data.url} target="_blank" rel="noopener noreferrer" className="text-blue-600 text-sm hover:underline">
                Visit Profile
              </a>
            )}

            {/* Display tags */}
            {member.tags?.length > 0 && (
              <div className="flex flex-wrap justify-center gap-1 mt-2">
                {member.tags.map(tag => (
                  <span key={tag} className="px-2 py-0.5 bg-gray-100 text-gray-600 rounded text-xs">
                    {tag}
                  </span>
                ))}
              </div>
            )}

            {member.data?.bio && (
              <div
                className="mt-2 text-sm"
                dangerouslySetInnerHTML={{ __html: member.data.bio }}
              />
            )}

            {/* Display links */}
            {member.links?.length > 0 && (
              <div className="flex justify-center gap-3 mt-3">
                {member.links.map((link, index) => (
                  <a
                    key={index}
                    href={link.url}
                    target="_blank"
                    rel="noopener noreferrer"
                    className="text-blue-600 text-sm hover:underline"
                  >
                    {link.name}
                  </a>
                ))}
              </div>
            )}
          </div>
        ))}
      </div>
    </section>
  );
}
```

### Products Grid Example

```jsx
export default function ProductGrid({ products }) {
  return (
    <div className="grid grid-cols-1 md:grid-cols-4 gap-6">
      {products.map(product => (
        <div key={product.id} className="border rounded-lg overflow-hidden">
          {product.image && (
            <img
              src={product.image}
              alt={product.title}
              className="w-full h-48 object-cover"
            />
          )}
          <div className="p-4">
            <h3 className="font-semibold">{product.title}</h3>
            <p className="text-gray-600 text-sm">{product.description}</p>
            {product.data?.price && (
              <p className="text-lg font-bold mt-2">
                ${product.data.price}
              </p>
            )}

            {/* Product links (e.g., buy now, documentation) */}
            {product.links?.length > 0 && (
              <div className="flex gap-2 mt-3">
                {product.links.map((link, index) => (
                  <a
                    key={index}
                    href={link.url}
                    target="_blank"
                    rel="noopener noreferrer"
                    className="text-sm text-blue-600 hover:underline"
                  >
                    {link.name}
                  </a>
                ))}
              </div>
            )}

            <a
              href={`/products/${product.slug}`}
              className="block mt-4 text-blue-600"
            >
              View Details
            </a>
          </div>
        </div>
      ))}
    </div>
  );
}
```

### Testimonials Example

```jsx
export default function Testimonials({ testimonials }) {
  return (
    <section className="bg-gray-50 py-16">
      <h2 className="text-3xl font-bold text-center mb-12">
        What Our Clients Say
      </h2>

      <div className="max-w-4xl mx-auto space-y-8">
        {testimonials.map(testimonial => (
          <blockquote
            key={testimonial.id}
            className="bg-white p-6 rounded-lg shadow"
          >
            <p className="text-lg italic mb-4">
              "{testimonial.data?.quote || testimonial.description}"
            </p>
            <footer className="flex items-center gap-4">
              {testimonial.image && (
                <img
                  src={testimonial.image}
                  alt={testimonial.title}
                  className="w-12 h-12 rounded-full"
                />
              )}
              <div>
                <cite className="font-semibold not-italic">
                  {testimonial.title}
                </cite>
                {testimonial.data?.company && (
                  <p className="text-gray-600 text-sm">
                    {testimonial.data.company}
                  </p>
                )}

                {/* Link to client's website or case study */}
                {testimonial.links?.length > 0 && (
                  <div className="flex gap-2 mt-1">
                    {testimonial.links.map((link, index) => (
                      <a
                        key={index}
                        href={link.url}
                        target="_blank"
                        rel="noopener noreferrer"
                        className="text-xs text-blue-600 hover:underline"
                      >
                        {link.name}
                      </a>
                    ))}
                  </div>
                )}
              </div>
            </footer>
          </blockquote>
        ))}
      </div>
    </section>
  );
}
```

## Using Collections with Templates

Reference a collection in a page template:

```html
<!-- In your template registry -->
<div data-cms-insertable="team-section">
  <span data-cms-id="team-title">Our Team</span>
  <select
    data-cms-id="team-collection"
    data-cms-type="collection-select"
  >
    <option value="team">Team</option>
  </select>
</div>
```

Then in your component:

```jsx
function TeamSectionBlock({ elementId, content }) {
  const collectionName = content[`${elementId}-team-collection`] || 'team';

  // Fetch collection items
  const { data: items } = useQuery(['collection', collectionName], () =>
    fetchCollection(collectionName)
  );

  return (
    <section data-cms-id={elementId}>
      <h2 data-cms-id={`${elementId}-team-title`}>
        {content[`${elementId}-team-title`]}
      </h2>
      <div className="grid grid-cols-3 gap-6">
        {items?.map(member => (
          <TeamMemberCard key={member.id} member={member} />
        ))}
      </div>
    </section>
  );
}

// TeamMemberCard with tags and links
function TeamMemberCard({ member }) {
  return (
    <div className="text-center p-4">
      <img src={member.image} alt={member.title} className="w-24 h-24 rounded-full mx-auto" />
      <h3 className="font-semibold mt-2">{member.title}</h3>
      <p className="text-gray-600 text-sm">{member.description}</p>

      {/* Tags */}
      {member.tags?.length > 0 && (
        <div className="flex flex-wrap justify-center gap-1 mt-2">
          {member.tags.map(tag => (
            <span key={tag} className="px-2 py-0.5 bg-blue-100 text-blue-700 rounded text-xs">
              {tag}
            </span>
          ))}
        </div>
      )}

      {/* Links */}
      {member.links?.length > 0 && (
        <div className="flex justify-center gap-2 mt-2">
          {member.links.map((link, i) => (
            <a key={i} href={link.url} target="_blank" rel="noopener noreferrer" className="text-blue-600 text-sm">
              {link.name}
            </a>
          ))}
        </div>
      )}
    </div>
  );
}
```

## Ordering Items

Items have an `order` field. Always sort by order:

```javascript
const sortedItems = items.sort((a, b) => a.order - b.order);
```

In the dashboard, items can be reordered via drag-and-drop.

## Best Practices

1. **Meaningful Slugs**: Use descriptive slugs for SEO-friendly URLs
2. **Consistent Content**: Keep content structure consistent across items
3. **Image Optimization**: Use CDN images for better performance
4. **Fallbacks**: Always provide fallback values for optional fields
5. **Sort by Order**: Respect the `order` field for consistent display
6. **Use Tags for Filtering**: Tags are great for categories, skills, or any filterable attributes
7. **Use Links for External Resources**: Store social profiles, portfolios, and external URLs in the links array instead of custom data fields
8. **Validate Links**: Links require valid URLs - ensure they include the protocol (https://)

## Next Steps

- [Media Library](./08-media.md) - Managing images for collections
- [API Reference](./09-api-reference.md) - Full API documentation
