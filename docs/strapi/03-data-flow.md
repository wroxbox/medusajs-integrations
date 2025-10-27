# Data Flow: Step-by-Step Scenarios

This document walks through common integration scenarios with detailed step-by-step examples.

## Scenario 1: Creating a New Product

**User Story:** A product manager creates a new product in Medusa Admin. The product structure automatically syncs to Strapi, and the content team enriches it.

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

#### Phase 4: Strapi Sync (Automatic)

**10. Subscriber Catches Event**
```typescript
// In medusa-plugin-strapi-ts/src/subscribers/strapi.ts
this.eventBus_.subscribe(
  ProductService.Events.CREATED,
  async (data: Product) => {
    await this.strapiService_.createProductInStrapi(
      data.id,
      authInterface
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

**12. Transform to Strapi Format**
```typescript
// Rename fields for Strapi schema
const productToSend = {
  medusa_id: "prod_01HQ5K9ZFWXYZ123ABC",
  title: "Organic Cotton T-Shirt",
  subtitle: "Soft, sustainable, and stylish",
  description: "A comfortable t-shirt...",
  handle: "organic-cotton-tshirt",
  thumbnail: "https://cdn.example.com/products/oct-thumb.jpg",
  weight: 200,
  
  // Renamed for Strapi relations
  product_collection: {
    medusa_id: "pcol_summer_essentials"
  },
  product_categories: [
    { medusa_id: "pcat_apparel" },
    { medusa_id: "pcat_tshirts" },
    { medusa_id: "pcat_organic" }
  ],
  product_tags: [
    { value: "sustainable" },
    { value: "organic" },
    { value: "cotton" }
  ],
  product_variants: [
    {
      medusa_id: "variant_01HQ5K9...1",
      title: "Small / White",
      sku: "OCT-S-WHT",
      money_amount: [
        {
          amount: 2999,
          currency_code: "usd"
        }
      ],
      product_option_value: [
        { value: "S", option: { value: "Size" } },
        { value: "White", option: { value: "Color" } }
      ]
    },
    // ... other variants
  ]
};
```

**13. Authenticate with Strapi**
```typescript
// Check cached token
let token = userTokens["medusa@service.local"]?.token;

if (!token || isTokenExpired(token)) {
  // Login
  const res = await axios.post(
    "http://localhost:1337/api/auth/local",
    {
      identifier: "medusa@service.local",
      password: process.env.STRAPI_MEDUSA_PASSWORD
    }
  );
  
  token = res.data.jwt;
  // Cache for future use
  userTokens["medusa@service.local"] = {
    token,
    time: Date.now(),
    user: res.data.user
  };
}
```

**14. POST to Strapi API**
```typescript
const response = await axios.post(
  "http://localhost:1337/api/products",
  {
    data: productToSend
  },
  {
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json"
    }
  }
);

// Response:
// {
//   data: {
//     id: 42,  // Strapi internal ID
//     attributes: {
//       medusa_id: "prod_01HQ5K9ZFWXYZ123ABC",
//       title: "Organic Cotton T-Shirt",
//       // ... all synced fields
//     }
//   },
//   meta: {}
// }
```

**15. Strapi Saves to Database**
```sql
-- Strapi PostgreSQL
INSERT INTO products (
  medusa_id,
  title,
  subtitle,
  description,
  handle,
  thumbnail,
  weight,
  created_at,
  updated_at
) VALUES (
  'prod_01HQ5K9ZFWXYZ123ABC',
  'Organic Cotton T-Shirt',
  'Soft, sustainable, and stylish',
  'A comfortable t-shirt...',
  'organic-cotton-tshirt',
  'https://cdn.example.com/products/oct-thumb.jpg',
  200,
  NOW(),
  NOW()
);

-- Also creates related variants, links to collection, categories, tags
```

#### Phase 5: Content Enrichment (Content Team)

**16. Content Manager Opens Strapi Admin**
```
Browser → http://localhost:1337/admin
Login as content editor
Navigate to Content Manager → Products
See newly synced "Organic Cotton T-Shirt"
```

**17. Enrich Product Content**
```
Click "Organic Cotton T-Shirt" to edit

Add Rich Description:
┌─────────────────────────────────────────────────┐
│ Rich Text Editor                                │
├─────────────────────────────────────────────────┤
│                                                 │
│ ## Why You'll Love It                           │
│                                                 │
│ Our **Organic Cotton T-Shirt** is more than    │
│ just clothing—it's a statement about the kind   │
│ of world you want to live in.                   │
│                                                 │
│ ### Features:                                   │
│ - 100% GOTS-certified organic cotton            │
│ - Pre-shrunk for perfect fit                    │
│ - Reinforced seams for durability               │
│ - Carbon-neutral shipping                       │
│                                                 │
│ [Add image: close-up of fabric texture]         │
│ [Add image: model wearing shirt in nature]      │
└─────────────────────────────────────────────────┘

Upload High-Res Images:
  1. Main product photo (4000x4000px)
  2. Lifestyle shot (3000x2000px)
  3. Detail shots (6 images)
  4. Size chart graphic

Add Custom Fields:
  - Care Instructions: "Machine wash cold, tumble dry low"
  - Sustainability Score: 9.5/10
  - Made In: "Portugal"
  - Certifications: ["GOTS", "Fair Trade", "Carbon Neutral"]

Set SEO:
  - Meta Title: "Organic Cotton T-Shirt | Sustainable Fashion"
  - Meta Description: "Premium organic cotton t-shirt. Sustainably made..."
  - Focus Keyword: "organic cotton t-shirt"
```

**18. Save & Publish**
```
Click "Save" button
Content saved to Strapi PostgreSQL
```

**Important:** Rich content fields do NOT sync back to Medusa. They live only in Strapi.

#### Phase 6: Storefront Display (Customer)

**19. Customer Visits Product Page**
```
Browser → https://store.example.com/products/organic-cotton-tshirt
```

**20. Storefront Fetches Data**

**Option A: Dual Fetch**
```javascript
// Fetch commerce data from Medusa
const product = await medusaClient.products.retrieve(
  "prod_01HQ5K9ZFWXYZ123ABC"
);
// Returns: prices, inventory, variants, cart functionality

// Fetch content from Strapi
const strapiRes = await fetch(
  `${STRAPI_URL}/api/products?filters[medusa_id][$eq]=prod_01HQ5K9ZFWXYZ123ABC&populate=*`
);
const content = await strapiRes.json();
// Returns: rich description, images, SEO fields
```

**Option B: Via Medusa Proxy**
```javascript
// Single call to Medusa
const response = await fetch(
  `${MEDUSA_URL}/strapi/content/products/prod_01HQ5K9ZFWXYZ123ABC`
);
const productWithContent = await response.json();
// Medusa fetches from Strapi and returns merged data
```

**21. Render Product Page**
```jsx
<ProductPage>
  {/* From Medusa */}
  <ProductTitle>{product.title}</ProductTitle>
  <Price>${product.variants[0].price / 100}</Price>
  <Stock>{product.variants[0].inventory_quantity} in stock</Stock>
  
  {/* From Strapi */}
  <RichDescription html={content.rich_description} />
  <ImageGallery images={content.images} />
  <CareInstructions>{content.care_instructions}</CareInstructions>
  <SustainabilityBadge score={content.sustainability_score} />
  
  {/* From Medusa */}
  <VariantSelector variants={product.variants} />
  <AddToCartButton productId={product.id} />
</ProductPage>
```

### Timeline

| Step | System | Time | Cumulative |
|------|--------|------|------------|
| 1-6 | Medusa Admin | 2-3 minutes | 3min |
| 7-9 | Medusa Backend | 50-100ms | 3min |
| 10-15 | Sync to Strapi | 200-500ms | 3min |
| 16-18 | Strapi Admin | 10-15 minutes | 18min |
| 19-21 | Storefront | 200-400ms | 18min |

**Result:** Product created in Medusa, automatically synced to Strapi, enriched by content team, displayed on storefront. Total time: ~20 minutes (mostly content creation).

---

## Scenario 2: Updating a Product

**User Story:** Product manager updates product price in Medusa. Changes sync to Strapi, but rich content remains untouched.

### Step-by-Step Flow

**1. Update Price in Medusa Admin**
```
Navigate to: Products → "Organic Cotton T-Shirt"
Edit variant "Medium / White"
Change price: $29.99 → $34.99
Click "Save"
```

**2. Medusa Processes Update**
```javascript
await productVariantService.update("variant_01HQ5K9...2", {
  prices: [
    {
      amount: 3499,  // Was 2999
      currency_code: "usd"
    }
  ]
});
```

**3. Event Emitted**
```
Event: "product_variant.updated"
Data: { id: "variant_01HQ5K9...2", fields: ["prices"] }
```

**4. Subscriber Checks Fields**
```typescript
// In subscriber
const updateFields = ['title', 'prices', 'sku', 'material'];
const found = data.fields?.find(f => updateFields.includes(f));

if (!found) {
  return;  // Skip sync for irrelevant fields
}

// 'prices' is in updateFields, so continue
```

**5. Check Ignore Flag**
```typescript
const ignore = await redis.get("variant_01HQ5K9...2_ignore_strapi");
if (ignore) {
  return;  // Prevent sync loop
}
```

**6. Fetch & Transform**
```typescript
const variant = await productVariantService.retrieve(
  "variant_01HQ5K9...2",
  { relations: ['prices', 'options'] }
);

const variantToSend = {
  medusa_id: "variant_01HQ5K9...2",
  title: "Medium / White",
  sku: "OCT-M-WHT",
  money_amount: [
    {
      medusa_id: "ma_01HQ...",
      amount: 3499,  // Updated price
      currency_code: "usd"
    }
  ]
};
```

**7. PUT to Strapi**
```typescript
await axios.put(
  `http://localhost:1337/api/product-variants?filters[medusa_id][$eq]=variant_01HQ5K9...2`,
  { data: variantToSend },
  { headers: { Authorization: `Bearer ${token}` } }
);
```

**8. Strapi Updates**
```sql
-- Strapi PostgreSQL
UPDATE product_variants
SET title = 'Medium / White',
    sku = 'OCT-M-WHT',
    updated_at = NOW()
WHERE medusa_id = 'variant_01HQ5K9...2';

-- Update related money_amount
UPDATE money_amounts
SET amount = 3499
WHERE medusa_id = 'ma_01HQ...';
```

**9. Rich Content Preserved**
```
Strapi product still has:
  - Rich description ✓ (unchanged)
  - High-res images ✓ (unchanged)
  - Care instructions ✓ (unchanged)
  - Sustainability score ✓ (unchanged)
```

**10. Storefront Shows Updated Price**
```javascript
// Next fetch shows new price
const product = await medusaClient.products.retrieve(id);
product.variants[1].price = 3499;  // Updated
```

### Key Takeaway

✅ **Commerce fields (prices, SKUs, inventory) sync from Medusa to Strapi**  
✅ **Content fields (descriptions, images, custom data) stay in Strapi**  
✅ **No data loss during updates**

---

## Scenario 3: Content-Only Update (Strapi)

**User Story:** Content manager adds product video and updates care instructions in Strapi. Changes stay in Strapi, don't sync to Medusa.

### Step-by-Step Flow

**1. Edit in Strapi Admin**
```
Content Manager → Products → "Organic Cotton T-Shirt"

Add Product Video:
  - Upload: product-video.mp4 (to Strapi media library)
  - Add to "Videos" field

Update Care Instructions:
  Old: "Machine wash cold, tumble dry low"
  New: "Machine wash cold, tumble dry low. Do not bleach. Iron on low heat."

Add New Custom Field:
  - Fabric Weight: "180 GSM"
```

**2. Save in Strapi**
```
Click "Save"
Strapi updates PostgreSQL
```

**3. No Event to Medusa**
```
❌ No automatic sync back to Medusa
(Limited reverse sync for title/subtitle only)
```

**4. Storefront Fetches Fresh Content**
```javascript
// Next time storefront loads
const content = await fetch(
  `${STRAPI_URL}/api/products?filters[medusa_id][$eq]=prod_01HQ5K9ZFWXYZ123ABC&populate=*`
).then(res => res.json());

// New content available:
content.data[0].attributes.videos[0].url;  // New video
content.data[0].attributes.care_instructions;  // Updated
content.data[0].attributes.fabric_weight;  // New field
```

**5. Render on Storefront**
```jsx
<ProductPage>
  {/* Strapi content - updated */}
  <VideoPlayer src={content.videos[0].url} />
  <CareInstructions>{content.care_instructions}</CareInstructions>
  <FabricWeight>{content.fabric_weight}</FabricWeight>
  
  {/* Medusa commerce - unchanged */}
  <Price>${product.price / 100}</Price>
  <AddToCart />
</ProductPage>
```

### Key Takeaway

✅ **Content updates in Strapi are immediately available to storefront**  
✅ **No sync to Medusa needed (content lives in Strapi only)**  
✅ **Content team works independently**

---

## Scenario 4: Product Deletion

**User Story:** Product discontinued, manager deletes from Medusa. Deletion propagates to Strapi.

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
// Soft delete: sets deleted_at timestamp
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
    await this.strapiService_.deleteProductInStrapi(
      data.id,
      authInterface
    );
  }
);
```

**5. Check Ignore Flag**
```typescript
const ignore = await shouldIgnore_("prod_01HQ5K9ZFWXYZ123ABC", "strapi");
if (ignore) return;
```

**6. DELETE from Strapi**
```typescript
await axios.delete(
  `http://localhost:1337/api/products?filters[medusa_id][$eq]=prod_01HQ5K9ZFWXYZ123ABC`,
  { headers: { Authorization: `Bearer ${token}` } }
);
```

**7. Strapi Deletes**
```sql
-- Strapi PostgreSQL (hard delete by default)
DELETE FROM products
WHERE medusa_id = 'prod_01HQ5K9ZFWXYZ123ABC';

-- Cascade deletes related variants (if configured)
DELETE FROM product_variants
WHERE product_id = (
  SELECT id FROM products WHERE medusa_id = 'prod_01HQ5K9ZFWXYZ123ABC'
);
```

**8. Storefront**
```javascript
// Product no longer returned in queries
const product = await medusaClient.products.retrieve(id);
// Error: Product not found

// Strapi also returns 404
const content = await fetch(`${STRAPI_URL}/api/products?filters[medusa_id][$eq]=${id}`);
// { data: [] }
```

### Key Takeaway

⚠️ **Deletion in Medusa removes product from both systems**  
⚠️ **All Strapi content (rich descriptions, images) is lost**  
⚠️ **Consider soft deletes and archiving for important content**

---

## Scenario 5: Adding Product to Collection

**User Story:** Product manager adds "Organic Cotton T-Shirt" to "Best Sellers" collection.

### Step-by-Step Flow

**1. Update Collection in Medusa**
```
Collections → "Best Sellers"
Click "Add Products"
Select "Organic Cotton T-Shirt"
Save
```

**2. Medusa Processes**
```javascript
await productCollectionService.addProducts(
  "pcol_best_sellers",
  ["prod_01HQ5K9ZFWXYZ123ABC"]
);
```

**3. Event Emitted**
```
Event: "product_collection.products_added"
Data: {
  id: "pcol_best_sellers",
  productIds: ["prod_01HQ5K9ZFWXYZ123ABC"]
}
```

**4. Subscriber Updates**
```typescript
this.eventBus_.subscribe(
  ProductCollectionService.Events.PRODUCTS_ADDED,
  async (data) => {
    await this.strapiService_.updateProductsWithinCollectionInStrapi(
      data.id,
      authInterface
    );
  }
);
```

**5. Fetch & Update Each Product**
```typescript
for (const productId of data.productIds) {
  const product = await productService.retrieve(productId, {
    relations: ['collection']
  });
  
  await axios.put(
    `http://localhost:1337/api/products?filters[medusa_id][$eq]=${productId}`,
    {
      data: {
        product_collection: {
          medusa_id: "pcol_best_sellers"
        }
      }
    }
  );
}
```

**6. Strapi Updates Relations**
```sql
-- Update product's collection reference
UPDATE products
SET product_collection_id = (
  SELECT id FROM product_collections WHERE medusa_id = 'pcol_best_sellers'
)
WHERE medusa_id = 'prod_01HQ5K9ZFWXYZ123ABC';
```

**7. Storefront Queries**
```javascript
// Query by collection
const products = await medusaClient.products.list({
  collection_id: ["pcol_best_sellers"]
});

// "Organic Cotton T-Shirt" now included
```

### Key Takeaway

✅ **Collection membership syncs automatically**  
✅ **Strapi maintains same product-collection relationships as Medusa**

---

## Scenario 6: Bulk Sync Operation

**User Story:** Setting up integration for existing store with 500 products. Need to sync all at once.

### Step-by-Step Flow

**1. Initial Setup Complete**
```
- Medusa backend running with plugin
- Strapi server running with plugin
- Service account created
- JWT authentication working
```

**2. Configure for Sync**
```javascript
// medusa-config.js
{
  resolve: "medusa-plugin-strapi-ts",
  options: {
    ...
    sync_on_init: true,  // Enable initial sync
    auto_start: true     // Start automatically
  }
}
```

**3. Restart Medusa**
```bash
npx medusa develop
```

**4. Plugin Initializes**
```typescript
// On startup, plugin calls:
await updateStrapiService.startInterface();

// Which triggers:
await updateStrapiService.intializeServer();

// Which calls:
await executeSync(strapiSuperAdminToken);
```

**5. Sync Endpoint Called**
```typescript
const result = await axios.post(
  `${STRAPI_URL}/strapi-plugin-medusajs/synchronise-medusa-tables`,
  {},
  {
    headers: { Authorization: `Bearer ${superAdminToken}` },
    timeout: 3600000  // 1 hour timeout
  }
);
```

**6. Strapi Plugin Processes**
```typescript
// In strapi-plugin-medusajs/controllers/setup.ts
async synchroniseMedusaTables() {
  // Fetch all products from Medusa
  const products = await this.fetchAllMedusaProducts();
  
  // Create/update in Strapi
  for (const product of products) {
    await this.createOrUpdateProduct(product);
  }
  
  // Fetch all collections
  const collections = await this.fetchAllMedusaCollections();
  for (const collection of collections) {
    await this.createOrUpdateCollection(collection);
  }
  
  // Continue for all entity types...
}
```

**7. Progress Monitoring**
```
# In Medusa logs
info: Syncing products... (0/500)
info: Syncing products... (50/500)
info: Syncing products... (100/500)
...
info: Syncing products... (500/500)
info: Successfully synced 500 products
info: Syncing collections... (0/20)
...
info: Sync complete ✓
```

**8. Verification**
```bash
# Check Strapi database
psql -d strapi_db -c "SELECT COUNT(*) FROM products;"
# Result: 500

# Check via API
curl http://localhost:1337/api/products?pagination[pageSize]=1
# Returns: { meta: { pagination: { total: 500 } } }
```

### Performance

| Entity Type | Count | Time | Rate |
|-------------|-------|------|------|
| Products | 500 | ~10min | 50/min |
| Variants | 2000 | ~20min | 100/min |
| Collections | 20 | ~1min | 20/min |
| Categories | 50 | ~2min | 25/min |
| Regions | 5 | ~10sec | 30/min |
| **Total** | **2575** | **~35min** | **74/min** |

**Note:** Performance depends on network latency, database speed, and API rate limits.

### Key Takeaway

✅ **Bulk sync available for initial setup**  
⚠️ **Can take 30+ minutes for large catalogs**  
💡 **Consider running during low-traffic periods**

---

## Common Edge Cases

### Edge Case 1: Network Failure During Sync

**Scenario:** Strapi server goes down mid-sync.

**What Happens:**
```typescript
// Plugin retries with exponential backoff
try {
  await axios.post(STRAPI_URL, data);
} catch (error) {
  if (error.code === 'ECONNREFUSED') {
    // Wait and retry
    await sleep(5000);
    return await syncToStrapi(data);  // Recursive retry
  }
}
```

**Result:** Sync completes once Strapi is back online. Event bus ensures no data loss.

### Edge Case 2: Duplicate medusa_id

**Scenario:** Product already exists in Strapi, create event fires again.

**What Happens:**
```typescript
// Plugin checks before creating
const existing = await axios.get(
  `${STRAPI_URL}/api/products?filters[medusa_id][$eq]=${id}`
);

if (existing.data.data.length > 0) {
  // Update instead of create
  return await updateProduct(id, data);
}
```

**Result:** Update performed, no duplicate created.

### Edge Case 3: Sync Loop

**Scenario:** Update in Strapi triggers update in Medusa, which triggers update in Strapi...

**What Happens:**
```typescript
// Redis ignore flags prevent loops
await addIgnore_(productId, 'strapi', 3);  // Set flag

// Later...
const ignore = await shouldIgnore_(productId, 'strapi');
if (ignore) {
  return;  // Skip sync
}
```

**Result:** Loop prevented, flag expires after 3 seconds.

### Edge Case 4: Token Expiration Mid-Operation

**Scenario:** JWT token expires during bulk sync.

**What Happens:**
```typescript
try {
  await axios.post(STRAPI_URL, data, {
    headers: { Authorization: `Bearer ${token}` }
  });
} catch (error) {
  if (error.response.status === 401) {
    // Re-authenticate
    const newToken = await login();
    // Retry with new token
    return await syncToStrapi(data, newToken);
  }
}
```

**Result:** Automatic re-authentication, operation continues.

---

## What's Next?

- **[Implementation Guide](./04-implementation.md)** - Set up the integration
- **[Pros & Cons](./05-pros-cons.md)** - Evaluate trade-offs
- **[FAQ](./FAQ.md)** - Common questions
- **[Architecture](./02-architecture.md)** - Technical details

