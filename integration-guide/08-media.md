# Media Library Guide

The Media Library provides centralized image and file management with CDN delivery.

## Features

- Upload images and files
- Automatic CDN delivery via Bunny.net
- Image picker in Visual Editor
- Organize by project
- Optimized delivery

## Setup

### Configure Bunny.net CDN

1. Create a [Bunny.net](https://bunny.net) account
2. Create a Storage Zone
3. Create a Pull Zone linked to the Storage Zone
4. In SpaceBlock Project Settings, enter:
   - **Storage Zone Name**: Your storage zone name
   - **Storage API Key**: Found in FTP & API Access
   - **CDN URL**: Your pull zone URL (e.g., `https://myzone.b-cdn.net`)

### Required Settings

```
Storage Zone Name: my-storage-zone
Storage API Key: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
CDN URL: https://my-storage-zone.b-cdn.net
```

## Using the Media Library

### In the Dashboard

1. Go to your project
2. Click the "Media" tab
3. Upload files via drag-and-drop or file picker
4. Click on images to copy URL

### In the Visual Editor

1. Click on an image element
2. Click "Choose Image" in the sidebar
3. Select from existing images or upload new
4. The image URL is automatically set

## Image URLs

Uploaded images are served from your CDN:

```
https://your-zone.b-cdn.net/project-id/filename.jpg
```

## API Endpoints

### List Media

```javascript
// Requires authentication
const response = await fetch('/api/projects/{projectId}/media/list', {
  headers: { 'Authorization': 'Bearer YOUR_TOKEN' }
});
const { media } = await response.json();

// Returns array of media items
[
  {
    id: 'media-123',
    filename: 'hero-image.jpg',
    url: 'https://cdn.example.com/project-123/hero-image.jpg',
    size: 245000,
    mimeType: 'image/jpeg',
    createdAt: '2024-01-15T10:30:00Z'
  }
]
```

### Upload Media

```javascript
const formData = new FormData();
formData.append('file', fileInput.files[0]);

const response = await fetch('/api/projects/{projectId}/media', {
  method: 'POST',
  headers: { 'Authorization': 'Bearer YOUR_TOKEN' },
  body: formData
});

const { media } = await response.json();
console.log(media.url); // CDN URL
```

### Delete Media

```javascript
await fetch('/api/projects/{projectId}/media/{mediaId}', {
  method: 'DELETE',
  headers: { 'Authorization': 'Bearer YOUR_TOKEN' }
});
```

## Using Media in Components

### In HTML

```html
<img
  data-cms-id="hero-image"
  data-cms-type="image"
  src="https://your-cdn.b-cdn.net/project-id/hero.jpg"
  alt="Hero image"
/>
```

### In React

```jsx
function HeroSection({ heroImage }) {
  return (
    <section>
      <img
        data-cms-id="hero-image"
        data-cms-type="image"
        src={heroImage || '/placeholder.jpg'}
        alt="Hero"
        className="w-full h-96 object-cover"
      />
    </section>
  );
}
```

## Image Optimization Tips

### Responsive Images

```html
<img
  data-cms-id="hero-image"
  data-cms-type="image"
  src="https://cdn.example.com/image.jpg"
  srcset="
    https://cdn.example.com/image-480.jpg 480w,
    https://cdn.example.com/image-800.jpg 800w,
    https://cdn.example.com/image-1200.jpg 1200w
  "
  sizes="(max-width: 480px) 480px, (max-width: 800px) 800px, 1200px"
  alt="Responsive image"
/>
```

### Lazy Loading

```html
<img
  data-cms-id="section-image"
  data-cms-type="image"
  src="https://cdn.example.com/image.jpg"
  loading="lazy"
  alt="Lazy loaded image"
/>
```

### With Next.js Image

```jsx
import Image from 'next/image';

function OptimizedImage({ src, alt }) {
  return (
    <div data-cms-id="hero-image" data-cms-type="image">
      <Image
        src={src}
        alt={alt}
        width={1200}
        height={600}
        priority
      />
    </div>
  );
}
```

## File Types

Supported file types:
- Images: `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`, `.svg`
- Documents: `.pdf`, `.doc`, `.docx`
- Other: Files are accepted but may not preview

## Storage Structure

Files are organized by project:
```
storage-zone/
├── project-123/
│   ├── hero-image.jpg
│   ├── team-photo.png
│   └── logo.svg
└── project-456/
    └── banner.jpg
```

## Best Practices

1. **Descriptive Filenames**: Use meaningful names like `team-john-doe.jpg`
2. **Optimize Before Upload**: Compress images for faster loading
3. **Use WebP**: Prefer WebP format for better compression
4. **Alt Text**: Always provide alt text for accessibility
5. **Lazy Load**: Use lazy loading for below-fold images

## Troubleshooting

### Upload Fails

- Check Bunny.net credentials in Project Settings
- Verify storage zone exists and API key is correct
- Check file size limits (default: 10MB)

### Images Not Loading

- Verify CDN URL is correct
- Check if pull zone is properly configured
- Clear browser cache

### 403 Forbidden

- Bunny API key may be expired
- Storage zone permissions may be misconfigured

## Next Steps

- [API Reference](./09-api-reference.md) - Complete API documentation
