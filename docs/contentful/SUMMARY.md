# Quick Reference Summary

This is a quick reference cheat sheet for the Contentful-Medusa integration.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                 TWO-WAY SYNC ARCHITECTURE                │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  MEDUSA (Commerce)  ◄────sync────►  CONTENTFUL (Content)│
│  • Products                         • Localized Content │
│  • Pricing                          • Rich Media        │
│  • Inventory                        • Multi-language    │
│  • Orders                           • Workflows         │
│                                                          │
│           ↓                              ↓              │
│           └──────── STOREFRONT ──────────┘              │
│                  (Fetches from both)                    │
└─────────────────────────────────────────────────────────┘
```

## Quick Setup (10 Minutes)

```bash
# 1. Install plugin
npm install medusa-plugin-contentful

# 2. Configure environment
cat >> .env << EOF
CONTENTFUL_SPACE_ID=your_space_id
CONTENTFUL_ACCESS_TOKEN=CFPAT_your_token
CONTENTFUL_ENV=master
EOF

# 3. Add to medusa-config.js
# plugins: [
#   {
#     resolve: "medusa-plugin-contentful",
#     options: {
#       space_id: process.env.CONTENTFUL_SPACE_ID,
#       access_token: process.env.CONTENTFUL_ACCESS_TOKEN,
#       environment: process.env.CONTENTFUL_ENV,
#     },
#   },
# ]

# 4. Create content types in Contentful
# (Use UI or migration scripts)

# 5. Configure webhook
# Contentful → Settings → Webhooks
# URL: https://your-backend.com/hooks/contentful

# 6. Start Medusa
npx medusa develop
```

## Key Files & Locations

### Medusa Backend

| File | Purpose |
|------|---------|
| `medusa-config.js` | Plugin configuration |
| `.env` | API tokens and credentials |
| `node_modules/medusa-plugin-contentful/` | Plugin code |
| `/hooks/contentful` | Webhook endpoint |

### Contentful

| Location | Purpose |
|----------|---------|
| Settings → API keys | Manage access tokens |
| Settings → Locales | Configure languages |
| Content model → Product | Product schema |
| Content → Products | Product entries |
| Settings → Webhooks | Configure webhooks |
| Assets | Media library |

## Environment Variables

### Medusa Backend

```bash
# Required
CONTENTFUL_SPACE_ID=abc123xyz
CONTENTFUL_ACCESS_TOKEN=CFPAT-management-token
CONTENTFUL_ENV=master

# Optional
CONTENTFUL_WEBHOOK_SECRET=your_secret_123
```

### Storefront

```bash
# Required (public-safe delivery token)
NEXT_PUBLIC_CONTENTFUL_SPACE_ID=abc123xyz
NEXT_PUBLIC_CONTENTFUL_DELIVERY_TOKEN=delivery_token_xyz
NEXT_PUBLIC_CONTENTFUL_ENV=master

# Optional (for preview mode)
NEXT_PUBLIC_CONTENTFUL_PREVIEW_TOKEN=preview_token_xyz
```

## API Endpoints

### Contentful APIs

| API | Purpose | Base URL | Token Type |
|-----|---------|----------|------------|
| Management (CMA) | Write operations | `api.contentful.com` | Management token |
| Delivery (CDA) | Read published | `cdn.contentful.com` | Delivery token |
| Preview (CPA) | Read drafts | `preview.contentful.com` | Preview token |

### Medusa Webhook

```
POST /hooks/contentful
Content-Type: application/json
X-Contentful-Topic: ContentManagement.Entry.publish

Body: Contentful webhook payload
```

## Data Flow Summary

### Medusa → Contentful (Automatic)

```
Product Created in Medusa
  ↓
Event: "product.created"
  ↓
Redis Event Bus
  ↓
Contentful Subscriber
  ↓
Transform Data
  ↓
POST to Contentful Management API
  ↓
Entry Created in Contentful ✓
```

**Time:** ~150-400ms per product

### Contentful → Medusa (Webhook)

```
Entry Updated in Contentful
  ↓
Click "Publish"
  ↓
Contentful Fires Webhook
  ↓
POST to /hooks/contentful
  ↓
Medusa Validates & Processes
  ↓
Update Product in Medusa ✓
```

**Time:** ~100-200ms

## Common Commands

### Medusa

```bash
# Start development server
npx medusa develop

# Check Redis connection
redis-cli ping

# View logs
npx medusa develop | grep contentful

# Test plugin loaded
npx medusa develop 2>&1 | grep "Contentful"
```

### Contentful CLI

```bash
# Install CLI
npm install -g contentful-cli

# Login
contentful login

# Export space
contentful space export --space-id $SPACE_ID

# Import space
contentful space import --space-id $SPACE_ID --content-file export.json

# Run migration
contentful space migration --space-id $SPACE_ID scripts/migration.js
```

### Testing

```bash
# Test Contentful API
curl "https://cdn.contentful.com/spaces/$SPACE_ID/entries?content_type=product" \
  -H "Authorization: Bearer $DELIVERY_TOKEN"

# Test Medusa webhook endpoint
curl -X POST https://your-backend.com/hooks/contentful \
  -H "Content-Type: application/json" \
  -d '{"test": "payload"}'

# Test Redis
redis-cli ping
```

## Code Snippets

### Fetch Product with Content (Storefront)

**Pattern 1: Direct Dual Fetch**
```typescript
import { medusaClient } from './medusa';
import { contentfulClient } from './contentful';

export async function getProduct(handle: string, locale: string = 'en-US') {
  const [productRes, contentRes] = await Promise.all([
    medusaClient.products.list({ handle }),
    contentfulClient.getEntries({
      content_type: 'product',
      'fields.handle': handle,
      locale: locale,
    }),
  ]);
  
  return {
    ...productRes.products[0],
    ...contentRes.items[0]?.fields,
  };
}
```

**Pattern 2: Server-Side Merge (Next.js)**
```typescript
// app/products/[handle]/page.tsx
export async function generateMetadata({ params }: { params: { handle: string } }) {
  const product = await getProduct(params.handle);
  
  return {
    title: product.seoTitle || product.title,
    description: product.seoDescription,
  };
}
```

### Create Contentful Client

```typescript
// lib/contentful.ts
import { createClient } from 'contentful';

// Delivery API (published content)
export const contentfulClient = createClient({
  space: process.env.NEXT_PUBLIC_CONTENTFUL_SPACE_ID!,
  accessToken: process.env.NEXT_PUBLIC_CONTENTFUL_DELIVERY_TOKEN!,
  environment: process.env.NEXT_PUBLIC_CONTENTFUL_ENV || 'master',
});

// Preview API (includes drafts)
export const previewClient = createClient({
  space: process.env.NEXT_PUBLIC_CONTENTFUL_SPACE_ID!,
  accessToken: process.env.NEXT_PUBLIC_CONTENTFUL_PREVIEW_TOKEN!,
  environment: process.env.NEXT_PUBLIC_CONTENTFUL_ENV || 'master',
  host: 'preview.contentful.com',
});
```

### Fetch by Locale

```typescript
// Fetch Spanish content
const content = await contentfulClient.getEntries({
  content_type: 'product',
  'fields.medusaId': productId,
  locale: 'es-ES',
});

// Fetch with fallback
const contentWithFallback = await contentfulClient.getEntries({
  content_type: 'product',
  'fields.medusaId': productId,
  locale: 'fr-FR',  // Will fallback to en-US if French not available
});
```

### Cache Contentful Responses

```typescript
import Redis from 'ioredis';
const redis = new Redis();

export async function getProductContent(productId: string, locale: string = 'en-US') {
  const cacheKey = `product:${productId}:${locale}`;
  
  // Check cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Fetch from Contentful
  const content = await contentfulClient.getEntries({
    content_type: 'product',
    'fields.medusaId': productId,
    locale: locale,
  });
  
  // Cache for 1 hour
  await redis.set(cacheKey, JSON.stringify(content), 'EX', 3600);
  
  return content;
}
```

## Content Type Structure (Product Example)

```javascript
{
  "name": "Product",
  "fields": [
    {
      "id": "medusaId",
      "name": "Medusa ID",
      "type": "Symbol",
      "required": true,
      "localized": false,
      "unique": true
    },
    {
      "id": "title",
      "name": "Title",
      "type": "Symbol",
      "required": true,
      "localized": true  // ← Supports multiple languages
    },
    {
      "id": "description",
      "name": "Description",
      "type": "Text",
      "localized": true
    },
    {
      "id": "richDescription",
      "name": "Rich Description",
      "type": "RichText",
      "localized": true  // ← Editorial content per language
    },
    {
      "id": "images",
      "name": "Images",
      "type": "Array",
      "items": { "type": "Link", "linkType": "Asset" }
    },
    {
      "id": "seoTitle",
      "name": "SEO Title",
      "type": "Symbol",
      "localized": true
    },
    {
      "id": "seoDescription",
      "name": "SEO Description",
      "type": "Text",
      "localized": true
    }
  ]
}
```

## Troubleshooting Checklist

### Products Not Syncing to Contentful?

- [ ] Redis running? (`redis-cli ping`)
- [ ] Plugin in `medusa-config.js`?
- [ ] Management token correct? (CFPAT-...)
- [ ] Content type "product" exists in Contentful?
- [ ] Field "medusaId" exists and is unique?
- [ ] Check Medusa logs for errors

### Webhook Not Working?

- [ ] Backend publicly accessible?
- [ ] Webhook URL correct in Contentful?
- [ ] Check Contentful webhook activity log
- [ ] Check Medusa logs for webhook receives
- [ ] Webhook secret matches (if configured)?
- [ ] For local dev: Use ngrok or similar

### Localization Not Working?

- [ ] Locale configured in Contentful? (Settings → Locales)
- [ ] Field marked as localized? (Content model → Field settings)
- [ ] Translation added for that locale?
- [ ] Fetching with correct locale parameter?
- [ ] Fallback locale configured?

### Rate Limit Issues?

- [ ] Check your plan (Free: 10 req/s, Team: 98 req/s)
- [ ] Implement Redis caching
- [ ] Use Delivery API (not Management API for reads)
- [ ] Consider upgrading plan

## Performance Tips

1. **Cache Everything**
```typescript
// Redis caching with TTL
await redis.set(key, value, 'EX', 3600);  // 1 hour
```

2. **Parallel Fetching**
```typescript
// Don't do this (sequential)
const product = await medusa.get(...);
const content = await contentful.get(...);

// Do this (parallel)
const [product, content] = await Promise.all([
  medusa.get(...),
  contentful.get(...)
]);
```

3. **Use GraphQL for Complex Queries**
```graphql
query {
  productCollection(where: { medusaId: "prod_123" }) {
    items {
      title
      richDescription
      imagesCollection(limit: 5) {
        items { url }
      }
    }
  }
}
```

4. **Leverage Contentful's CDN**
```typescript
// Use Delivery API (CDN-backed)
const client = contentful.createClient({
  accessToken: DELIVERY_TOKEN,  // Not management token
  // host defaults to cdn.contentful.com ✓
});
```

5. **Image Optimization**
```html
<!-- Original: 4000x4000px, 2MB -->
<img src="https://images.ctfassets.net/.../image.jpg" />

<!-- Optimized: 800x800px, 50KB -->
<img src="https://images.ctfassets.net/.../image.jpg?w=800&h=800&fm=webp&q=80" />

<!-- Responsive -->
<img 
  srcset="
    https://...image.jpg?w=400&fm=webp 400w,
    https://...image.jpg?w=800&fm=webp 800w,
    https://...image.jpg?w=1200&fm=webp 1200w
  "
  sizes="(max-width: 400px) 400px, (max-width: 800px) 800px, 1200px"
/>
```

## Pricing Quick Reference

| Tier | Monthly Cost | API Calls | Spaces | Users | Best For |
|------|--------------|-----------|--------|-------|----------|
| **Community** | Free | 48 req/s | 2 | 5 | Testing, small projects |
| **Team** | $489 | 98 req/s | 3 | 10 | Growing businesses |
| **Enterprise** | Custom | Custom | ∞ | ∞ | Large organizations |

**Additional Costs:**
- Bandwidth overages: $0.12/GB (after 1TB on Team plan)
- No overage charges on free tier (hard limits)

## Decision Matrix

| Need | Use Contentful | Use Alternative |
|------|---------------|-----------------|
| Multi-language | ✅ | Strapi (complex setup) |
| Managed service | ✅ | Sanity (also managed) |
| Open source | ❌ | Strapi or Payload |
| Self-hosting | ❌ | Strapi or Payload |
| Enterprise CMS | ✅ | — |
| Budget < $500/mo | ❌ | Strapi (self-hosted) |
| Global CDN | ✅ | — |
| Two-way sync | ✅ | Payload (best), Strapi (limited) |

## Useful Links

### Official Documentation

- [Medusa v1 Contentful Plugin](https://docs.medusajs.com/v1/plugins/cms/contentful)
- [Medusa v2 Contentful Integration](https://docs.medusajs.com/resources/integrations/guides/contentful)
- [Contentful Docs](https://www.contentful.com/developers/docs/)
- [Contentful Management API](https://www.contentful.com/developers/docs/references/content-management-api/)
- [Contentful Delivery API](https://www.contentful.com/developers/docs/references/content-delivery-api/)

### npm Packages

- [medusa-plugin-contentful](https://www.npmjs.com/package/medusa-plugin-contentful)
- [contentful](https://www.npmjs.com/package/contentful) - Delivery API client
- [contentful-management](https://www.npmjs.com/package/contentful-management) - Management API client

### Community

- [Medusa Discord](https://discord.gg/medusajs) - #plugins channel
- [Contentful Community](https://www.contentful.com/community/)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/contentful)

## Related Documentation

- **[README](./README.md)** - Full overview and getting started
- **[Overview](./01-overview.md)** - Understand the integration
- **[Architecture](./02-architecture.md)** - Technical deep dive
- **[Data Flow](./03-data-flow.md)** - Step-by-step scenarios
- **[Implementation](./04-implementation.md)** - Setup guide
- **[Pros & Cons](./05-pros-cons.md)** - Decision guide
- **[FAQ](./FAQ.md)** - Common questions

---

**Need Help?** Check the [FAQ](./FAQ.md) or [Implementation Guide](./04-implementation.md)

**Ready to Start?** Follow the [Quick Setup](#quick-setup-10-minutes) above!

