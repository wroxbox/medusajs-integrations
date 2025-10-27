# Sanity CMS Integration - Quick Reference

**Quick reference guide and cheat sheet for Sanity + Medusa integration.**

## TL;DR

**What**: Sanity CMS for content + Medusa for commerce  
**Why**: Localization, rich content, content team independence  
**When**: Multi-language stores with content-rich products  
**Cost**: Free tier (generous) → $99/month+ for growth  
**Setup**: ~3-5 days initial implementation  

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│               SANITY + MEDUSA INTEGRATION                │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────┐              ┌─────────────────┐ │
│  │   MEDUSA         │              │   SANITY CMS    │ │
│  │   (Port 9000)    │──────sync───►│   (SaaS)        │ │
│  │                  │              │                 │ │
│  │  • Products      │              │  • Specs        │ │
│  │  • Pricing       │              │  • Localization │ │
│  │  • Inventory     │              │  • Addons       │ │
│  │  • Orders        │              │                 │ │
│  └────────┬─────────┘              └────────┬────────┘ │
│           │                                  │          │
│           │         ┌──────────────┐        │          │
│           └────────►│  STOREFRONT  │◄───────┘          │
│                     │  (Port 8000) │                   │
│                     │              │                   │
│                     │  Dual Fetch: │                   │
│                     │  • Medusa    │                   │
│                     │  • Sanity    │                   │
│                     │              │                   │
│                     │  /studio     │                   │
│                     └──────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

---

## Data Flow Summary

### Product Sync (Medusa → Sanity)

```
Product Created/Updated in Medusa
         ↓
Event: product.created/updated
         ↓
Subscriber triggers workflow
         ↓
Query product from Medusa
         ↓
Transform to Sanity format
         ↓
Upsert to Sanity via API
         ↓
Document stored in Sanity ✓
```

### Storefront Read (Customer Views Product)

```
Customer visits product page
         ↓
Parallel fetch:
├── Medusa API (commerce data)
└── Sanity API (content data)
         ↓
Merge data in component
         ↓
Render to customer ✓
```

---

## Key Files and Purposes

### Medusa Backend

| File | Purpose |
|------|---------|
| `src/modules/sanity/service.ts` | Sanity client, sync logic |
| `src/modules/sanity/index.ts` | Module registration |
| `src/subscribers/sanity-product-sync.ts` | Event listener |
| `src/workflows/sanity-sync-products/` | Sync orchestration |
| `src/links/product-sanity.ts` | Product↔Sanity relationship |
| `src/api/admin/sanity/syncs/route.ts` | Bulk sync endpoint |
| `src/api/admin/sanity/documents/[id]/sync/route.ts` | Single product sync |
| `src/api/admin/sanity/documents/[id]/route.ts` | Get Sanity document |
| `medusa-config.ts` | Module configuration |
| `.env` | `SANITY_API_TOKEN`, `SANITY_PROJECT_ID` |

### Next.js Storefront

| File | Purpose |
|------|---------|
| `sanity.config.ts` | Studio configuration |
| `src/sanity/env.ts` | Environment variables |
| `src/sanity/lib/client.ts` | Sanity client for fetching |
| `src/sanity/schemaTypes/documents/product.ts` | Product schema |
| `src/app/studio/[[...tool]]/page.tsx` | Studio route |
| `src/app/[countryCode]/(main)/products/[handle]/page.tsx` | Product page (dual fetch) |
| `.env.local` | `NEXT_PUBLIC_SANITY_PROJECT_ID` |

---

## Environment Variables

### Medusa (`.env`)

```bash
# Required
SANITY_API_TOKEN=sk_abc123...           # Editor permissions
SANITY_PROJECT_ID=abc123xyz             # From manage.sanity.io
SANITY_STUDIO_URL=http://localhost:8000/studio

# Existing
DATABASE_URL=postgresql://...
STORE_CORS=http://localhost:8000
```

### Storefront (`.env.local`)

```bash
# Required
NEXT_PUBLIC_SANITY_PROJECT_ID=abc123xyz
NEXT_PUBLIC_SANITY_DATASET=production

# Existing
NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY=pk_...
```

---

## Common Commands

### Setup

```bash
# Create Sanity project
npx sanity@latest init

# Install dependencies (Medusa)
npm install @sanity/client

# Install dependencies (Storefront)
npm install sanity next-sanity @sanity/vision

# Setup Medusa database
npx medusa db:setup
```

### Development

```bash
# Start Medusa
npm run dev  # Port 9000

# Start Storefront
npm run dev  # Port 8000

# Access Sanity Studio
open http://localhost:8000/studio
```

### Sync Operations

```bash
# Bulk sync all products
curl -X POST http://localhost:9000/admin/sanity/syncs

# Sync single product
curl -X POST http://localhost:9000/admin/sanity/documents/prod_123/sync

# Check sync status
curl http://localhost:9000/admin/sanity/syncs

# Get Sanity document
curl http://localhost:9000/admin/sanity/documents/prod_123
```

### Testing

```bash
# Test Sanity API connection
curl https://YOUR_PROJECT_ID.api.sanity.io/v2024-01-01/data/query/production \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d 'query=*[_type=="product"][0]'

# Query specific product
curl https://YOUR_PROJECT_ID.api.sanity.io/v2024-01-01/data/query/production \
  -d 'query=*[_id=="prod_123"][0]'
```

---

## API Endpoints

### Medusa Admin APIs

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/admin/sanity/syncs` | Bulk sync all products |
| GET | `/admin/sanity/syncs` | List workflow executions |
| POST | `/admin/sanity/documents/:id/sync` | Sync single product |
| GET | `/admin/sanity/documents/:id` | Get Sanity document + Studio link |

### Sanity APIs

| Endpoint | Purpose |
|----------|---------|
| `https://[PROJECT].api.sanity.io/v2024-01-01/data/query/production` | GROQ queries |
| `https://[PROJECT].api.sanity.io/v2024-01-01/data/mutate/production` | Create/update/delete |
| `https://[PROJECT].api.sanity.io/v2024-01-01/data/doc/production/:id` | Get document by ID |

---

## GROQ Query Examples

### Get Product by ID

```groq
*[_id == $id][0]
```

Usage:
```typescript
const product = await client.fetch(
  `*[_id == $id][0]`,
  { id: 'prod_123' }
)
```

### Get Product with Language-Specific Spec

```groq
*[_id == $id][0] {
  _id,
  title,
  "spec": specs[lang == $lang][0]
}
```

### Get Product with Addons

```groq
*[_id == $id][0] {
  _id,
  title,
  addons {
    title,
    products[]-> {
      _id,
      title
    }
  }
}
```

### Get All Products with English Specs

```groq
*[_type == "product"] {
  _id,
  title,
  "spec": specs[lang == "en"][0].content
}
```

### Count Products

```groq
count(*[_type == "product"])
```

---

## Sanity Document Structure

```typescript
{
  "_id": "prod_01ABC123",           // Same as Medusa product ID
  "_type": "product",
  "_createdAt": "2024-01-15T10:00:00Z",
  "_updatedAt": "2024-01-16T14:30:00Z",
  "_rev": "abc123-xyz789",          // Revision ID
  
  "title": "Premium Cotton T-Shirt", // Synced from Medusa
  
  "specs": [                         // Managed in Sanity
    {
      "_key": "en_001",
      "_type": "spec",
      "lang": "en",
      "title": "Premium Cotton T-Shirt",
      "content": "Made from 100% organic cotton..."
    },
    {
      "_key": "es_002",
      "_type": "spec",
      "lang": "es",
      "title": "Camiseta de Algodón Premium",
      "content": "Hecha de algodón 100% orgánico..."
    }
  ],
  
  "addons": {                        // Managed in Sanity
    "title": "Complete Your Look",
    "products": [
      { "_ref": "prod_jeans_001", "_type": "reference" },
      { "_ref": "prod_shoes_002", "_type": "reference" }
    ]
  }
}
```

---

## What Syncs vs What Doesn't

### ✅ Syncs Automatically (Medusa → Sanity)

- Product ID
- Product title
- Initial specs array structure

### ❌ Does NOT Sync

#### Medusa → Sanity:
- Pricing
- Inventory
- Variants
- Images
- Collections
- Categories

#### Sanity → Medusa:
- Content updates
- Localized specs
- Addons
- Custom fields

**One-way sync only: Medusa → Sanity**

---

## Troubleshooting Checklist

### Product Not Syncing

- [ ] Check Medusa logs for errors
- [ ] Verify `SANITY_API_TOKEN` is set
- [ ] Verify token has "Editor" permissions
- [ ] Check `SANITY_PROJECT_ID` is correct
- [ ] Ensure CORS includes `http://localhost:9000`
- [ ] Look for "Connected to Sanity" in logs
- [ ] Test Sanity API connection

### Studio Not Loading

- [ ] Clear Next.js cache (`rm -rf .next`)
- [ ] Check `sanity.config.ts` exists in root
- [ ] Verify `NEXT_PUBLIC_SANITY_PROJECT_ID` is set
- [ ] Check Studio route exists at `/studio/[[...tool]]/page.tsx`
- [ ] Look for console errors in browser

### Content Not Showing

- [ ] Verify product exists in Sanity Studio
- [ ] Check storefront calls `client.getDocument(id)`
- [ ] Inspect Network tab for Sanity API call
- [ ] Check for JavaScript errors in console
- [ ] Verify product ID matches between systems
- [ ] Test Sanity query directly

---

## Performance Tips

### 1. Parallel Fetching

```typescript
// Good ✅
const [medusa, sanity] = await Promise.all([
  fetchMedusa(handle),
  fetchSanity(id)
])

// Bad ❌
const medusa = await fetchMedusa(handle)
const sanity = await fetchSanity(id)
```

### 2. Aggressive Caching

```typescript
// Sanity content (stable)
cache: 'force-cache',
next: { revalidate: 3600 }  // 1 hour

// Medusa data (dynamic)
cache: 'no-store'  // or revalidate: 60
```

### 3. Use Projections

```groq
// Good ✅ - Only fetch what you need
*[_id == $id][0] {
  _id,
  "spec": specs[lang == $lang][0].content
}

// Bad ❌ - Fetches everything
*[_id == $id][0]
```

### 4. Optimize Images

```typescript
import imageUrlBuilder from '@sanity/image-url'

const url = imageUrlBuilder(client)
  .image(source)
  .width(800)
  .quality(80)
  .format('webp')
  .url()
```

---

## Cost Estimation

### Sanity Pricing

| Tier | Cost | Users | API Requests | Storage |
|------|------|-------|--------------|---------|
| **Free** | $0 | 3 | 100k/month | 10k docs |
| **Growth** | $99/month | 5 | 500k/month | Unlimited |
| **Enterprise** | Custom | Unlimited | Custom | Custom |

### Typical Usage

| Store Size | Products | Visitors | Est. Requests | Est. Cost |
|------------|----------|----------|---------------|-----------|
| Small | < 1,000 | < 100k | < 100k | **Free** |
| Medium | 5,000 | 500k | 200-400k | **$99-150/month** |
| Large | 50,000 | 5M | 1M+ | **$300+/month** |

---

## Quick Decision Guide

### ✅ Use Sanity If:

- [ ] Multi-language required (3+ languages)
- [ ] Content-rich products
- [ ] Dedicated content team
- [ ] Real-time collaboration needed
- [ ] International e-commerce
- [ ] Budget allows ($100+/month)

### ❌ Skip Sanity If:

- [ ] Single language only
- [ ] Simple product descriptions
- [ ] No content team
- [ ] Tight budget
- [ ] MVP/testing phase
- [ ] Want self-hosting

**Score: 4+ ✅ → Use Sanity | 3 or less → Skip**

---

## Security Checklist

- [ ] Store API tokens in `.env` (never commit)
- [ ] Use "Editor" token for backend only
- [ ] Use "Viewer" token for frontend (if needed)
- [ ] Configure CORS in Sanity project
- [ ] Add production URLs to CORS
- [ ] Never expose Editor token to frontend
- [ ] Rotate tokens periodically
- [ ] Monitor API usage

---

## Production Checklist

### Before Launch

- [ ] Test product sync thoroughly
- [ ] Bulk sync existing products
- [ ] Train content team on Studio
- [ ] Configure production CORS
- [ ] Set production environment variables
- [ ] Test storefront data fetching
- [ ] Implement error boundaries
- [ ] Set up monitoring
- [ ] Document content workflows
- [ ] Create backup/export process

### Monitoring

- [ ] Sanity API usage (dashboard)
- [ ] Workflow execution success rate
- [ ] Page load performance
- [ ] Error tracking (Sentry, etc.)
- [ ] Content update velocity

---

## Support Resources

### Documentation

- [Medusa Docs](https://docs.medusajs.com)
- [Sanity Docs](https://www.sanity.io/docs)
- [GROQ Reference](https://www.sanity.io/docs/groq)
- [This Integration Guide](./04-implementation.md)

### Community

- [Medusa Discord](https://discord.gg/medusajs)
- [Sanity Slack](https://slack.sanity.io)
- [Medusa GitHub](https://github.com/medusajs/medusa)

### Tools

- [Sanity Management](https://manage.sanity.io)
- [GROQ Playground](https://www.sanity.io/docs/query-cheat-sheet)
- [Sanity CLI Docs](https://www.sanity.io/docs/cli)

---

## Quick Links

- [README](./README.md) - Documentation index
- [Overview](./01-overview.md) - What and why
- [Architecture](./02-architecture.md) - How it works
- [Data Flow](./03-data-flow.md) - Step-by-step scenarios
- [Implementation](./04-implementation.md) - Setup guide
- [Pros & Cons](./05-pros-cons.md) - Decision making
- [FAQ](./FAQ.md) - Common questions

---

**Need help?** Check the [FAQ](./FAQ.md) or ask in [Medusa Discord](https://discord.gg/medusajs).

