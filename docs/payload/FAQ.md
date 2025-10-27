# Payload Integration - Frequently Asked Questions

## General Questions

### Q: What exactly does the Payload integration do?

**A**: The integration creates a hybrid architecture where:

- **Medusa** manages commerce (products, pricing, inventory, orders, payments)
- **Payload CMS** manages content (rich descriptions, images, SEO metadata)
- **Next.js storefront** fetches data from both systems simultaneously

When you create a product in Medusa, it automatically syncs basic structure to Payload. Content managers then enhance products in Payload with rich text, images, and SEO without touching commerce data.

### Q: Does the storefront read data from Medusa or Payload?

**A**: **Both!** The storefront reads from:

| Data Type | Source | Example |
|-----------|--------|---------|
| Product structure | Medusa | Title, handle, variants |
| **Prices** | **Medusa** | Current prices, currency |
| **Inventory** | **Medusa** | Stock levels |
| **Cart/Checkout** | **Medusa** | Commerce operations |
| Rich descriptions | Payload | Formatted content |
| Images | Payload | Product photos |
| SEO metadata | Payload | Meta tags |

**How it works**:
```typescript
// Storefront queries Medusa with special field
GET /store/products/blue-hoodie?fields=*payload_product

// Medusa returns combined data:
{
  // From Medusa database:
  id: "prod_123",
  title: "Blue Hoodie",
  variants: [...],
  prices: [...],          // ← Medusa
  inventory: 25,          // ← Medusa
  
  // From Payload (via link):
  payload_product: {
    description: {...},   // ← Payload rich text
    images: [...],        // ← Payload media
    seo: {...}           // ← Payload SEO
  }
}
```

### Q: Is data synced in real-time?

**A**: **One-way sync, near real-time (< 1 second):**

- Medusa → Payload: **Yes, automatic** (events trigger workflows)
- Payload → Medusa: **No, by design**

**Why no reverse sync?**
- Payload manages content presentation
- Medusa is source of truth for commerce structure
- Prevents circular updates and conflicts
- Content changes don't need to affect commerce operations

### Q: Can content managers change product titles in Payload?

**A**: **Yes and no:**

- They CAN change the title in Payload
- The change stays in Payload (doesn't sync back to Medusa)
- Storefront shows Payload title (if available), falls back to Medusa title
- Medusa title remains unchanged

**This is intentional** - allows marketing team to optimize titles for different contexts while keeping original product name in commerce system.

### Q: What happens if I delete a product?

**A**: Delete in Medusa → Automatically deletes from Payload

1. Admin deletes product in Medusa
2. Event: `product.deleted` emitted
3. Subscriber triggers `deletePayloadProductsWorkflow`
4. Workflow calls Payload API to delete matching product
5. Product removed from both systems

**Cannot delete from Payload** - access control prevents this (only Medusa can delete).

## Data Flow Questions

### Q: When I create a product, what exactly gets synced to Payload?

**A**: **Basic structure only:**

✅ **Synced to Payload:**
- Product ID (as `medusa_id`)
- Title
- Subtitle  
- Description (converted to rich text format)
- Handle (URL slug)
- Options (e.g., "Size", "Color")
- Variants (e.g., "Small", "Medium", "Large")
- Created/Updated timestamps

❌ **NOT synced:**
- Prices
- Inventory levels
- Product collections
- Categories
- Tags
- Images/thumbnails
- Shipping profiles
- Any Medusa-specific commerce data

**After sync**, content managers add:
- Rich formatted descriptions
- Product images
- SEO metadata
- Any custom content fields

### Q: How do variant updates work?

**A**: **Step by step:**

1. Admin updates variant in Medusa (e.g., changes title "Small" → "Small (Limited Edition)")
2. Event: `product-variant.updated` emitted
3. Subscriber triggers `updatePayloadProductVariantsWorkflow`
4. Workflow:
   - Queries Medusa for variant details
   - Queries linked Payload product
   - Updates variant array in Payload
   - Sends PATCH request to Payload API
5. Variant title updated in Payload (< 1 second)

**Important**: Variant prices and inventory stay in Medusa only.

### Q: Can I sync existing products in bulk?

**A**: **Yes!** Use the manual sync:

1. Open Medusa Admin
2. Navigate to Settings → Payload
3. Click "Sync Products to Payload"
4. System processes all products without `payload_id` in metadata
5. Syncs in batches of 1000

**Performance**: ~500 products in ~5 minutes (depends on product complexity)

### Q: What if sync fails?

**A**: **Workflows are transactional:**

1. If Payload API call fails, workflow automatically rolls back
2. Product metadata won't be updated
3. Retry mechanism (Medusa's workflow engine handles this)
4. Check logs for specific error

**Manual recovery:**
```bash
# Re-trigger sync for specific product
POST /admin/payload/sync/products

# Or delete and recreate product in Medusa
```

## Technical Questions

### Q: Do I need two separate databases?

**A**: **Yes, recommended:**

```
Medusa DB: postgres://localhost:5432/medusa
Payload DB: postgres://localhost:5432/payload
```

**Why separate?**
- Independent scaling
- Different backup strategies
- Data isolation (content changes can't corrupt commerce)
- Can use different PostgreSQL versions if needed

**Can they be on same server?** Yes, just different database names.

### Q: How does the linking work between Medusa and Payload?

**A**: **Logical link via IDs:**

1. **In Medusa**: Product metadata contains `payload_id`
   ```json
   {
     "id": "prod_123",
     "metadata": {
       "payload_id": "67abc"
     }
   }
   ```

2. **In Payload**: Product contains `medusa_id`
   ```json
   {
     "id": "67abc",
     "medusa_id": "prod_123"
   }
   ```

3. **Link Definition** (`src/links/product-payload.ts`):
   ```typescript
   defineLink(
     { linkable: ProductModule.linkable.product, field: "id" },
     { linkable: { serviceName: PAYLOAD_MODULE, alias: "payload_product" } }
   )
   ```

4. **Query with link**:
   ```typescript
   // Request:
   fields: "*payload_product"
   
   // Medusa automatically:
   // 1. Reads product.metadata.payload_id
   // 2. Calls PayloadModuleService.list({ product_id: "prod_123" })
   // 3. Fetches from Payload API
   // 4. Merges and returns
   ```

### Q: What's the performance impact?

**A**: **Minimal, but noticeable:**

**Without Payload:**
- Storefront → Medusa: ~50ms
- 1 database query

**With Payload:**
- Storefront → Medusa → Payload → Response: ~100-150ms
- 2 database queries (Medusa + Payload)
- 1 HTTP call (Medusa to Payload)

**Mitigation strategies:**
- Enable Next.js caching (`cache: "force-cache"`)
- Use CDN for images
- Keep Medusa and Payload on same network (low latency)
- Consider Redis cache for Payload data

### Q: How do I handle images?

**A**: **Multiple approaches:**

**Option 1: Upload to Payload (recommended)**
- Content manager uploads in Payload admin
- Stored in Payload's media collection
- Automatic resizing (thumbnail, card, full)
- Served from Next.js public directory

**Option 2: Paste URLs from S3**
- Configure allowed domains in Media collection
- Paste image URL from existing Medusa images
- Payload imports and optimizes

**Option 3: Custom integration**
- Sync Medusa images to Payload on product creation
- Requires custom workflow step

### Q: What about SEO?

**A**: **Payload provides rich SEO management:**

```typescript
// In Payload Products collection
seo: {
  meta_title: "Blue Hoodie - Organic Cotton | YourBrand",
  meta_description: "Shop our premium blue hoodie...",
  meta_keywords: "hoodie, blue hoodie, organic cotton"
}
```

**Rendering in storefront:**
```tsx
// app/products/[handle]/page.tsx
export async function generateMetadata({ params }) {
  const product = await getProduct(params.handle)
  
  return {
    title: product.payload_product?.seo?.meta_title || product.title,
    description: product.payload_product?.seo?.meta_description,
    keywords: product.payload_product?.seo?.meta_keywords
  }
}
```

**Future enhancements** (you can add):
- Open Graph tags
- Twitter cards
- Schema.org structured data
- Canonical URLs

## Use Case Questions

### Q: Should I use this for my store?

**A**: **Use the decision matrix:**

**✅ Use if you have:**
- Content-heavy products (fashion, home goods, artisanal)
- Dedicated content team
- Budget for infrastructure (~2x cost)
- Need for rich text, images, SEO
- 100+ products or growing catalog

**❌ Skip if you have:**
- Simple product descriptions
- Small team (<3 people)
- Tight budget/timeline
- Commodity products
- < 50 products

**See**: [Pros & Cons - Decision Matrix](./05-pros-cons.md#decision-matrix)

### Q: Can I start simple and add Payload later?

**A**: **Absolutely! Recommended path:**

**Phase 1 (Months 1-3): Medusa Only**
- Launch with Medusa
- Use markdown in description field
- Get product-market fit
- Learn what content needs you have

**Phase 2 (Months 3-6): Evaluate**
- Is content management painful?
- Is team frustrated?
- Is SEO suffering?
- Ready for complexity?

**Phase 3 (Month 6+): Add Payload**
- Set up integration
- Bulk sync existing products
- Gradually enrich content
- Measure impact on conversions

### Q: What about multi-language support?

**A**: **Two systems, two strategies:**

**Medusa**: Built-in multi-currency, multi-region
- Prices per currency
- Regions per country
- Shipping per region

**Payload**: Localization plugin available
```typescript
// Payload config
localization: {
  locales: ['en', 'es', 'fr'],
  defaultLocale: 'en'
}

// Products with localized content
{
  title: {
    en: "Blue Hoodie",
    es: "Sudadera Azul",
    fr: "Sweat à Capuche Bleu"
  }
}
```

**Integration**: Locale selection in storefront determines which content to fetch from Payload

### Q: Can I customize Payload collections?

**A**: **Yes, highly customizable!**

**Add custom fields:**
```typescript
// Products.ts
fields: [
  // ... existing fields
  {
    name: 'brand_story',
    type: 'richText',
    label: 'Brand Story'
  },
  {
    name: 'sustainability',
    type: 'group',
    fields: [
      { name: 'eco_friendly', type: 'checkbox' },
      { name: 'certifications', type: 'array' }
    ]
  }
]
```

**Add new collections:**
```typescript
// src/collections/BlogPosts.ts
export const BlogPosts: CollectionConfig = {
  slug: 'blog-posts',
  fields: [
    { name: 'title', type: 'text' },
    { name: 'content', type: 'richText' },
    { 
      name: 'related_products',
      type: 'relationship',
      relationTo: 'products',
      hasMany: true
    }
  ]
}
```

## Deployment Questions

### Q: How do I deploy this in production?

**A**: **Two separate deployments:**

**Medusa Backend:**
- Deploy to Railway, Digital Ocean, AWS, etc.
- PostgreSQL database
- Set environment variables
- Domain: `api.yourstore.com`

**Next.js + Payload:**
- Deploy to Vercel, Netlify, or traditional hosting
- PostgreSQL database (can be same server, different DB)
- Set environment variables
- Domain: `www.yourstore.com`

**Network:**
- Ensure Medusa can reach Payload (set `PAYLOAD_SERVER_URL`)
- Configure CORS properly
- Use production API keys

**See**: [Implementation Guide - Production Deployment](./04-implementation.md#production-deployment)

### Q: What's the cost increase?

**A**: **Roughly 2x for small scale:**

**Infrastructure:**
- Servers: 1 → 2 (+$25-50/month)
- Databases: 1 → 2 (+$15-25/month)
- Monitoring: 1 system → 2 systems
- **Total**: ~$40-75/month additional

**Team time:**
- Setup: ~40 hours (one-time)
- Maintenance: ~5 hours/month
- Training: ~4 weeks for team

**Offset by:**
- Reduced developer time for content changes
- Faster content updates = more sales
- Better SEO = more organic traffic

### Q: How do I monitor the integration?

**A**: **Monitor both systems:**

**Medusa:**
- Workflow execution logs
- Event bus health
- API response times
- Database queries

**Payload:**
- API response times
- Database queries
- Storage usage (media)
- Authentication failures

**Integration health:**
- Successful sync rate
- Failed workflows (alerts)
- API connectivity (Medusa → Payload)
- Product count consistency

**Tools:**
- Application: DataDog, New Relic, Sentry
- Infrastructure: CloudWatch, Prometheus
- Custom: Sync status dashboard

## Still Have Questions?

1. **Check the full documentation**:
   - [Overview](./01-overview.md)
   - [Architecture](./02-architecture.md)
   - [Data Flow](./03-data-flow.md)
   - [Implementation Guide](./04-implementation.md)
   - [Pros & Cons](./05-pros-cons.md)

2. **Community support**:
   - [Medusa Discord](https://discord.gg/medusajs)
   - [Payload Discord](https://discord.gg/payload)

3. **Official docs**:
   - [Medusa Documentation](https://docs.medusajs.com)
   - [Payload Documentation](https://payloadcms.com/docs)

4. **Example code**:
   - See `medusa-examples/payload-integration` for complete working example

