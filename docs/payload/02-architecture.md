# Payload Integration - Architecture Deep Dive

## System Components

### 1. Medusa Backend (Port 9000)

#### Payload Module (`src/modules/payload`)

A custom Medusa module that acts as an HTTP client for Payload CMS.

**Purpose**: Provide a service layer for communicating with Payload's REST API

**Key Files**:
```
src/modules/payload/
├── index.ts          # Module definition
├── service.ts        # PayloadModuleService class
└── types.ts          # TypeScript type definitions
```

**Service Methods**:

```typescript
class PayloadModuleService {
  // Create a new item in Payload collection
  async create(collection: string, data: PayloadUpsertData): Promise<PayloadItemResult>
  
  // Update an item in Payload collection
  async update(collection: string, data: PayloadUpsertData): Promise<PayloadItemResult>
  
  // Delete items from Payload collection
  async delete(collection: string, options: PayloadQueryOptions): Promise<PayloadApiResponse>
  
  // Find items in Payload collection
  async find(collection: string, options: PayloadQueryOptions): Promise<PayloadBulkResult>
  
  // List items with filtering
  async list(filter: { product_id: string | string[] }): Promise<PayloadItem[]>
}
```

**Authentication**:
- Uses API Key authentication
- Header format: `Authorization: users API-Key {YOUR_API_KEY}`
- All requests include `is_from_medusa: true` query parameter

#### Link Definition (`src/links/product-payload.ts`)

Defines a read-only relationship between Medusa products and Payload products.

```typescript
defineLink(
  { linkable: ProductModule.linkable.product, field: "id" },
  { 
    linkable: {
      serviceName: PAYLOAD_MODULE,
      alias: "payload_product",
      primaryKey: "product_id"
    }
  },
  { readOnly: true }
)
```

**What This Does**:
- Creates a relationship where `Product.id` maps to `PayloadProduct.product_id`
- Allows querying products with their Payload data: `fields: ["*payload_product"]`
- Read-only: Changes in Payload don't automatically update Medusa

#### Event Subscribers (`src/subscribers`)

Event-driven sync mechanism that listens to Medusa events and triggers Payload workflows.

```
src/subscribers/
├── product-created.ts           # New product → sync to Payload
├── product-deleted.ts           # Deleted product → delete from Payload
├── variant-created.ts           # New variant → sync to Payload
├── variant-updated.ts           # Updated variant → sync to Payload
├── variant-deleted.ts           # Deleted variant → delete from Payload
├── option-created.ts            # New option → sync to Payload
├── option-deleted.ts            # Deleted option → delete from Payload
└── products-sync-payload.ts     # Bulk sync trigger
```

**Event Flow Example**:

```
┌──────────────────────────────────────────────────────────────┐
│  PRODUCT CREATION FLOW                                       │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Admin creates product in Medusa                          │
│                    ↓                                          │
│  2. Medusa emits "product.created" event                     │
│                    ↓                                          │
│  3. Subscriber catches event                                 │
│                    ↓                                          │
│  4. Triggers "create-payload-products" workflow              │
│                    ↓                                          │
│  5. Workflow queries product details from Medusa             │
│                    ↓                                          │
│  6. Transforms data to Payload format                        │
│                    ↓                                          │
│  7. Creates product in Payload via API                       │
│                    ↓                                          │
│  8. Stores Payload ID in Medusa product metadata             │
│                    ↓                                          │
│  9. Links product to Payload data                            │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

#### Workflows (`src/workflows`)

Orchestrated multi-step processes for syncing data to Payload.

**Main Workflows**:

1. **createPayloadProductsWorkflow**: Create products in Payload
   ```typescript
   Input: { product_ids: string[] }
   Steps:
   1. Query product details from Medusa
   2. Transform to Payload format
   3. Create in Payload (via createPayloadItemsStep)
   4. Store Payload ID in Medusa metadata
   Output: { items: PayloadProduct[] }
   ```

2. **updatePayloadProductVariantsWorkflow**: Update variants in Payload
   ```typescript
   Input: { variant_ids: string[] }
   Steps:
   1. Query variant details with linked Payload product
   2. Transform variant data
   3. Update Payload product's variants array
   Output: { items: PayloadProduct[] }
   ```

3. **deletePayloadProductsWorkflow**: Delete products from Payload
   ```typescript
   Input: { product_ids: string[] }
   Steps:
   1. Transform to delete query
   2. Delete from Payload by medusa_id
   Output: void
   ```

**Workflow Steps** (`src/workflows/steps`):

```
steps/
├── create-payload-items.ts      # HTTP POST to Payload
├── update-payload-items.ts      # HTTP PATCH to Payload
├── retrieve-payload-items.ts    # HTTP GET from Payload
└── delete-payload-items.ts      # HTTP DELETE to Payload
```

Each step includes compensation logic (rollback on failure).

#### Admin API (`src/api/admin/payload/sync/[collection]/route.ts`)

Manual sync trigger endpoint.

```typescript
POST /admin/payload/sync/{collection}

Example: POST /admin/payload/sync/products

Response: {
  message: "Syncing products with Payload"
}
```

**Behavior**:
- Emits event: `{collection}.sync-payload`
- Subscriber processes all products without `payload_id` in metadata
- Processes in batches of 1000 products

#### Admin UI (`src/admin/routes/settings/payload/page.tsx`)

Settings page in Medusa Admin for manual synchronization.

**Features**:
- Button to trigger product sync
- Loading state during sync
- Success notification

### 2. Payload CMS (Embedded in Next.js, Port 8000)

#### Configuration (`storefront/src/payload.config.ts`)

```typescript
buildConfig({
  editor: lexicalEditor(),
  collections: [Users, Products, Media],
  secret: process.env.PAYLOAD_SECRET,
  db: postgresAdapter({
    pool: { connectionString: process.env.PAYLOAD_DATABASE_URL }
  }),
  sharp // Image processing
})
```

#### Collections (`storefront/src/collections`)

**Products Collection** (`Products.ts`):

```typescript
{
  slug: 'products',
  fields: [
    { name: 'medusa_id', type: 'text', unique: true, hidden: true },
    { name: 'title', type: 'text', required: true },
    { name: 'subtitle', type: 'text' },
    { name: 'description', type: 'richText' }, // Lexical editor
    { name: 'thumbnail', type: 'upload', relationTo: 'media' },
    { name: 'images', type: 'array', fields: [
      { name: 'image', type: 'upload', relationTo: 'media' }
    ]},
    { name: 'seo', type: 'group', fields: [
      { name: 'meta_title', type: 'text' },
      { name: 'meta_description', type: 'textarea' },
      { name: 'meta_keywords', type: 'text' }
    ]},
    { name: 'options', type: 'array', fields: [...] },
    { name: 'variants', type: 'array', fields: [...] }
  ],
  access: {
    create: ({ req }) => !!req.query.is_from_medusa, // Only Medusa can create
    delete: ({ req }) => !!req.query.is_from_medusa  // Only Medusa can delete
  },
  hooks: {
    beforeChange: [
      // Convert markdown descriptions to Lexical format
      async ({ data, req }) => {
        if (typeof data.description === "string") {
          data.description = convertMarkdownToLexical(data.description)
        }
        return data
      }
    ]
  }
}
```

**Key Access Control**:
- Products can ONLY be created/deleted by Medusa (`is_from_medusa` query param)
- Admins can UPDATE products to add content
- `medusa_id` fields are read-only in UI (hidden and protected)
- Options/variants count cannot be changed (validation prevents this)

**Media Collection** (`Media.ts`):

```typescript
{
  slug: 'media',
  upload: {
    staticDir: 'public',
    imageSizes: [
      { name: 'thumbnail', width: 400, height: 300 },
      { name: 'card', width: 768, height: 1024 },
      { name: 'tablet', width: 1024 }
    ],
    mimeTypes: ['image/*'],
    pasteURL: {
      allowList: [
        { protocol: "https", hostname: "medusa-public-images.s3.eu-west-1.amazonaws.com" }
      ]
    }
  },
  fields: [
    { name: 'alt', type: 'text' }
  ]
}
```

**Features**:
- Automatic image resizing
- Multiple size variants
- Alt text for accessibility
- Can paste URLs from allowed S3 buckets (imports Medusa images)

**Users Collection** (`Users.ts`):

```typescript
{
  slug: 'users',
  auth: {
    useAPIKey: true // Enable API key authentication
  },
  fields: []
}
```

**Purpose**: User management and API key generation for Medusa integration

### 3. Next.js Storefront (Port 8000)

#### Data Fetching (`src/lib/data/products.ts`)

Fetches combined data from Medusa with Payload relations:

```typescript
sdk.client.fetch(`/store/products`, {
  method: "GET",
  query: {
    limit,
    offset,
    region_id: region?.id,
    fields: "*variants.calculated_price,+variants.inventory_quantity,+metadata,+tags,*payload_product"
    //                                                                               ^^^^^^^^^^^^^^^^
    //                                                                               Includes Payload data
  }
})
```

**What Gets Fetched**:
- Base product data from Medusa
- Calculated prices (with region/currency)
- Inventory levels
- Metadata (including `payload_id`)
- Tags
- **Linked Payload product data** (descriptions, images, SEO)

#### Type Definitions (`src/types/global.ts`)

```typescript
export type StoreProductWithPayload = StoreProduct & {
  payload_product?: {
    medusa_id: string
    title: string
    description: any // Lexical rich text data
    thumbnail?: {
      url: string
      // ... other media fields
    }
    images?: Array<{
      id: string
      image: {
        url: string
        // ... other media fields
      }
    }>
    seo?: {
      meta_title?: string
      meta_description?: string
      meta_keywords?: string
    }
    options?: Array<{
      title: string
      medusa_id: string
    }>
    variants?: Array<{
      title: string
      medusa_id: string
      option_values: Array<{
        medusa_id: string
        medusa_option_id: string
        value: string
      }>
    }>
  }
}
```

#### Component Usage Example

```tsx
// Product display component
function ProductInfo({ product }: { product: StoreProductWithPayload }) {
  return (
    <div>
      {/* Use Payload title if available, fallback to Medusa */}
      <h1>{product?.payload_product?.title || product.title}</h1>
      
      {/* Rich text description from Payload */}
      {product?.payload_product?.description && (
        <RichText data={product.payload_product.description} />
      )}
      
      {/* Fallback to plain text if no Payload description */}
      {!product?.payload_product?.description && (
        <p>{product.description}</p>
      )}
      
      {/* Payload thumbnail or Medusa thumbnail */}
      <img 
        src={product.payload_product?.thumbnail?.url || product.thumbnail} 
        alt={product.title}
      />
    </div>
  )
}
```

## Data Synchronization Architecture

### Medusa → Payload Sync Flow

```
┌────────────────────────────────────────────────────────────────────────┐
│                     COMPLETE SYNC ARCHITECTURE                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐                                                   │
│  │  MEDUSA ADMIN   │                                                   │
│  │  Create Product │                                                   │
│  └────────┬────────┘                                                   │
│           │                                                             │
│           ▼                                                             │
│  ┌───────────────────────────────────────────────────────────┐        │
│  │  MEDUSA DATABASE                                          │        │
│  │  • Product record created with ID                         │        │
│  │  • Variants and options created                           │        │
│  └───────────────┬───────────────────────────────────────────┘        │
│                  │                                                     │
│                  │ (triggers event)                                    │
│                  ▼                                                     │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  EVENT BUS                                                 │      │
│  │  Event: "product.created"                                  │      │
│  │  Data: { id: "prod_123" }                                  │      │
│  └───────────────┬────────────────────────────────────────────┘      │
│                  │                                                     │
│                  │ (subscriber listening)                              │
│                  ▼                                                     │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  SUBSCRIBER: product-created.ts                            │      │
│  │  Triggers: createPayloadProductsWorkflow                   │      │
│  └───────────────┬────────────────────────────────────────────┘      │
│                  │                                                     │
│                  ▼                                                     │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  WORKFLOW: create-payload-products                         │      │
│  │                                                             │      │
│  │  Step 1: Query product from Medusa Graph                   │      │
│  │  ┌──────────────────────────────────────────────┐          │      │
│  │  │ useQueryGraphStep({                          │          │      │
│  │  │   entity: "product",                         │          │      │
│  │  │   fields: [id, title, description, variants] │          │      │
│  │  │ })                                            │          │      │
│  │  └──────────────────────────────────────────────┘          │      │
│  │                  ↓                                          │      │
│  │  Step 2: Transform to Payload format                       │      │
│  │  ┌──────────────────────────────────────────────┐          │      │
│  │  │ {                                            │          │      │
│  │  │   medusa_id: product.id,                    │          │      │
│  │  │   title: product.title,                     │          │      │
│  │  │   description: product.description,         │          │      │
│  │  │   options: [...],                           │          │      │
│  │  │   variants: [...]                           │          │      │
│  │  │ }                                            │          │      │
│  │  └──────────────────────────────────────────────┘          │      │
│  │                  ↓                                          │      │
│  │  Step 3: Create in Payload                                 │      │
│  │  ┌──────────────────────────────────────────────┐          │      │
│  │  │ createPayloadItemsStep(data)                 │          │      │
│  │  │   ↓                                           │          │      │
│  │  │ PayloadModuleService.create(                 │          │      │
│  │  │   "products",                                 │          │      │
│  │  │   data                                        │          │      │
│  │  │ )                                             │          │      │
│  │  └──────────────────────────────────────────────┘          │      │
│  └───────────────┬────────────────────────────────────────────┘      │
│                  │                                                     │
│                  │ (HTTP POST)                                         │
│                  ▼                                                     │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  PAYLOAD CMS (via Next.js API Route)                      │      │
│  │  POST /api/products?is_from_medusa=true                    │      │
│  │  Headers: { Authorization: "users API-Key XXX" }           │      │
│  │                                                             │      │
│  │  Access Control Checks:                                    │      │
│  │  ✓ is_from_medusa query param present                      │      │
│  │  ✓ Valid API key                                           │      │
│  │                                                             │      │
│  │  Hooks Execute:                                            │      │
│  │  • beforeChange: Convert markdown → Lexical                │      │
│  └───────────────┬────────────────────────────────────────────┘      │
│                  │                                                     │
│                  ▼                                                     │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  PAYLOAD DATABASE                                          │      │
│  │  • Product record created                                  │      │
│  │  • Returns: { doc: { id: "pay_456", medusa_id: "prod_123" } }│   │
│  └───────────────┬────────────────────────────────────────────┘      │
│                  │                                                     │
│                  │ (response)                                          │
│                  ▼                                                     │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  WORKFLOW (continued)                                      │      │
│  │                                                             │      │
│  │  Step 4: Update Medusa product metadata                    │      │
│  │  ┌──────────────────────────────────────────────┐          │      │
│  │  │ updateProductsWorkflow.runAsStep({           │          │      │
│  │  │   input: {                                    │          │      │
│  │  │     products: [{                              │          │      │
│  │  │       id: "prod_123",                         │          │      │
│  │  │       metadata: {                             │          │      │
│  │  │         payload_id: "pay_456"                 │          │      │
│  │  │       }                                        │          │      │
│  │  │     }]                                         │          │      │
│  │  │   }                                            │          │      │
│  │  │ })                                             │          │      │
│  │  └──────────────────────────────────────────────┘          │      │
│  └───────────────┬────────────────────────────────────────────┘      │
│                  │                                                     │
│                  ▼                                                     │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  MEDUSA DATABASE (updated)                                 │      │
│  │  • Product.metadata.payload_id = "pay_456"                 │      │
│  │  • Link established for queries                            │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### Payload → Medusa Sync Flow

**Important**: There is NO automatic sync from Payload back to Medusa.

```
┌──────────────────────────────────────────────────────────┐
│  PAYLOAD CONTENT UPDATES (Stay in Payload)               │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  1. Content manager logs into Payload Admin              │
│                    ↓                                      │
│  2. Edits product:                                       │
│     • Updates rich text description                      │
│     • Uploads new images                                 │
│     • Adds SEO metadata                                  │
│     • Updates title/subtitle                             │
│                    ↓                                      │
│  3. Saves changes in Payload                             │
│                    ↓                                      │
│  4. Changes stored in Payload database                   │
│                    ↓                                      │
│  5. NO sync back to Medusa ❌                            │
│                    ↓                                      │
│  6. Storefront fetches updated data from Payload         │
│     via linked query                                     │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Why No Reverse Sync?**
- Payload manages content, Medusa manages commerce
- Content changes (descriptions, SEO) don't affect commerce operations
- Prevents circular updates and conflicts
- Medusa title/description remain as fallbacks

**Exception - What CAN'T Be Changed in Payload**:
- Number of options (validated)
- Number of variants (validated)
- `medusa_id` fields (access controlled)
- Creating new products (only Medusa can create)
- Deleting products (only Medusa can delete)

## Database Architecture

### Two Separate Databases

```
┌─────────────────────────────────────────────────────────────┐
│                    DATABASE ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────┐                           │
│  │  MEDUSA DATABASE              │                           │
│  │  (PostgreSQL)                 │                           │
│  │                               │                           │
│  │  Tables:                      │                           │
│  │  • product                    │                           │
│  │    - id (primary key)         │                           │
│  │    - title                    │                           │
│  │    - description              │                           │
│  │    - metadata (JSON)          │                           │
│  │      └─ payload_id ───────────┼───────┐                  │
│  │                               │       │                   │
│  │  • product_variant            │       │                   │
│  │  • product_option             │       │                   │
│  │  • price_list                 │       │                   │
│  │  • inventory_item             │       │                   │
│  │  • order                      │       │ (link)            │
│  │  • customer                   │       │                   │
│  │  • ... 50+ commerce tables    │       │                   │
│  └──────────────────────────────┘       │                   │
│                                           │                   │
│  ┌──────────────────────────────┐       │                   │
│  │  PAYLOAD DATABASE             │       │                   │
│  │  (PostgreSQL - separate)      │       │                   │
│  │                               │       │                   │
│  │  Tables:                      │       │                   │
│  │  • products                   │       │                   │
│  │    - id (primary key) ◄───────┼───────┘                  │
│  │    - medusa_id (unique)       │                           │
│  │    - title                    │                           │
│  │    - description (JSON)       │                           │
│  │    - _status                  │                           │
│  │    - ... Payload system cols  │                           │
│  │                               │                           │
│  │  • media                      │                           │
│  │  • users                      │                           │
│  │  • payload_preferences        │                           │
│  │  • payload_migrations         │                           │
│  │  • ... Payload system tables  │                           │
│  └──────────────────────────────┘                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Key Points**:
- Separate databases = independent scaling and backups
- `payload_id` in Medusa metadata links to Payload's `id`
- `medusa_id` in Payload links to Medusa's product `id`
- Link is logical, not a database foreign key

## API Communication

### Medusa → Payload API Calls

```typescript
// Authentication Header
Authorization: users API-Key {YOUR_API_KEY}

// Query Parameter (for access control)
?is_from_medusa=true

// Example: Create Product
POST http://localhost:8000/api/products?is_from_medusa=true
{
  "medusa_id": "prod_01JH1234567890",
  "title": "Awesome T-Shirt",
  "subtitle": "100% Cotton",
  "description": "A comfortable t-shirt...",
  "options": [
    { "title": "Size", "medusa_id": "opt_01JH..." }
  ],
  "variants": [
    {
      "title": "Small",
      "medusa_id": "variant_01JH...",
      "option_values": [
        { "medusa_id": "optval_01JH...", "value": "Small" }
      ]
    }
  ]
}

// Response
{
  "doc": {
    "id": "67890abcdef",
    "medusa_id": "prod_01JH1234567890",
    "title": "Awesome T-Shirt",
    // ... other fields
  }
}
```

### Storefront → Medusa API (with Payload data)

```typescript
// Request
GET http://localhost:9000/store/products?fields=*payload_product

// Response
{
  "products": [
    {
      "id": "prod_01JH1234567890",
      "title": "Awesome T-Shirt", // from Medusa
      "thumbnail": "https://...", // from Medusa
      "variants": [...],
      "metadata": {
        "payload_id": "67890abcdef"
      },
      // Linked Payload data (via remote link)
      "payload_product": {
        "id": "67890abcdef",
        "medusa_id": "prod_01JH1234567890",
        "title": "Amazing T-Shirt", // Updated in Payload
        "description": { /* Lexical rich text */ },
        "thumbnail": {
          "url": "/media/tshirt.jpg",
          "alt": "Tshirt front view"
        },
        "seo": {
          "meta_title": "Buy Amazing T-Shirt | YourStore",
          "meta_description": "..."
        }
      }
    }
  ]
}
```

## Next: Data Flow Details

For a step-by-step breakdown of how data moves through the system in various scenarios, see [Data Flow](./03-data-flow.md).

