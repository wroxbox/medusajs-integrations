# Sanity CMS Integration - Data Flow

This document walks through detailed data flow scenarios showing exactly how data moves through the system.

## Table of Contents

1. [Scenario 1: Creating a New Product](#scenario-1-creating-a-new-product)
2. [Scenario 2: Enriching Content in Sanity](#scenario-2-enriching-content-in-sanity)
3. [Scenario 3: Customer Views Product](#scenario-3-customer-views-product)
4. [Scenario 4: Updating Product Title](#scenario-4-updating-product-title)
5. [Scenario 5: Deleting a Product](#scenario-5-deleting-a-product)
6. [Scenario 6: Bulk Sync Operation](#scenario-6-bulk-sync-operation)
7. [Scenario 7: Manual Sync from Admin](#scenario-7-manual-sync-from-admin)
8. [Scenario 8: Adding Product Addons](#scenario-8-adding-product-addons)

---

## Scenario 1: Creating a New Product

**User Action**: Admin creates a new product in Medusa Admin Dashboard

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Admin Creates Product                                       │
└─────────────────────────────────────────────────────────────────────┘

Admin fills form:
├── Title: "Premium Cotton T-Shirt"
├── Description: "A comfortable t-shirt"
├── Variants: Small, Medium, Large
├── Price: $29.99
└── Clicks "Save"

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Medusa Saves to Database                                    │
└─────────────────────────────────────────────────────────────────────┘

PostgreSQL Transaction:
INSERT INTO product (id, title, description, ...)
VALUES ('prod_01ABC123', 'Premium Cotton T-Shirt', ...);

Generated:
├── Product ID: prod_01ABC123
├── Handle: premium-cotton-t-shirt
├── Created At: 2024-01-15T10:30:00Z
└── Status: draft

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Event Bus Emits Event                                       │
└─────────────────────────────────────────────────────────────────────┘

Event Details:
{
  name: "product.created",
  data: {
    id: "prod_01ABC123"
  },
  metadata: {
    source: "admin",
    timestamp: "2024-01-15T10:30:01Z"
  }
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Subscriber Catches Event                                    │
└─────────────────────────────────────────────────────────────────────┘

File: src/subscribers/sanity-product-sync.ts

async function upsertSanityProduct({ event, container }) {
  console.log('Product created, triggering sync:', event.data.id);
  
  await sanitySyncProductsWorkflow(container).run({
    input: { product_ids: [event.data.id] }
  });
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Workflow Starts Execution                                   │
└─────────────────────────────────────────────────────────────────────┘

Workflow: sanity-sync-products
Input: { product_ids: ["prod_01ABC123"] }
Transaction ID: wf_tx_xyz789

Workflow Engine:
├── Creates execution record
├── Starts syncStep
└── Enables retry/rollback capabilities

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 6: Sync Step - Query Product                                   │
└─────────────────────────────────────────────────────────────────────┘

const { data: products } = await query.graph({
  entity: "product",
  fields: ["id", "title", "sanity_product.*"],
  filters: { id: ["prod_01ABC123"] },
  pagination: { skip: 0, take: 200 }
});

Result:
{
  id: "prod_01ABC123",
  title: "Premium Cotton T-Shirt",
  sanity_product: null  // Not yet synced
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 7: Sync Step - Transform Data                                  │
└─────────────────────────────────────────────────────────────────────┘

transformProductForCreate(product):

Input (Medusa DTO):
{
  id: "prod_01ABC123",
  title: "Premium Cotton T-Shirt",
  // other fields ignored
}

Output (Sanity Document):
{
  _type: "product",
  _id: "prod_01ABC123",          // Same as Medusa ID
  title: "Premium Cotton T-Shirt",
  specs: [
    {
      _key: "prod_01ABC123",     // Unique key for array
      _type: "spec",
      title: "Premium Cotton T-Shirt",
      lang: "en",                 // Default language
      content: ""                 // Empty - filled in Sanity Studio
    }
  ]
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 8: Sanity Module - Check Existence                             │
└─────────────────────────────────────────────────────────────────────┘

const existing = await this.client.getDocument('prod_01ABC123');

Result: null (document doesn't exist)

Decision: CREATE new document

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 9: HTTP Request to Sanity API                                  │
└─────────────────────────────────────────────────────────────────────┘

Request:
POST https://abc123.api.sanity.io/v2024-01-01/data/mutate/production

Headers:
{
  "Authorization": "Bearer sk-abc123...",
  "Content-Type": "application/json"
}

Body:
{
  "mutations": [
    {
      "create": {
        "_type": "product",
        "_id": "prod_01ABC123",
        "title": "Premium Cotton T-Shirt",
        "specs": [...]
      }
    }
  ]
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 10: Sanity Processes Request                                   │
└─────────────────────────────────────────────────────────────────────┘

Sanity Actions:
├── Validates document schema
├── Generates _createdAt, _updatedAt
├── Generates _rev (revision ID)
├── Stores in Content Lake
├── Propagates to CDN nodes
└── Triggers real-time listeners

Response:
{
  "transactionId": "abc...",
  "results": [{
    "id": "prod_01ABC123",
    "operation": "create",
    "document": {
      "_id": "prod_01ABC123",
      "_type": "product",
      "_createdAt": "2024-01-15T10:30:02Z",
      "_updatedAt": "2024-01-15T10:30:02Z",
      "_rev": "rev123",
      "title": "Premium Cotton T-Shirt",
      "specs": [...]
    }
  }]
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 11: Workflow Completes                                         │
└─────────────────────────────────────────────────────────────────────┘

Sync Step Returns:
{
  total: 1,                        // 1 product synced
  upsertMap: [{
    before: null,                  // Didn't exist
    after: { _id: "prod_01ABC123", ... }  // Created document
  }]
}

Workflow Status: ✓ Success

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 12: Product Now in Both Systems                                │
└─────────────────────────────────────────────────────────────────────┘

Medusa Database:
✓ Product with commerce data

Sanity Content Lake:
✓ Product document with basic structure
✓ Ready for content enrichment

Admin can now:
├── View in Medusa Admin (commerce details)
└── Edit in Sanity Studio (content enrichment)
```

### Timeline

| Time | Event |
|------|-------|
| T+0ms | Admin clicks "Save" |
| T+100ms | Product saved to PostgreSQL |
| T+150ms | Event emitted |
| T+200ms | Subscriber triggers workflow |
| T+250ms | Product queried from Medusa |
| T+300ms | Data transformed |
| T+400ms | HTTP request to Sanity |
| T+600ms | Sanity stores document |
| T+650ms | Workflow completes |

**Total Time**: ~650ms (async, doesn't block admin UI)

---

## Scenario 2: Enriching Content in Sanity

**User Action**: Content manager adds localized specifications in Sanity Studio

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Open Sanity Studio                                          │
└─────────────────────────────────────────────────────────────────────┘

Content Manager:
1. Navigates to http://localhost:8000/studio
2. Logs in with Sanity credentials
3. Sees list of products
4. Clicks "Premium Cotton T-Shirt"

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Studio Loads Document                                       │
└─────────────────────────────────────────────────────────────────────┘

Request to Sanity:
GET https://abc123.api.sanity.io/v2024-01-01/data/query/production
  ?query=*[_id == "prod_01ABC123"][0]

Response:
{
  "_id": "prod_01ABC123",
  "title": "Premium Cotton T-Shirt",
  "specs": [
    {
      "_key": "prod_01ABC123",
      "lang": "en",
      "title": "Premium Cotton T-Shirt",
      "content": ""  // Empty
    }
  ],
  "addons": null
}

Studio renders form with existing data

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Content Manager Edits                                       │
└─────────────────────────────────────────────────────────────────────┘

Actions:
1. Fills English content:
   "Made from 100% organic cotton, this premium t-shirt offers 
    unmatched comfort and durability. Perfect for everyday wear."

2. Adds Spanish spec (clicks "Add item" in specs array):
   {
     lang: "es",
     title: "Camiseta de Algodón Premium",
     content: "Hecha de algodón 100% orgánico..."
   }

3. Adds French spec:
   {
     lang: "fr",
     title: "T-Shirt en Coton Premium",
     content: "Fabriqué à partir de coton 100% bio..."
   }

4. Clicks "Publish"

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Studio Sends Patch to Sanity                                │
└─────────────────────────────────────────────────────────────────────┘

Request:
POST https://abc123.api.sanity.io/v2024-01-01/data/mutate/production

Body:
{
  "mutations": [{
    "patch": {
      "id": "prod_01ABC123",
      "set": {
        "specs": [
          {
            "_key": "prod_01ABC123",
            "lang": "en",
            "title": "Premium Cotton T-Shirt",
            "content": "Made from 100% organic cotton..."
          },
          {
            "_key": "es_001",
            "lang": "es",
            "title": "Camiseta de Algodón Premium",
            "content": "Hecha de algodón 100% orgánico..."
          },
          {
            "_key": "fr_001",
            "lang": "fr",
            "title": "T-Shirt en Coton Premium",
            "content": "Fabriqué à partir de coton 100% bio..."
          }
        ]
      }
    }
  }]
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Sanity Updates Document                                     │
└─────────────────────────────────────────────────────────────────────┘

Sanity Actions:
├── Validates patch
├── Creates new revision
├── Updates _updatedAt timestamp
├── Stores new version
├── Propagates to CDN
└── Triggers real-time listeners

New Document State:
{
  "_id": "prod_01ABC123",
  "_rev": "rev456",  // New revision
  "_updatedAt": "2024-01-15T11:00:00Z",
  "title": "Premium Cotton T-Shirt",
  "specs": [
    { "lang": "en", "content": "Made from 100%..." },
    { "lang": "es", "content": "Hecha de algodón..." },
    { "lang": "fr", "content": "Fabriqué à partir..." }
  ]
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 6: Content Available Globally                                  │
└─────────────────────────────────────────────────────────────────────┘

✓ Content stored in Sanity
✓ Available via CDN globally
✓ NOT synced back to Medusa (one-way only)
✓ Immediately available for storefront

NOTE: Medusa database unchanged - still has original description
```

### Important Notes

- **No Sync Back**: Changes in Sanity DO NOT sync to Medusa
- **Medusa as Source**: Medusa remains source of truth for commerce
- **Sanity for Content**: Sanity is source of truth for localized content
- **Independent Updates**: Both systems can be updated independently

---

## Scenario 3: Customer Views Product

**User Action**: Customer navigates to product page on storefront

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Customer Clicks Product                                     │
└─────────────────────────────────────────────────────────────────────┘

Customer Action:
Clicks "Premium Cotton T-Shirt" from product listing
URL: /us/products/premium-cotton-t-shirt

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Next.js Processes Request                                   │
└─────────────────────────────────────────────────────────────────────┘

File: app/[countryCode]/(main)/products/[handle]/page.tsx

Server Component Executes:
export default async function ProductPage({ params }) {
  // Extract parameters
  const { countryCode, handle } = params
  
  // Start BOTH fetches in parallel
  const [product, sanity] = await Promise.all([
    fetchFromMedusa(countryCode, handle),
    fetchFromSanity(productId)
  ])
  
  return <ProductTemplate product={product} sanity={sanity} />
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Fetch #1 - Medusa API Request                               │
└─────────────────────────────────────────────────────────────────────┘

Request:
GET http://localhost:9000/store/products?handle=premium-cotton-t-shirt
    &region_id=reg_01...
    &fields=*variants.calculated_price,+variants.inventory_quantity

Headers:
{
  "x-publishable-api-key": "pk_..."
}

Medusa Processes:
├── Finds product by handle
├── Calculates prices for region (US)
├── Fetches inventory levels
├── Loads variants
└── Returns product data

Response:
{
  "products": [{
    "id": "prod_01ABC123",
    "title": "Premium Cotton T-Shirt",
    "handle": "premium-cotton-t-shirt",
    "description": "A comfortable t-shirt",  // Original description
    "variants": [
      {
        "id": "variant_small",
        "title": "Small",
        "calculated_price": {
          "calculated_amount": 2999,  // $29.99
          "currency_code": "usd"
        },
        "inventory_quantity": 45
      },
      // ... more variants
    ],
    "images": [...],
    "collection": {...}
  }],
  "count": 1
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Fetch #2 - Sanity API Request                               │
└─────────────────────────────────────────────────────────────────────┘

Request:
GET https://abc123.api.sanity.io/v2024-01-01/data/query/production
    ?query=*[_id == $id][0]
    &$id="prod_01ABC123"

Sanity Processes:
├── Parses GROQ query
├── Fetches document from Content Lake
├── Returns via CDN (fast!)
└── Caches globally

Response:
{
  "_id": "prod_01ABC123",
  "_type": "product",
  "title": "Premium Cotton T-Shirt",
  "specs": [
    {
      "lang": "en",
      "title": "Premium Cotton T-Shirt",
      "content": "Made from 100% organic cotton, this premium t-shirt 
                  offers unmatched comfort and durability. Perfect for 
                  everyday wear."
    },
    {
      "lang": "es",
      "title": "Camiseta de Algodón Premium",
      "content": "Hecha de algodón 100% orgánico..."
    },
    {
      "lang": "fr",
      "title": "T-Shirt en Coton Premium",
      "content": "Fabriqué à partir de coton 100% bio..."
    }
  ],
  "addons": {
    "title": "Complete Your Look",
    "products": [
      { "_ref": "prod_jeans_001" },
      { "_ref": "prod_shoes_002" }
    ]
  }
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Select Language-Specific Content                            │
└─────────────────────────────────────────────────────────────────────┘

// In page component
const userLang = getUserLanguage() // "en" from browser or URL

const spec = sanity.specs?.find(s => s.lang === userLang)

Selected Spec:
{
  "lang": "en",
  "content": "Made from 100% organic cotton..."
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 6: Render Template with Merged Data                            │
└─────────────────────────────────────────────────────────────────────┘

<ProductTemplate 
  product={medusaProduct}    // Commerce data
  sanity={spec}              // Content data
  region={region}
  countryCode="us"
/>

Template Uses:

FROM MEDUSA:
├── Product ID ────────────► For cart operations
├── Title ─────────────────► Heading
├── Variants ──────────────► Size/color selection
├── Prices ────────────────► Price display
├── Inventory ─────────────► Stock indicators
├── Images ────────────────► Product gallery
└── Add to Cart ───────────► Commerce API calls

FROM SANITY:
├── specs[lang].content ───► Rich description
└── addons.products ───────► Related products widget

MERGED:
Description = sanity.content || product.description
(Sanity content takes priority, fallback to Medusa)

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 7: Page Rendered to Customer                                   │
└─────────────────────────────────────────────────────────────────────┘

Customer Sees:
├── Product Title: "Premium Cotton T-Shirt"
├── Description: "Made from 100% organic cotton..." (from Sanity!)
├── Price: $29.99 (from Medusa)
├── Variants: Small, Medium, Large (from Medusa)
├── Stock: 45 available (from Medusa)
├── Add to Cart button (Medusa API)
└── "Complete Your Look" section (from Sanity)

✓ Rich content from Sanity
✓ Real-time commerce from Medusa
✓ Best of both worlds!
```

### Performance Metrics

| Operation | Time | Notes |
|-----------|------|-------|
| Medusa API call | ~100-200ms | Database query + calculations |
| Sanity API call | ~50-100ms | CDN-cached, fast globally |
| Parallel fetching | ~150ms total | Both run simultaneously |
| Page render | ~50ms | Server-side rendering |
| **Total TTFB** | **~200ms** | Acceptable performance |

### Caching Strategy

```typescript
// Medusa data - frequently changing
cache: 'no-store'  // or revalidate: 60

// Sanity data - stable content
cache: 'force-cache', 
next: { revalidate: 3600 }  // 1 hour

// Best practice: Sanity can cache longer, Medusa refreshes often
```

---

## Scenario 4: Updating Product Title

**User Action**: Admin changes product title in Medusa

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Admin Updates Title                                         │
└─────────────────────────────────────────────────────────────────────┘

Admin Action:
Old Title: "Premium Cotton T-Shirt"
New Title: "Premium Organic Cotton T-Shirt"
Clicks "Save"

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Medusa Updates Database                                     │
└─────────────────────────────────────────────────────────────────────┘

UPDATE product 
SET title = 'Premium Organic Cotton T-Shirt',
    updated_at = NOW()
WHERE id = 'prod_01ABC123';

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Event Emitted                                               │
└─────────────────────────────────────────────────────────────────────┘

Event: "product.updated"
Data: { id: "prod_01ABC123" }

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Subscriber Triggers Workflow                                │
└─────────────────────────────────────────────────────────────────────┘

Same as creation, but now:
- Document EXISTS in Sanity
- Will UPDATE instead of CREATE

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Transform for Update                                        │
└─────────────────────────────────────────────────────────────────────┘

transformProductForUpdate(product):

Returns ONLY changed fields:
{
  set: {
    title: "Premium Organic Cotton T-Shirt"
  }
}

NOTE: specs[] are NOT overwritten!

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 6: Sanity PATCH Request                                        │
└─────────────────────────────────────────────────────────────────────┘

POST https://abc123.api.sanity.io/v2024-01-01/data/mutate/production

Body:
{
  "mutations": [{
    "patch": {
      "id": "prod_01ABC123",
      "set": {
        "title": "Premium Organic Cotton T-Shirt"
      }
    }
  }]
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 7: Sanity Updates Document                                     │
└─────────────────────────────────────────────────────────────────────┘

Updated Document:
{
  "_id": "prod_01ABC123",
  "title": "Premium Organic Cotton T-Shirt",  // UPDATED
  "specs": [...]  // PRESERVED - not touched!
}

✓ Title synced
✓ Content preserved
✓ Both systems now have new title
```

### What Gets Updated

| Field | Synced? | Notes |
|-------|---------|-------|
| Title | ✅ Yes | Updates on product.updated |
| Specs | ❌ No | Preserved in Sanity |
| Addons | ❌ No | Managed only in Sanity |

---

## Scenario 5: Deleting a Product

**User Action**: Admin deletes product in Medusa

**Important**: This integration does NOT automatically delete from Sanity. You need to implement this if desired.

### Current Behavior

```
Product Deleted in Medusa
         │
         ▼
Event: "product.deleted"  
         │
         ▼
No subscriber configured for this event
         │
         ▼
Product remains in Sanity
         │
         ▼
Storefront won't find product in Medusa
         │
         ▼
404 error on product page
```

### Recommended Implementation

If you want auto-delete in Sanity:

```typescript
// Add to subscribers
export default async function deleteSanityProduct({
  event: { data },
  container,
}: SubscriberArgs<{ id: string }>) {
  const sanityModule = container.resolve(SANITY_MODULE);
  
  await sanityModule.delete(data.id);
}

export const config: SubscriberConfig = {
  event: ["product.deleted"],
};
```

---

## Scenario 6: Bulk Sync Operation

**User Action**: Admin triggers bulk sync of all products

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Admin Triggers Bulk Sync                                    │
└─────────────────────────────────────────────────────────────────────┘

Admin Action:
POST /admin/sanity/syncs

Or custom admin UI button

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Workflow Starts with Empty Input                            │
└─────────────────────────────────────────────────────────────────────┘

Input: {} // Empty = ALL products

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Batch Processing                                            │
└─────────────────────────────────────────────────────────────────────┘

Batch 1 (products 1-200):
├── Query 200 products
├── Sync to Sanity (parallel)
└── Total: 200

Batch 2 (products 201-400):
├── Query 200 products
├── Sync to Sanity (parallel)
└── Total: 400

... continues until all products synced

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Completion                                                   │
└─────────────────────────────────────────────────────────────────────┘

Workflow Returns:
{
  total: 1250  // Total products synced
}
```

### Use Cases for Bulk Sync

1. **Initial Setup**: Sync existing catalog to Sanity
2. **Recovery**: Resync after issues
3. **Testing**: Verify sync logic
4. **Migration**: Move from different CMS

---

## Scenario 7: Manual Sync from Admin

**User Action**: Admin manually syncs single product from Medusa Admin UI

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Admin Opens Product                                         │
└─────────────────────────────────────────────────────────────────────┘

Admin navigates to product detail page in Medusa Admin

Sees Sanity widget showing:
├── Status Badge: "Synced" or "Not Synced"
├── Sync Button
└── Link to Sanity Studio

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Click "Sync" Button                                         │
└─────────────────────────────────────────────────────────────────────┘

Widget calls API:
POST /admin/sanity/documents/prod_01ABC123/sync

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Triggers Workflow for Single Product                        │
└─────────────────────────────────────────────────────────────────────┘

Same workflow as automatic sync, but:
Input: { product_ids: ["prod_01ABC123"] }

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Widget Updates                                              │
└─────────────────────────────────────────────────────────────────────┘

After sync completes:
├── Refetches Sanity document
├── Updates status badge
└── Shows success toast: "Sync triggered"
```

---

## Scenario 8: Adding Product Addons

**User Action**: Content manager adds related products in Sanity Studio

### Step-by-Step Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Open Product in Studio                                      │
└─────────────────────────────────────────────────────────────────────┘

Content manager edits "Premium Cotton T-Shirt"

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Add Addons Section                                          │
└─────────────────────────────────────────────────────────────────────┘

Clicks "Add addons"
├── Title: "Complete Your Look"
└── Adds references to other products:
    - Slim Fit Jeans
    - Casual Sneakers
    - Baseball Cap

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Sanity Stores References                                    │
└─────────────────────────────────────────────────────────────────────┘

Document updated with:
{
  "addons": {
    "title": "Complete Your Look",
    "products": [
      { "_ref": "prod_jeans_001", "_type": "reference" },
      { "_ref": "prod_shoes_002", "_type": "reference" },
      { "_ref": "prod_hat_003", "_type": "reference" }
    ]
  }
}

      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Storefront Displays Addons                                  │
└─────────────────────────────────────────────────────────────────────┘

On product page, fetch addons with GROQ:

const addons = await client.fetch(`
  *[_id == $id][0].addons {
    title,
    products[]-> {
      _id,
      title
    }
  }
`, { id: productId })

Then fetch Medusa data for each addon product to get prices/inventory

Displays:
"Complete Your Look"
├── [Image] Slim Fit Jeans - $79.99
├── [Image] Casual Sneakers - $89.99
└── [Image] Baseball Cap - $24.99
```

---

## Data Consistency Considerations

### What Happens If...

**Q: Sanity API is down during sync?**
A: Workflow fails, retry automatically, or sync later manually

**Q: Product deleted in Medusa but exists in Sanity?**
A: Storefront shows 404 (Medusa check happens first)

**Q: Title different in Medusa vs Sanity?**
A: Medusa title shown (not using Sanity title on storefront in this example)

**Q: Content exists in Sanity but product not in Medusa?**
A: Content orphaned (consider cleanup script)

### Best Practices

1. **Monitor Sync Failures**: Check workflow executions regularly
2. **Periodic Bulk Sync**: Run weekly to ensure consistency
3. **Orphan Cleanup**: Script to find Sanity docs without Medusa products
4. **Backup Strategy**: Sanity has built-in versioning
5. **Testing**: Always test sync in development first

---

## Next Steps

- **[Implementation Guide](./04-implementation.md)** - Set up the integration
- **[Pros & Cons](./05-pros-cons.md)** - Evaluate if this is right for you
- **[Architecture](./02-architecture.md)** - Understand the technical details

