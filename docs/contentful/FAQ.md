# Frequently Asked Questions

## General Questions

### Q: What is the Contentful integration?

**A:** The Contentful integration connects Medusa (e-commerce backend) with Contentful (enterprise CMS) to manage localized, rich product content separately from commerce operations. It provides two-way sync between the systems.

---

### Q: Do I need Contentful for my Medusa store?

**A:** No, Contentful is optional. Use it if you need:
- Multi-language content management
- Rich editorial content
- Professional content workflows
- Managed CMS infrastructure

Skip it if you have simple content needs or want to avoid SaaS costs.

---

### Q: Which Medusa version is supported?

**A:** Both Medusa v1 and v2 are supported, but with different implementations:
- **v1**: Uses `medusa-plugin-contentful` (legacy plugin system)
- **v2**: Uses updated integration with workflows and modules

This documentation primarily covers v1. Check [Medusa v2 docs](https://docs.medusajs.com/resources/integrations/guides/contentful) for v2 implementation.

---

### Q: Is Contentful free?

**A:** Contentful has a free "Community" tier with limitations:
- 2 spaces
- 48 API req/s
- 5 users
- 25,000 records

For production, most businesses need the Team plan ($489/month) or Enterprise (custom pricing).

---

### Q: Can I migrate from Contentful to another CMS later?

**A:** Yes, but it's complex:
- Export content via Contentful API
- Restructure data for new CMS
- Migrate media assets
- Rewrite storefront integration code
- Test thoroughly

Estimated effort: 100-200 hours. Consider vendor lock-in before committing.

---

## Technical Questions

### Q: How does sync work? (Medusa → Contentful)

**A:** Event-driven synchronization:
1. Product created/updated in Medusa
2. Event emitted to Redis event bus
3. Contentful subscriber catches event
4. Fetches full product data from Medusa
5. Transforms to Contentful format
6. Posts to Contentful Management API
7. Entry created/updated in Contentful

All automatic, no manual intervention.

---

### Q: How does reverse sync work? (Contentful → Medusa)

**A:** Webhook-based synchronization:
1. Content updated in Contentful UI
2. Entry published
3. Contentful fires webhook to Medusa
4. Medusa receives POST request
5. Validates payload
6. Updates product in Medusa database

Requires configured webhook and publicly accessible Medusa backend.

---

### Q: What data syncs automatically?

**A:** Medusa → Contentful (automatic):
- Products (title, description, handle, metadata)
- Variants (SKU, options, inventory info)
- Collections (name, handle, product relationships)
- Categories (name, hierarchy)
- Regions (currency, countries, providers)

Contentful → Medusa (via webhook):
- Basic fields only (title, description)
- Localized content stays in Contentful

---

### Q: Can I customize which fields sync?

**A:** Yes, using the `custom_<TYPE>_fields` option:

```javascript
{
  resolve: "medusa-plugin-contentful",
  options: {
    // ...
    custom_product_fields: {
      title: "productName",  // Map Medusa 'title' to Contentful 'productName'
      subtitle: "tagline",
    },
  },
}
```

---

### Q: How do I handle localization?

**A:** Contentful's built-in localization:

1. **Configure locales** in Contentful (en-US, es-ES, de-DE, etc.)
2. **Mark fields as localized** in content type (title, description, etc.)
3. **Add translations** for each locale in Contentful UI
4. **Fetch by locale** in storefront:

```typescript
const content = await contentful.getEntries({
  content_type: 'product',
  'fields.medusaId': productId,
  locale: 'es-ES'  // Spanish
});
```

---

### Q: Do I need Redis?

**A:** Yes, Redis is required for:
- Event bus (Medusa → Contentful sync)
- Ignore flags (prevent sync loops)
- Optional: Caching Contentful responses

---

### Q: Can I use this with Medusa v2?

**A:** Yes, but the integration is different:
- v2 uses workflows and modules (not plugins)
- Updated integration available in Medusa v2 docs
- This documentation covers v1 primarily

For v2, see: [Medusa v2 Contentful Integration](https://docs.medusajs.com/resources/integrations/guides/contentful)

---

### Q: How do I test locally with webhooks?

**A:** Webhooks require a public URL. For local development:

**Option 1: ngrok (Recommended)**
```bash
# Start Medusa
npx medusa develop

# In another terminal
ngrok http 9000

# Use ngrok URL in Contentful webhook:
# https://abc123.ngrok.io/hooks/contentful
```

**Option 2: Cloudflare Tunnel**
```bash
cloudflare tunnel --url http://localhost:9000
```

**Option 3: Skip webhooks in development**
- Test Medusa → Contentful sync only
- Test Contentful → Medusa manually via API

---

## Troubleshooting

### Q: Products aren't syncing to Contentful. What's wrong?

**A:** Check these common issues:

1. **Redis not running**
```bash
redis-cli ping
# Should return: PONG
```

2. **Plugin not loaded**
```bash
# Check medusa-config.js
# Verify plugin is in plugins array
```

3. **Wrong API token**
```bash
# Need MANAGEMENT token (CFPAT-...)
# NOT delivery token
```

4. **Content type doesn't exist**
```bash
# Create "product" content type in Contentful
# With "medusaId" field
```

5. **Check Medusa logs**
```bash
npx medusa develop
# Look for errors after creating product
```

---

### Q: Webhook not working (Contentful → Medusa)

**A:** Troubleshoot:

1. **Check webhook activity log**
```
Contentful → Settings → Webhooks → [Your webhook] → Activity
Look for delivery status (should be 200 OK)
```

2. **Verify URL is publicly accessible**
```bash
curl -X POST https://your-backend.com/hooks/contentful \
  -H "Content-Type: application/json" \
  -d '{"test": "payload"}'
  
# Should NOT return connection refused
```

3. **Check webhook secret (if configured)**
```bash
# Verify secret matches in both systems
echo $CONTENTFUL_WEBHOOK_SECRET
```

4. **Check Medusa logs**
```bash
# Should see:
# info: Webhook received from Contentful
```

---

### Q: Getting "Rate limit exceeded" errors

**A:** Solutions:

1. **Check your tier**
```
Free: 10 req/s
Team: 98 req/s
```

2. **Implement caching**
```typescript
// Cache Contentful responses in Redis
const cached = await redis.get(`product:${id}`);
if (cached) return JSON.parse(cached);

const content = await contentful.getEntries({...});
await redis.set(`product:${id}`, JSON.stringify(content), 'EX', 3600);
```

3. **Upgrade plan**
```
Settings → Plan & billing → Upgrade
```

4. **Use Contentful's CDN (Delivery API)**
```typescript
// Don't use Management API for reads
// Use Delivery API (cdn.contentful.com)
const client = contentful.createClient({
  accessToken: DELIVERY_TOKEN  // Not management token
});
```

---

### Q: Localized content not showing on storefront

**A:** Check:

1. **Translations exist in Contentful**
```
Open entry → Switch locale → Verify content
```

2. **Fetching with correct locale**
```typescript
// Make sure to specify locale
const content = await contentful.getEntries({
  locale: 'es-ES'  // ← Must specify
});
```

3. **Field is marked as localized**
```
Contentful → Content model → Product → [Field]
Settings → Enable localization for this field ✓
```

4. **Fallback locale configured**
```
Settings → Locales → es-ES → Fallback: en-US
```

---

### Q: Sync creates duplicates in Contentful

**A:** This shouldn't happen. The plugin checks for existing entries:

```typescript
// Plugin checks before creating
const existing = await contentful.getEntries({
  content_type: 'product',
  'fields.medusaId': productId
});

if (existing.items.length > 0) {
  // Update existing
} else {
  // Create new
}
```

If you have duplicates:
1. Manually delete duplicates in Contentful
2. Verify `medusaId` field is unique in content type
3. Check Medusa logs for errors during sync

---

### Q: Images from Contentful not loading

**A:** Check:

1. **Assets are published**
```
Contentful → Assets → [Image] → Status: Published
```

2. **Using correct URL**
```typescript
// Correct
const imageUrl = content.fields.images[0].fields.file.url;

// Add protocol if missing
const fullUrl = imageUrl.startsWith('//') 
  ? `https:${imageUrl}` 
  : imageUrl;
```

3. **CORS configured (if loading from JavaScript)**
```
Contentful automatically sets CORS headers
Should work out of the box
```

---

## Performance Questions

### Q: How fast is the Medusa → Contentful sync?

**A:** Typically very fast:
- Event emission: 1-5ms
- Subscriber processing: 10-50ms
- API call to Contentful: 100-300ms
- **Total: 150-400ms** per product

Bulk sync (1000 products): ~10-20 minutes.

---

### Q: Does using Contentful slow down my storefront?

**A:** It adds one extra API call:

**Without Contentful:**
```
Storefront → Medusa API (products)
Response time: 100ms
```

**With Contentful (dual fetch):**
```
Storefront → Medusa API (commerce): 100ms
Storefront → Contentful API (content): 80ms (CDN-cached)
Total (parallel): ~100ms (if fetched in parallel)
```

**Recommendation:** Use server-side parallel fetching or implement caching.

---

### Q: How much does Contentful's CDN improve performance?

**A:** Significant global improvement:

```
Without CDN (self-hosted):
  - US users: 50ms ✓
  - EU users: 200ms
  - Asia users: 400ms
  
With Contentful CDN:
  - US users: 50ms ✓
  - EU users: 50ms ✓
  - Asia users: 50ms ✓
```

Contentful uses Fastly's global CDN (150+ POPs worldwide).

---

### Q: Can I cache Contentful responses?

**A:** Yes, highly recommended:

```typescript
// Redis caching
const cacheKey = `product:${id}:${locale}`;
const cached = await redis.get(cacheKey);

if (cached) {
  return JSON.parse(cached);
}

const content = await contentful.getEntries({...});
await redis.set(cacheKey, JSON.stringify(content), 'EX', 3600);  // 1 hour

return content;
```

Or use Contentful's Sync API for efficient bulk updates.

---

## Cost & Pricing Questions

### Q: How much does Contentful cost?

**A:** Pricing tiers:

| Tier | Price | Limits |
|------|-------|--------|
| Community (Free) | $0 | 2 spaces, 48 req/s, 5 users |
| Team | $489/month | 3 spaces, 98 req/s, 10 users |
| Enterprise | Custom | Unlimited, SLA, support |

Plus potential overages for bandwidth.

---

### Q: When will I hit the free tier limits?

**A:** Common scenarios:

**Product Catalog:**
- 25,000 records limit = ~2,500 products with variants
- Enough for small stores

**API Calls:**
- 10 req/s = 36,000 requests/hour
- 100 visitors/hour × 3 API calls = 300 calls/hour ✓
- 1,000 visitors/hour × 3 API calls = 3,000 calls/hour ✓
- 10,000 visitors/hour = might hit limits ⚠️

**Users:**
- 5 users = small team
- Need Team plan for 6+ users

---

### Q: Are there any hidden costs?

**A:** Watch out for:

1. **Bandwidth overages**
   - Team plan includes 1TB/month
   - $0.12/GB overage
   - High-traffic sites can hit this

2. **API call patterns**
   - Inefficient queries = more API calls
   - No caching = more API calls
   - Real-time updates = more API calls

3. **Asset storage**
   - Generally included
   - Very large files may incur charges

---

### Q: Is Contentful worth the cost?

**A:** Depends on your needs:

**Worth it if:**
- You need enterprise localization
- Global audience (CDN saves money)
- Want zero DevOps (saves labor costs)
- Content team needs professional tools

**Not worth it if:**
- Simple content needs
- Single language only
- Limited budget
- Happy with open-source alternatives

**Cost comparison:**
```
Contentful (Team): $489/month
vs.
Strapi (self-hosted): 
  - Server: $50/month
  - Database: $25/month
  - DevOps: $200/month (4 hours)
  Total: $275/month
  
Savings with Strapi: $214/month
But: More DevOps work, no CDN, basic localization
```

---

## Migration Questions

### Q: Can I import existing products into Contentful?

**A:** Yes, two methods:

**Method 1: Automatic (on first start)**
```javascript
// In medusa-config.js
{
  resolve: "medusa-plugin-contentful",
  options: {
    // ...
    sync_on_init: true  // Sync all products on startup
  }
}
```

**Method 2: Manual bulk sync**
```bash
# Via Medusa API or script
node scripts/sync-to-contentful.js
```

---

### Q: How do I migrate from Strapi to Contentful?

**A:** Migration steps:

1. **Export from Strapi**
```bash
# Export Strapi content
node scripts/export-strapi.js > strapi-export.json
```

2. **Transform data**
```bash
# Map Strapi format to Contentful format
node scripts/transform-to-contentful.js
```

3. **Create content types in Contentful**
```bash
# Use migration scripts
contentful space migration scripts/create-content-types.js
```

4. **Import to Contentful**
```bash
# Use Contentful Import API
node scripts/import-to-contentful.js
```

5. **Update Medusa config**
```javascript
// Replace strapi plugin with contentful
```

6. **Update storefront**
```typescript
// Replace Strapi API calls with Contentful
```

Estimated time: 40-80 hours depending on complexity.

---

### Q: What happens to my data if I stop using Contentful?

**A:** You own your data:

1. **Export content**
```bash
# Via Contentful Management API
contentful space export --space-id $SPACE_ID
```

2. **Download assets**
```bash
# Download all media files
node scripts/download-contentful-assets.js
```

3. **Keep using exported data**
```bash
# Import to new CMS or serve statically
```

But: You lose real-time updates, CDN, localization features.

---

## What's Next?

- **[Implementation Guide](./04-implementation.md)** - Set it up
- **[Pros & Cons](./05-pros-cons.md)** - Evaluate trade-offs
- **[Data Flow](./03-data-flow.md)** - See it in action
- **[SUMMARY](./SUMMARY.md)** - Quick reference

