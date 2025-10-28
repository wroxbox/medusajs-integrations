# Data Flow: Step-by-Step Scenarios

This document walks through common integration scenarios with detailed step-by-step examples.

## Scenario 1: Creating a New Product

**User Story:** A product manager creates a new product in Medusa Admin. The product structure automatically syncs to Contentful, and the content team adds localized content.

### Step-by-Step Flow

#### Phase 1: Create in Medusa (Product Manager)

**1. Open Medusa Admin**
```
Browser → http://localhost:7001
Login as admin
Navigate to Products → New Product
```

**2. Fill Product Form**
```
Title: "Organic Cotton T-Shirt"
Subtitle: "Soft, sustainable, and stylish"
Description: "A comfortable t-shirt made from 100% organic cotton"
Handle: "organic-cotton-tshirt" (auto-generated)
Material: "Organic Cotton"
Weight: 200g
```

**3. Add Variants**
```
Variant 1:
  - Title: "Small / White"
  - SKU: "OCT-S-WHT"
  - Price: $29.99
  - Options: Size=S, Color=White
  
Variant 2:
  - Title: "Medium / White"
  - SKU: "OCT-M-WHT"
  - Price: $29.99
  - Options: Size=M, Color=White
  
Variant 3:
  - Title: "Large / Black"
  - SKU: "OCT-L-BLK"
  - Price: $32.99
  - Options: Size=L, Color=Black
```

**4. Set Collection & Categories**
```
Collection: "Summer Essentials"
Categories: ["Apparel", "T-Shirts", "Organic"]
Tags: ["sustainable", "organic", "cotton"]
```

**5. Upload Thumbnail**
```
Thumbnail: https://cdn.example.com/products/oct-thumb.jpg
```

**6. Click "Publish Product"**
```
POST /admin/products
```

#### Phase 2: Medusa Processing

**7. ProductService Creates Product**
```javascript
// In Medusa backend
const product = await productService.create({
  title: "Organic Cotton T-Shirt",
  subtitle: "Soft, sustainable, and stylish",
  description: "A comfortable t-shirt...",
  handle: "organic-cotton-tshirt",
  // ... all fields
});

// Medusa saves to PostgreSQL
// Generated ID: prod_01HQ5K9ZFWXYZ123ABC
```

**8. Medusa Emits Event**
```javascript
eventBusService.emit("product.created", {
  id: "prod_01HQ5K9ZFWXYZ123ABC",
  // ... other fields
});
```

#### Phase 3: Event Bus Distribution

**9. Redis Pub/Sub**
```
Redis Channel: "product.created"
Message: { id: "prod_01HQ5K9ZFWXYZ123ABC" }
→ Delivered to all subscribers (~1-5ms)
```

#### Phase 4: Contentful Sync (Automatic)

**10. Subscriber Catches Event**
```typescript
// In medusa-plugin-contentful/src/subscribers/contentful.ts
this.eventBus_.subscribe(
  ProductService.Events.CREATED,
  async (data: Product) => {
    await this.contentfulService_.createProductInContentful(
      data.id
    );
  }
);
```

**11. Fetch Full Product Data**
```typescript
const product = await productService.retrieve(
  "prod_01HQ5K9ZFWXYZ123ABC",
  {
    relations: [
      'options',
      'variants',
      'variants.prices',
      'variants.options',
      'type',
      'collection',
      'categories',
      'tags',
      'images'
    ]
  }
);

// Returns complete product object with all relations
```

**12. Transform to Contentful Format**
```typescript
// Create entry for Contentful
const entryFields = {
  medusaId: {
    "en-US": "prod_01HQ5K9ZFWXYZ123ABC"
  },
  title: {
    "en-US": "Organic Cotton T-Shirt"
  },
  subtitle: {
    "en-US": "Soft, sustainable, and stylish"
  },
  description: {
    "en-US": "A comfortable t-shirt made from 100% organic cotton"
  },
  handle: {
    "en-US": "organic-cotton-tshirt"
  },
  thumbnail: {
    "en-US": "https://cdn.example.com/products/oct-thumb.jpg"
  },
  weight: {
    "en-US": 200
  }
  // Note: Initially only default locale (en-US)
  // Other locales will be added by content team
};
```

**13. Authenticate with Contentful**
```typescript
// Use management API token from config
const headers = {
  'Authorization': `Bearer ${process.env.CONTENTFUL_ACCESS_TOKEN}`,
  'Content-Type': 'application/vnd.contentful.management.v1+json',
  'X-Contentful-Content-Type': 'product'
};
```

**14. POST to Contentful Management API**
```typescript
const response = await axios.post(
  `https://api.contentful.com/spaces/${SPACE_ID}/environments/${ENV}/entries`,
  {
    fields: entryFields
  },
  { headers }
);

// Response:
// {
//   sys: {
//     id: "7qVBlCjgXqoqy4wMkKMEwM",  // Contentful entry ID
//     type: "Entry",
//     contentType: { sys: { id: "product" } },
//     version: 1,
//     createdAt: "2024-01-15T10:30:00Z"
//   },
//   fields: {
//     medusaId: { "en-US": "prod_01HQ5K9ZFWXYZ123ABC" },
//     title: { "en-US": "Organic Cotton T-Shirt" },
//     // ... all synced fields
//   }
// }
```

**15. Contentful Stores Entry**
```
Entry created in Contentful Cloud
Status: Draft (not published yet)
Entry ID: 7qVBlCjgXqoqy4wMkKMEwM
```

**16. Optional: Auto-Publish**
```typescript
// If auto_publish enabled in plugin config
await axios.put(
  `https://api.contentful.com/spaces/${SPACE_ID}/environments/${ENV}/entries/${entryId}/published`,
  {},
  { headers }
);

// Entry now published and visible via Delivery API
```

#### Phase 5: Content Enrichment (Content Team)

**17. Content Manager Opens Contentful**
```
Browser → https://app.contentful.com
Login as content editor
Navigate to Content → Product
See newly synced "Organic Cotton T-Shirt"
```

**18. Add Localized Content**

**English (en-US) - Already synced:**
```
Title: "Organic Cotton T-Shirt"
Description: "A comfortable t-shirt made from 100% organic cotton"
```

**Add Spanish (es-ES):**
```
Title: "Camiseta de Algodón Orgánico"
Description: "Una camiseta cómoda hecha de algodón 100% orgánico"

Rich Description (Rich Text Editor):
┌─────────────────────────────────────────────────┐
│ ## Por Qué Te Encantará                         │
│                                                 │
│ Nuestra **Camiseta de Algodón Orgánico** es    │
│ más que ropa—es una declaración sobre el tipo  │
│ de mundo en el que quieres vivir.              │
│                                                 │
│ ### Características:                            │
│ - 100% algodón orgánico certificado GOTS       │
│ - Pre-encogido para un ajuste perfecto        │
│ - Costuras reforzadas para durabilidad         │
│ - Envío neutro en carbono                      │
└─────────────────────────────────────────────────┘

SEO Title: "Camiseta de Algodón Orgánico | Moda Sostenible"
SEO Description: "Camiseta premium de algodón orgánico..."
```

**Add German (de-DE):**
```
Title: "Bio-Baumwolle T-Shirt"
Description: "Ein bequemes T-Shirt aus 100% Bio-Baumwolle"

Rich Description:
┌─────────────────────────────────────────────────┐
│ ## Warum Sie es lieben werden                   │
│                                                 │
│ Unser **Bio-Baumwolle T-Shirt** ist mehr als   │
│ nur Kleidung—es ist ein Statement über die     │
│ Art von Welt, in der Sie leben möchten.        │
│                                                 │
│ ### Merkmale:                                   │
│ - 100% GOTS-zertifizierte Bio-Baumwolle       │
│ - Vorgeschrumpft für perfekte Passform         │
│ - Verstärkte Nähte für Langlebigkeit           │
│ - CO2-neutraler Versand                        │
└─────────────────────────────────────────────────┘
```

**19. Upload High-Res Images**
```
Upload to Contentful Assets:
  1. Main product photo (4000x4000px) → hero_image
  2. Lifestyle shot (3000x2000px) → lifestyle_1
  3. Detail shots (6 images) → detail_images[]
  4. Size chart graphic → size_chart

Contentful automatically:
  - Uploads to CDN
  - Generates multiple sizes
  - Creates WebP versions
  - Provides transformation URLs
```

**20. Add Custom Content Fields**
```
Care Instructions (localized):
  - en-US: "Machine wash cold, tumble dry low"
  - es-ES: "Lavar a máquina en frío, secar en secadora a baja temperatura"
  - de-DE: "Maschinenwäsche kalt, Trockner niedrig"

Sustainability Score: 9.5/10
Made In: "Portugal"
Certifications: ["GOTS", "Fair Trade", "Carbon Neutral"]
Fabric Weight: "180 GSM"
```

**21. Publish Entry**
```
Click "Publish" button
Entry status: Draft → Published
Content now available via Delivery API
```

**Important:** Localized content does NOT sync back to Medusa. It lives only in Contentful.

#### Phase 6: Storefront Display (Customer)

**22. Customer Visits Product Page**
```
Browser → https://store.example.com/products/organic-cotton-tshirt?lang=es
```

**23. Storefront Detects Locale**
```javascript
// Next.js or similar
const locale = req.query.lang || req.headers['accept-language'] || 'en-US';
// Result: 'es' (Spanish)
```

**24. Storefront Fetches Data (Pattern 3: Server-Side Merge)**

```typescript
// Server-side in Next.js getServerSideProps
export async function getServerSideProps({ params, query }) {
  const { handle } = params;
  const locale = query.lang || 'en-US';
  
  // Parallel fetch from both APIs
  const [productRes, contentRes] = await Promise.all([
    // Commerce from Medusa
    fetch(`${MEDUSA_URL}/store/products?handle=${handle}`),
    
    // Content from Contentful (in Spanish)
    fetch(
      `${CONTENTFUL_URL}/entries?content_type=product&fields.handle=${handle}&locale=${locale}`,
      {
        headers: {
          Authorization: `Bearer ${CONTENTFUL_DELIVERY_TOKEN}`
        }
      }
    )
  ]);
  
  const { products } = await productRes.json();
  const { items } = await contentRes.json();
  
  const product = products[0];
  const content = items[0]?.fields;
  
  return {
    props: {
      product,
      content,
      locale
    }
  };
}
```

**25. Render Localized Product Page**
```jsx
<ProductPage locale="es">
  {/* From Medusa (same in all languages) */}
  <Price currency="EUR">€{product.variants[0].price / 100}</Price>
  <Stock>{product.variants[0].inventory_quantity} en stock</Stock>
  
  {/* From Contentful (Spanish) */}
  <ProductTitle>{content.title}</ProductTitle>
  {/* "Camiseta de Algodón Orgánico" */}
  
  <RichDescription>
    <RenderRichText content={content.richDescription} />
    {/* Spanish rich text content */}
  </RichDescription>
  
  <ImageGallery images={content.images} />
  {/* High-res images from Contentful CDN */}
  
  <CareInstructions>{content.careInstructions}</CareInstructions>
  {/* "Lavar a máquina en frío..." */}
  
  <SEOMeta 
    title={content.seoTitle} 
    description={content.seoDescription}
  />
  
  {/* From Medusa (commerce functionality) */}
  <VariantSelector variants={product.variants} />
  <AddToCartButton productId={product.id} />
</ProductPage>
```

**26. Customer Switches Language**
```
User clicks language selector: ES → EN

Next.js router: router.push(`?lang=en-US`)
Page re-fetches with locale='en-US'
Content automatically switches to English
Commerce data stays the same
```

### Timeline

| Step | System | Time | Cumulative |
|------|--------|------|------------|
| 1-6 | Medusa Admin | 2-3 minutes | 3min |
| 7-9 | Medusa Backend | 50-100ms | 3min |
| 10-16 | Sync to Contentful | 200-500ms | 3min |
| 17-21 | Contentful (add localized content) | 20-30 minutes | 33min |
| 22-26 | Storefront | 200-400ms (per page load) | 33min |

**Result:** Product created in Medusa, automatically synced to Contentful, enriched with multi-language content by content team, displayed on storefront in user's language. Total time: ~35 minutes (mostly content creation).

---

## Scenario 2: Updating Product Content in Contentful

**User Story:** Content manager updates Spanish product description in Contentful. Changes sync back to Medusa via webhook.

### Step-by-Step Flow

**1. Open Product in Contentful**
```
Contentful UI → Content → Products
Search for "Organic Cotton T-Shirt"
Click to edit
```

**2. Update Spanish Content**
```
Switch locale: en-US → es-ES

Update Title:
  Old: "Camiseta de Algodón Orgánico"
  New: "Camiseta Premium de Algodón Orgánico"

Update Description:
  Add: "Edición limitada de verano 2024"
```

**3. Save Changes**
```
Click "Save"
Entry status: Published → Changed (Draft)
```

**4. Publish Changes**
```
Click "Publish changes"
Entry status: Changed → Published
Version: 1 → 2
```

**5. Contentful Fires Webhook**
```json
POST https://your-backend.com/hooks/contentful
Content-Type: application/json
X-Contentful-Topic: ContentManagement.Entry.publish

{
  "sys": {
    "type": "Entry",
    "id": "7qVBlCjgXqoqy4wMkKMEwM",
    "contentType": {
      "sys": { "id": "product" }
    }
  },
  "fields": {
    "medusaId": {
      "en-US": "prod_01HQ5K9ZFWXYZ123ABC"
    },
    "title": {
      "en-US": "Organic Cotton T-Shirt",
      "es-ES": "Camiseta Premium de Algodón Orgánico"  // Updated
    },
    "description": {
      "es-ES": "...Edición limitada de verano 2024"  // Updated
    }
  }
}
```

**6. Medusa Webhook Handler Receives**
```typescript
// POST /hooks/contentful
app.post("/hooks/contentful", async (req, res) => {
  const payload = req.body;
  
  // Extract Medusa ID
  const medusaId = payload.fields.medusaId["en-US"];
  
  // Check content type
  if (payload.sys.contentType.sys.id !== "product") {
    return res.status(200).json({ message: "Ignored" });
  }
  
  // Process update
  await webhookHandler.handleProductUpdate(medusaId, payload.fields);
  
  res.status(200).json({ success: true });
});
```

**7. Update Medusa Product (Selective)**
```typescript
// Only sync base fields (not localized content)
const updateData = {
  title: payload.fields.title["en-US"],  // Only default locale
  description: payload.fields.description["en-US"]
};

// Set ignore flag to prevent sync loop
await redis.set(`product:${medusaId}:ignore_contentful`, "1", "EX", 5);

// Update in Medusa
await productService.update(medusaId, updateData);
```

**8. Sync Loop Prevention**
```typescript
// This update triggers product.updated event in Medusa
// But subscriber checks ignore flag:

this.eventBus_.subscribe(
  ProductService.Events.UPDATED,
  async (data) => {
    const ignore = await redis.get(`product:${data.id}:ignore_contentful`);
    if (ignore) {
      return;  // Skip sync back to Contentful
    }
    
    await this.contentfulService_.updateProduct(data.id);
  }
);
```

**9. Localized Content Stays in Contentful**
```
Spanish title: "Camiseta Premium de Algodón Orgánico" ✓
  → Stays in Contentful only
  → Not synced to Medusa
  → Available via Contentful Delivery API

English title: "Organic Cotton T-Shirt"
  → Optionally synced to Medusa (default locale)
  → Medusa stores basic title
  → Contentful stores all locales
```

**10. Storefront Shows Updated Content**
```javascript
// Next Spanish customer visit
const content = await contentful.getEntries({
  content_type: 'product',
  'fields.medusaId': 'prod_01HQ5K9ZFWXYZ123ABC',
  locale: 'es-ES'
});

console.log(content.items[0].fields.title);
// Output: "Camiseta Premium de Algodón Orgánico" ✓ Updated
```

### Key Takeaway

✅ **Localized content updated in Contentful appears immediately on storefront**  
✅ **Base fields (default locale) optionally sync back to Medusa**  
✅ **Sync loop prevented with ignore flags**  
✅ **Multi-language content managed independently in Contentful**

---

## Scenario 3: Adding New Locale

**User Story:** Business expands to France. Marketing team adds French translations to all products.

### Step-by-Step Flow

**1. Configure New Locale in Contentful**
```
Contentful → Settings → Locales → Add locale
Locale: French (France)
Code: fr-FR
Fallback: en-US (if French not available)
```

**2. Content Type Fields Auto-Support New Locale**
```
All localized fields now accept fr-FR content:
  - title (localized)
  - description (localized)
  - richDescription (localized)
  - seoTitle (localized)
  - seoDescription (localized)
  - careInstructions (localized)

Non-localized fields stay single value:
  - medusaId (not localized)
  - weight (not localized)
  - sku (not localized)
```

**3. Bulk Translation (Option A: Manual)**
```
For each product:
  1. Open in Contentful
  2. Switch to fr-FR locale
  3. Translate title, description, etc.
  4. Save and publish
```

**4. Bulk Translation (Option B: Contentful API)**
```typescript
// Script to bulk add French translations
const products = await contentful.getEntries({
  content_type: 'product',
  locale: 'en-US'
});

for (const product of products.items) {
  const frenchTitle = await translate(product.fields.title, 'fr-FR');
  const frenchDescription = await translate(product.fields.description, 'fr-FR');
  
  await contentfulManagement.updateEntry(
    product.sys.id,
    {
      fields: {
        title: {
          'fr-FR': frenchTitle
        },
        description: {
          'fr-FR': frenchDescription
        }
      }
    }
  );
  
  await contentfulManagement.publishEntry(product.sys.id);
}
```

**5. Update Storefront to Support French**
```typescript
// Next.js i18n config
module.exports = {
  i18n: {
    locales: ['en-US', 'es-ES', 'de-DE', 'fr-FR'],  // Add fr-FR
    defaultLocale: 'en-US',
  },
};

// Fetch content in user's locale
const locale = router.locale;  // 'fr-FR'
const content = await contentful.getEntries({
  content_type: 'product',
  'fields.medusaId': productId,
  locale: locale  // Fetch French content
});
```

**6. French Customer Views Site**
```
Browser: Accept-Language: fr-FR
Storefront detects: French user

Fetches commerce from Medusa: Prices, inventory (same for all)
Fetches content from Contentful (fr-FR):
  - Title: "T-Shirt en Coton Biologique"
  - Description: "Un t-shirt confortable en coton 100% biologique..."
  - Rich Description: (French marketing content)
  - Care Instructions: "Laver en machine à froid..."
  - SEO: French meta tags

Renders French product page ✓
```

### Key Takeaway

✅ **New locales added in Contentful settings**  
✅ **No changes needed in Medusa (commerce is language-agnostic)**  
✅ **Content team translates independently**  
✅ **Storefront automatically fetches correct locale**

---

## Scenario 4: Product Deletion

**User Story:** Product discontinued, manager deletes from Medusa. Deletion propagates to Contentful.

### Step-by-Step Flow

**1. Delete in Medusa Admin**
```
Products → "Organic Cotton T-Shirt"
Click "..." menu → "Delete Product"
Confirm deletion
```

**2. Medusa Processes**
```javascript
await productService.delete("prod_01HQ5K9ZFWXYZ123ABC");
// Soft delete: sets deleted_at timestamp in Medusa
```

**3. Event Emitted**
```
Event: "product.deleted"
Data: { id: "prod_01HQ5K9ZFWXYZ123ABC" }
```

**4. Subscriber Catches**
```typescript
this.eventBus_.subscribe(
  ProductService.Events.DELETED,
  async (data) => {
    await this.contentfulService_.deleteProductInContentful(
      data.id
    );
  }
);
```

**5. Find Entry in Contentful**
```typescript
// Search by medusaId
const entries = await contentful.getEntries({
  content_type: 'product',
  'fields.medusaId': 'prod_01HQ5K9ZFWXYZ123ABC'
});

const entryId = entries.items[0]?.sys.id;
// Result: "7qVBlCjgXqoqy4wMkKMEwM"
```

**6. Unpublish Entry**
```typescript
// First unpublish (make unavailable)
await contentfulManagement.unpublishEntry(entryId);
```

**7. Delete Entry**
```typescript
// Then delete (permanent)
await contentfulManagement.deleteEntry(entryId);
```

**8. Contentful Entry Removed**
```
Entry deleted from Contentful
All localized content lost (en, es, de, fr)
All media references removed
```

**9. Storefront**
```javascript
// Product no longer returned
const product = await medusa.products.retrieve(id);
// Error: Product not found

// Contentful also returns empty
const content = await contentful.getEntries({
  content_type: 'product',
  'fields.medusaId': id
});
// Result: { items: [] }
```

### Key Takeaway

⚠️ **Deletion in Medusa removes product from both systems**  
⚠️ **All Contentful content (all locales, rich text, images) is lost**  
⚠️ **Consider archiving instead of deleting for important content**

**Alternative: Archive Instead of Delete**
```typescript
// In Contentful, archive instead of delete
await contentfulManagement.archiveEntry(entryId);
// Entry hidden but not deleted, can be restored
```

---

## Scenario 5: Bulk Import with Localization

**User Story:** Importing 500 products from CSV. Need to sync to Contentful and add basic translations.

### Step-by-Step Flow

**1. Import to Medusa First**
```bash
# Run import script
node scripts/import-products.js products.csv

# Creates 500 products in Medusa
# Each emits product.created event
# All automatically sync to Contentful
```

**2. Monitor Sync Progress**
```bash
# Medusa logs show sync status
info: Syncing product 1/500 to Contentful
info: Syncing product 50/500 to Contentful
...
info: Successfully synced 500 products to Contentful
```

**3. Products Created in Contentful**
```
500 product entries in Contentful
All in default locale (en-US) only
Status: Draft (unpublished)
```

**4. Bulk Add Translations via API**
```typescript
// Script to add Spanish for all products
const products = await contentful.getEntries({
  content_type: 'product',
  limit: 1000
});

for (const product of products.items) {
  const esTitle = await translateAPI(product.fields.title, 'es');
  const esDesc = await translateAPI(product.fields.description, 'es');
  
  await contentfulManagement.updateEntry(
    product.sys.id,
    {
      fields: {
        title: { 'es-ES': esTitle },
        description: { 'es-ES': esDesc }
      }
    }
  );
}
```

**5. Bulk Publish**
```typescript
// Publish all products
for (const product of products.items) {
  await contentfulManagement.publishEntry(product.sys.id);
}
```

**6. Storefront Now Shows Localized Content**
```
All 500 products available
English content: ✓ (from Medusa sync)
Spanish content: ✓ (from bulk translation)
```

### Performance

| Operation | Count | Time | Rate |
|-----------|-------|------|------|
| Import to Medusa | 500 | ~5min | 100/min |
| Sync to Contentful | 500 | ~10min | 50/min |
| Bulk translate | 500 | ~15min | 33/min |
| Bulk publish | 500 | ~5min | 100/min |
| **Total** | **500** | **~35min** | **14/min** |

### Key Takeaway

✅ **Bulk import works automatically (Medusa → Contentful)**  
✅ **Use Contentful API for bulk translations**  
✅ **Monitor sync progress in logs**  
💡 **Consider rate limits (Free tier: 10 req/s)**

---

## Common Edge Cases

### Edge Case 1: Network Failure During Sync

**Scenario:** Contentful API unavailable mid-sync.

**What Happens:**
```typescript
try {
  await axios.post(CONTENTFUL_API, data);
} catch (error) {
  if (error.code === 'ECONNREFUSED' || error.response?.status >= 500) {
    // Log error
    logger.error(`Failed to sync product ${id}:`, error.message);
    
    // Retry logic (exponential backoff)
    await sleep(5000);
    return await syncToContentful(data);  // Retry
  }
}
```

**Result:** Sync completes once Contentful is back online. Event bus ensures no data loss.

### Edge Case 2: Duplicate medusaId

**Scenario:** Product already exists in Contentful, create event fires again.

**What Happens:**
```typescript
// Check before creating
const existing = await contentful.getEntries({
  content_type: 'product',
  'fields.medusaId': id
});

if (existing.items.length > 0) {
  // Update instead of create
  return await updateEntry(existing.items[0].sys.id, data);
}

// Create new
return await createEntry(data);
```

**Result:** Update performed, no duplicate created.

### Edge Case 3: Missing Locale

**Scenario:** Storefront requests German content, but German translation doesn't exist.

**What Happens:**
```typescript
// Fetch with locale
const content = await contentful.getEntries({
  content_type: 'product',
  'fields.medusaId': productId,
  locale: 'de-DE'  // German
});

// If German not available, Contentful returns fallback locale (en-US)
const title = content.items[0].fields.title;
// Returns English title if German not available
```

**Result:** Fallback to default locale (configured in Contentful settings).

### Edge Case 4: Webhook Fails

**Scenario:** Medusa backend down, webhook from Contentful fails.

**What Happens:**
```
Contentful sends webhook → Medusa returns 503 (Service Unavailable)
Contentful marks webhook as failed
Can be replayed manually in Contentful dashboard
```

**Solution:**
```
1. Go to Contentful → Settings → Webhooks
2. Click on webhook → View activity log
3. Find failed delivery
4. Click "Retry"
```

**Result:** Webhook replayed once Medusa is online.

### Edge Case 5: API Rate Limit Hit

**Scenario:** Bulk sync hits Contentful API rate limit (10 req/s on free tier).

**What Happens:**
```typescript
// Contentful returns 429 Too Many Requests
{
  "sys": { "type": "Error", "id": "RateLimitExceeded" },
  "message": "Rate limit exceeded",
  "requestId": "...",
  "headers": {
    "X-Contentful-RateLimit-Reset": "1234567890"  // Unix timestamp
  }
}
```

**Plugin Handles:**
```typescript
if (error.response?.status === 429) {
  const resetTime = error.response.headers['x-contentful-ratelimit-reset'];
  const waitTime = resetTime - Date.now() / 1000 + 1;  // Wait until reset + 1s
  
  await sleep(waitTime * 1000);
  return await syncToContentful(data);  // Retry after rate limit resets
}
```

**Result:** Sync pauses until rate limit resets, then continues.

---

## What's Next?

- **[Implementation Guide](./04-implementation.md)** - Set up the integration
- **[Pros & Cons](./05-pros-cons.md)** - Evaluate trade-offs
- **[FAQ](./FAQ.md)** - Common questions
- **[Architecture](./02-architecture.md)** - Technical details

