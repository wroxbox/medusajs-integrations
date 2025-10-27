# Sanity CMS Integration - FAQ

Frequently asked questions about Sanity + Medusa integration.

## General Questions

### Q: What is Sanity?

**A**: Sanity is a headless CMS (Content Management System) platform that provides:
- Real-time content editing
- Structured content modeling
- Global CDN delivery
- GROQ query language
- Collaboration features
- API-first architecture

Think of it as a specialized database for content with a powerful editing interface.

### Q: Why use Sanity with Medusa?

**A**: Main reasons:
1. **Localization** - Native multi-language support
2. **Content Team Independence** - Non-developers can manage content
3. **Rich Content Features** - Structured data, relationships, versioning
4. **Real-time Collaboration** - Multiple editors working simultaneously
5. **Global Performance** - CDN delivery worldwide

See [Pros & Cons](./05-pros-cons.md) for detailed comparison.

### Q: Is Sanity free?

**A**: Sanity has a generous free tier:
- 3 users
- 10,000 documents
- 100,000 API requests/month
- Included CDN

Paid plans start at $99/month for more users and requests. See [sanity.io/pricing](https://www.sanity.io/pricing).

### Q: Can I self-host Sanity?

**A**: No, Sanity is a managed SaaS only. If you need self-hosting, consider [Payload integration](../payload/) instead.

### Q: What's the difference between Sanity and Payload integrations?

**A**:

| Feature | Sanity | Payload |
|---------|--------|---------|
| Hosting | External SaaS | Self-hosted (embedded) |
| Localization | Native | Custom implementation |
| Data Fetching | Dual (Medusa + Sanity) | Single (via Medusa) |
| Real-time Collab | Yes | No |
| Cost | Usage-based | Free (hosting only) |
| Vendor Lock-in | Yes | No |

**Choose Sanity if**: You need localization and real-time collaboration  
**Choose Payload if**: You want self-hosting and integrated data fetching

---

## Technical Questions

### Q: Does content sync both ways?

**A**: No, only **Medusa → Sanity** (one-way).

**What syncs**:
- Product creation → Creates document in Sanity
- Product title update → Updates title in Sanity

**What doesn't sync back**:
- Content edits in Sanity → Stay in Sanity only
- Localized specs → Sanity only
- Addons → Sanity only

**Why one-way?** Medusa is the source of truth for commerce data. Sanity is for content enhancement only.

### Q: What data is stored in Sanity?

**A**: Minimal product data for content purposes:

```typescript
{
  _id: "prod_123",              // Same as Medusa product ID
  _type: "product",
  title: "Product Title",       // Synced from Medusa
  specs: [                      // Managed in Sanity only
    {
      lang: "en",
      title: "English Title",
      content: "English description..."
    },
    // ... more languages
  ],
  addons: {                     // Managed in Sanity only
    title: "Complete Your Look",
    products: [/*refs*/]
  }
}
```

**NOT stored in Sanity**:
- Pricing
- Inventory
- Variants
- Images (Medusa handles)
- Collections/Categories

### Q: How does the storefront fetch data?

**A**: **Dual fetching** - two separate API calls:

```typescript
// 1. Fetch commerce data from Medusa
const product = await fetchFromMedusa(handle)

// 2. Fetch content from Sanity
const sanity = await sanityClient.getDocument(product.id)

// 3. Merge and display
return <ProductPage product={product} sanity={sanity} />
```

**Why not through Medusa?** Direct Sanity fetch is faster (CDN) and allows complex GROQ queries.

### Q: What happens if Sanity is down?

**A**: Your site continues to work:

1. **Commerce operations** (add to cart, checkout) still work (Medusa)
2. **Content from Sanity** won't load
3. **Fallback**: Display Medusa product descriptions

**Best Practice**: Implement fallback logic:

```typescript
const description = sanity?.content || product.description
```

### Q: Can I query products from Sanity?

**A**: Yes! Use GROQ queries:

```typescript
// Get product by ID
const product = await client.fetch(`*[_id == $id][0]`, { id })

// Get products by language
const products = await client.fetch(`
  *[_type == "product"] {
    _id,
    title,
    "spec": specs[lang == $lang][0]
  }
`, { lang: "es" })

// Get product with addons
const productWithAddons = await client.fetch(`
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
`, { id })
```

### Q: How do I add custom fields to Sanity?

**A**: Update the Sanity schema:

**File**: `storefront/src/sanity/schemaTypes/documents/product.ts`

```typescript
const productSchema = {
  fields: [
    // ... existing fields
    {
      name: "materials",
      type: "array",
      title: "Materials",
      of: [{ type: "string" }]
    },
    {
      name: "care_instructions",
      type: "text",
      title: "Care Instructions"
    },
    {
      name: "size_guide",
      type: "object",
      fields: [
        { name: "chart_image", type: "image" },
        { name: "notes", type: "text" }
      ]
    }
  ]
}
```

Restart your dev server to see changes in Studio.

### Q: Can I use Sanity images?

**A**: Yes! Add image fields to schema:

```typescript
{
  name: "gallery",
  type: "array",
  of: [{ type: "image" }]
}
```

Use `next-sanity` image builder:

```typescript
import imageUrlBuilder from '@sanity/image-url'

const builder = imageUrlBuilder(client)

function urlFor(source) {
  return builder.image(source)
}

// In component
<img src={urlFor(product.gallery[0]).width(800).url()} />
```

---

## Implementation Questions

### Q: Do I need to create a Sanity account?

**A**: Yes, go to [sanity.io](https://www.sanity.io) and sign up. Free tier is sufficient to start.

### Q: What are the steps to integrate?

**A**: High-level:

1. Create Sanity project
2. Get API token with Editor permissions
3. Install Sanity module in Medusa
4. Configure environment variables
5. Set up workflows and subscribers
6. Install Sanity in Next.js storefront
7. Configure Sanity Studio
8. Update product pages to fetch from Sanity
9. Test sync

See [Implementation Guide](./04-implementation.md) for detailed steps.

### Q: How long does implementation take?

**A**:

| Phase | Time |
|-------|------|
| Sanity project setup | 30 minutes |
| Medusa backend integration | 2-3 days |
| Next.js + Studio setup | 1 day |
| Testing | 1 day |
| **Total** | **3-5 days** |

Assumes developer familiar with Medusa and TypeScript.

### Q: Can I use with existing Medusa project?

**A**: Yes! Just copy the source files:

1. `src/modules/sanity/`
2. `src/workflows/sanity-sync-products/`
3. `src/subscribers/sanity-product-sync.ts`
4. `src/links/product-sanity.ts`
5. `src/api/admin/sanity/`

Then configure `medusa-config.ts` and `.env`.

### Q: Do I need to migrate existing products?

**A**: Not initially. Products sync automatically when:
- Created
- Updated

To sync existing catalog:

```bash
curl -X POST http://localhost:9000/admin/sanity/syncs
```

This triggers bulk sync of all products.

---

## Troubleshooting

### Q: Product created but not syncing to Sanity?

**Check**:

1. **Medusa logs** - Look for errors:
   ```bash
   # In Medusa directory
   npm run dev
   # Check output for "Connected to Sanity"
   ```

2. **Environment variables**:
   ```bash
   echo $SANITY_API_TOKEN
   echo $SANITY_PROJECT_ID
   ```

3. **API token permissions** - Must be "Editor", not "Viewer"

4. **CORS settings** - Add `http://localhost:9000` to Sanity CORS

5. **Test Sanity connection**:
   ```bash
   curl https://YOUR_PROJECT_ID.api.sanity.io/v2024-01-01/data/query/production \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -d 'query=*[_type=="product"][0]'
   ```

### Q: Sanity Studio not loading at /studio?

**Solutions**:

1. **Clear Next.js cache**:
   ```bash
   rm -rf .next
   npm run dev
   ```

2. **Check route exists**:
   - File: `src/app/studio/[[...tool]]/page.tsx` should exist

3. **Verify environment variables**:
   ```bash
   echo $NEXT_PUBLIC_SANITY_PROJECT_ID
   ```

4. **Check browser console** for errors

5. **Verify `sanity.config.ts`** exists in root directory

### Q: Sanity content not showing on storefront?

**Debug checklist**:

1. **Verify product exists in Sanity**:
   - Open Studio at `/studio`
   - Check product is there

2. **Check storefront code**:
   ```typescript
   // Should have something like:
   const sanity = await client.getDocument(productId)
   ```

3. **Browser Network tab**:
   - Look for request to `*.api.sanity.io`
   - Check response

4. **Console errors** - Any JavaScript errors?

5. **Test Sanity query directly**:
   ```bash
   curl https://YOUR_PROJECT_ID.api.sanity.io/v2024-01-01/data/query/production \
     -d 'query=*[_id=="prod_YOUR_ID"][0]'
   ```

### Q: "API token is invalid" error?

**Causes**:
- Token copied incorrectly (extra spaces)
- Token doesn't have Editor permissions
- Token was deleted/regenerated

**Solution**:
1. Go to [manage.sanity.io](https://manage.sanity.io)
2. Navigate to your project → API → Tokens
3. Create new token with **Editor** permissions
4. Update `.env` with new token
5. Restart Medusa

### Q: Workflow execution failed?

**Check**:

1. **View workflow executions**:
   ```bash
   curl http://localhost:9000/admin/sanity/syncs
   ```

2. **Common failures**:
   - Invalid product data
   - Sanity API rate limit
   - Network timeout
   - Schema validation errors

3. **Retry failed sync**:
   ```bash
   curl -X POST http://localhost:9000/admin/sanity/documents/PRODUCT_ID/sync
   ```

### Q: Data out of sync between Medusa and Sanity?

**Causes**:
- Workflow failed silently
- Content edited in both places
- Sync not triggered

**Solution - Bulk re-sync**:
```bash
curl -X POST http://localhost:9000/admin/sanity/syncs
```

This re-syncs ALL products.

---

## Performance & Scaling

### Q: Does Sanity slow down my site?

**A**: Minimal impact if implemented correctly:

**Latency**:
- Medusa API: ~100-200ms
- Sanity API: ~50-100ms (CDN-cached)
- **Parallel fetch**: ~150-250ms total

**Optimization tips**:
1. Use `Promise.all()` for parallel fetching
2. Cache aggressively
3. Implement ISR (Incremental Static Regeneration)
4. Use CDN for static assets

**Example**:
```typescript
// Good: Parallel
const [medusa, sanity] = await Promise.all([
  fetchMedusa(),
  fetchSanity()
]) // ~150ms

// Bad: Sequential
const medusa = await fetchMedusa() // ~150ms
const sanity = await fetchSanity()  // +100ms = 250ms total
```

### Q: What are Sanity's rate limits?

**A**:

**Free Tier**:
- 100,000 API requests/month
- ~115 requests/hour average
- 10 requests/second burst

**Growth Plan ($99/month)**:
- 500,000 requests/month
- ~575 requests/hour average
- Higher burst limits

**Typical Usage**:
- Product sync: 1 request per product
- Storefront view: 1-2 requests per page load
- Example: 1000 products + 50k monthly visitors = ~50k-100k requests

### Q: Can Sanity handle large catalogs?

**A**: Yes!

**Tested scales**:
- 10,000+ documents: No issues
- 100,000+ documents: Works well
- 1,000,000+ documents: Contact Sanity for enterprise

**Considerations**:
- Bulk syncs take longer with large catalogs
- Workflow batches 200 products at a time
- API request costs increase

### Q: How do I optimize costs?

**Strategies**:

1. **Cache aggressively**:
   ```typescript
   cache: 'force-cache',
   next: { revalidate: 3600 } // 1 hour
   ```

2. **Use projections** (fetch only needed fields):
   ```groq
   *[_id == $id][0] {
     _id,
     "spec": specs[lang == $lang][0].content
   }
   ```

3. **Minimize admin queries** - Use Medusa admin for most operations

4. **Monitor usage** - Check Sanity dashboard regularly

5. **Batch operations** - Don't sync one at a time

---

## Content Management

### Q: Can non-developers use Sanity Studio?

**A**: Yes! Sanity Studio is designed for content editors:
- No coding required
- Intuitive interface
- Real-time preview
- Undo/redo

**Training time**: 30-60 minutes for basic usage.

### Q: Can multiple people edit at the same time?

**A**: Yes! Sanity has real-time collaboration:
- See who's editing what
- Live cursor presence
- No merge conflicts
- Changes sync instantly

### Q: Can I preview changes before publishing?

**A**: Yes, but requires custom setup. Sanity supports:
- Draft content (unpublished)
- Custom preview panes
- Integration with preview URLs

See [Sanity docs on preview](https://www.sanity.io/docs/previews).

### Q: How do I add a new language?

**A**: Just add a new spec in Sanity Studio:

1. Open product in Studio
2. Click "Add item" in specs array
3. Fill in:
   - Lang: "de" (for German)
   - Title: German title
   - Content: German description
4. Publish

Then in storefront, detect user language:

```typescript
const userLang = getUserLanguage() // "de"
const spec = sanity.specs?.find(s => s.lang === userLang)
```

---

## Business & Operations

### Q: What's the ROI of Sanity integration?

**A**: Depends on your situation:

**Positive ROI scenarios**:
- Multi-language required (save dev time on custom i18n)
- Frequent content updates (save dev time)
- Dedicated content team (increase velocity)

**Example calculation**:
```
Without Sanity:
- Content update = 2 hours (dev time) × $100/hour = $200
- 20 updates/month = $4,000/month

With Sanity:
- Sanity cost = $99/month
- Content manager can update in 15 minutes
- Savings = $3,900/month
```

**Negative ROI scenarios**:
- Single language, simple products
- Rare content updates
- No content team

### Q: How do I manage user access?

**A**: Sanity project members:

1. Go to [manage.sanity.io](https://manage.sanity.io)
2. Select project → Members
3. Invite users with roles:
   - **Administrator**: Full access
   - **Editor**: Content editing only
   - **Viewer**: Read-only

Free tier includes 3 users. Additional users $10/month each.

### Q: What if we want to migrate away from Sanity?

**A**: Content is exportable:

1. **Export data**:
   ```bash
   sanity dataset export production export.ndjson
   ```

2. **Import to Medusa** (write script to parse and import)

3. **Remove Sanity module** from Medusa

4. **Update storefront** to use Medusa descriptions only

**Difficulty**: Medium (1-2 weeks work)

---

## Next Steps

- **New to integration?** Start with [Overview](./01-overview.md)
- **Ready to implement?** Follow [Implementation Guide](./04-implementation.md)
- **Need to decide?** Review [Pros & Cons](./05-pros-cons.md)
- **Quick reference?** Check [Summary](./SUMMARY.md)

## Still Have Questions?

- [Medusa Discord](https://discord.gg/medusajs)
- [Sanity Slack](https://slack.sanity.io)
- [Sanity Documentation](https://www.sanity.io/docs)
- [Medusa Documentation](https://docs.medusajs.com)

