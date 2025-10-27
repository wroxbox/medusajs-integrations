# Sanity CMS Integration - Architecture Deep Dive

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          COMPLETE SYSTEM ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────┐          ┌──────────────────────────────────┐   │
│  │   MEDUSA BACKEND      │          │   SANITY (EXTERNAL SAAS)         │   │
│  │   localhost:9000      │          │   api.sanity.io                  │   │
│  │                       │          │                                  │   │
│  │  ┌─────────────────┐ │          │  ┌────────────────────────────┐ │   │
│  │  │  PostgreSQL DB  │ │          │  │   Content Lake (Storage)   │ │   │
│  │  │                 │ │          │  │                            │ │   │
│  │  │  • Products     │ │          │  │  • Product Documents       │ │   │
│  │  │  • Variants     │ │          │  │    - Specs (localized)     │ │   │
│  │  │  • Prices       │ │          │  │    - Addons                │ │   │
│  │  │  • Orders       │ │          │  │  • Asset Storage           │ │   │
│  │  │  • Inventory    │ │          │  │  • Version History         │ │   │
│  │  └─────────────────┘ │          │  └────────────────────────────┘ │   │
│  │           ▲           │          │                │                │   │
│  │           │           │          │                ▼                │   │
│  │  ┌────────┴────────┐ │          │  ┌────────────────────────────┐ │   │
│  │  │  Medusa Core    │ │◄────────►│  │   Sanity API               │ │   │
│  │  │                 │ │   HTTP   │  │                            │ │   │
│  │  │  • Event Bus    │ │          │  │  • REST API                │ │   │
│  │  │  • Modules      │ │          │  │  • GROQ Engine             │ │   │
│  │  │  • Workflows    │ │          │  │  • Real-time Listener      │ │   │
│  │  └────────┬────────┘ │          │  │  • CDN (assets)            │ │   │
│  │           │           │          │  └──────────┬─────────────────┘ │   │
│  │  ┌────────▼────────┐ │          │             │                   │   │
│  │  │ Custom Modules  │ │          │             │                   │   │
│  │  │                 │ │          └─────────────┼───────────────────┘   │
│  │  │ • Sanity Module │ │                        │                       │
│  │  │   (Client)      │──────────────────────────┘                       │
│  │  │                 │ │     @sanity/client                             │
│  │  └─────────────────┘ │                                                │
│  │           │           │                                                │
│  │  ┌────────▼────────┐ │                                                │
│  │  │  Subscribers    │ │                                                │
│  │  │                 │ │                                                │
│  │  │ • Product Sync  │ │                                                │
│  │  └─────────────────┘ │                                                │
│  │           │           │                                                │
│  │  ┌────────▼────────┐ │                                                │
│  │  │   Workflows     │ │                                                │
│  │  │                 │ │                                                │
│  │  │ • Sync Products │ │                                                │
│  │  └─────────────────┘ │                                                │
│  │           │           │                                                │
│  │  ┌────────▼────────┐ │                                                │
│  │  │     Links       │ │                                                │
│  │  │                 │ │                                                │
│  │  │ • Product↔Sanity│ │                                                │
│  │  └─────────────────┘ │                                                │
│  │                       │                                                │
│  └───────────┬───────────┘                                                │
│              │                                                            │
│              │  REST API                                                  │
│              │                                                            │
│  ┌───────────▼─────────────────────────────────────────┐                │
│  │   NEXT.JS STOREFRONT                                │                │
│  │   localhost:8000                                    │                │
│  │                                                     │                │
│  │  ┌──────────────────┐  ┌─────────────────────────┐│                │
│  │  │  Sanity Studio   │  │   Storefront Pages      ││                │
│  │  │  /studio         │  │                         ││                │
│  │  │                  │  │  ┌──────────────────┐   ││                │
│  │  │  • Visual Editor │  │  │  Product Page    │   ││                │
│  │  │  • Schema Config │  │  │                  │   ││                │
│  │  │  • Content Mgmt  │  │  │  1. Fetch from   │   ││                │
│  │  │  • Collaboration │  │  │     Medusa API   │   ││                │
│  │  │                  │  │  │  2. Fetch from   │   ││                │
│  │  └──────────────────┘  │  │     Sanity API   │   ││                │
│  │                        │  │  3. Merge & Show │   ││                │
│  │  ┌──────────────────┐  │  └──────────────────┘   ││                │
│  │  │  Sanity Client   │  │                         ││                │
│  │  │  (next-sanity)   │──┼──► Fetches content     ││                │
│  │  └──────────────────┘  │    directly             ││                │
│  │                        │                         ││                │
│  │  ┌──────────────────┐  │                         ││                │
│  │  │  Medusa SDK      │──┼──► Fetches commerce    ││                │
│  │  └──────────────────┘  │    data                 ││                │
│  │                        └─────────────────────────┘│                │
│  └─────────────────────────────────────────────────────┘                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Component Breakdown

### 1. Medusa Backend Components

#### 1.1 Sanity Module

**File**: `medusa/src/modules/sanity/index.ts`, `service.ts`

**Purpose**: Custom Medusa module that wraps `@sanity/client` to provide Sanity API integration.

**Implementation**:

```typescript
// Module definition
import { Module } from "@medusajs/framework/utils";
import SanityModuleService from "./service";

export const SANITY_MODULE = "sanity";

export default Module(SANITY_MODULE, {
  service: SanityModuleService,
});
```

**Service Class Structure**:

```typescript
class SanityModuleService {
  private client: SanityClient              // Sanity API client
  private studioUrl?: string                 // Studio URL for admin links
  private logger: Logger                     // Medusa logger
  private typeMap: Record<string, string>   // Type mappings
  
  constructor(dependencies, options) {
    // Initialize @sanity/client
    this.client = createClient({
      projectId: options.project_id,
      apiVersion: options.api_version,
      dataset: options.dataset,
      token: options.api_token               // Required: Editor permissions
    })
  }
  
  // Core methods
  async upsertSyncDocument(type, data)      // Create or update
  async createSyncDocument(type, data)      // Create only
  async updateSyncDocument(type, data)      // Update only
  async list(filter)                         // Query documents
  async retrieve(id)                         // Get by ID
  async delete(id)                           // Delete document
  async update(id, data)                     // Patch document
  async getStudioLink(type, id)             // Generate Studio URL
  
  // Transform methods
  private transformProductForCreate()        // Medusa → Sanity (create)
  private transformProductForUpdate()        // Medusa → Sanity (update)
}
```

**Configuration** (`medusa-config.ts`):

```typescript
modules: [
  {
    resolve: "./src/modules/sanity",
    options: {
      api_token: process.env.SANITY_API_TOKEN,          // From manage.sanity.io
      project_id: process.env.SANITY_PROJECT_ID,        // From Sanity project
      api_version: "2024-01-01",                         // Current date format
      dataset: "production",                             // Or "development"
      studio_url: "http://localhost:8000/studio",       // Studio location
      type_map: {
        product: "product"                               // Medusa type → Sanity type
      }
    }
  }
]
```

**Key Features**:

1. **Upsert Logic**: Checks if document exists before deciding create vs update
2. **Transformation**: Converts Medusa DTOs to Sanity document format
3. **Error Handling**: Logs and throws for debugging
4. **Studio Links**: Generates deep links to edit documents in Studio

**Transform Example**:

```typescript
// Medusa Product DTO
{
  id: "prod_01ABCDEFG",
  title: "Premium T-Shirt",
  // ... other Medusa fields
}

// Transformed to Sanity Document
{
  _id: "prod_01ABCDEFG",           // Same as Medusa ID
  _type: "product",
  title: "Premium T-Shirt",
  specs: [
    {
      _key: "prod_01ABCDEFG",      // Unique key for array item
      _type: "spec",
      title: "Premium T-Shirt",    // Initial title
      lang: "en",                   // Default language
      content: ""                   // Empty - filled in Sanity
    }
  ]
}
```

#### 1.2 Event Subscriber

**File**: `medusa/src/subscribers/sanity-product-sync.ts`

**Purpose**: Listen for Medusa product events and trigger sync workflow.

**Implementation**:

```typescript
export default async function upsertSanityProduct({
  event: { data },
  container,
}: SubscriberArgs<{ id: string }>) {
  // Trigger workflow with product ID
  await sanitySyncProductsWorkflow(container).run({
    input: {
      product_ids: [data.id],    // Single product
    },
  });
}

export const config: SubscriberConfig = {
  event: ["product.created", "product.updated"],  // Listen to both events
};
```

**Event Flow**:

```
Product Created/Updated in Admin
         │
         ▼
Event Bus emits event
         │
         ▼
Subscriber catches event
         │
         ▼
Extract product ID
         │
         ▼
Trigger workflow with ID
```

**Characteristics**:
- **Automatic**: No manual trigger needed
- **Asynchronous**: Doesn't block product creation
- **Retryable**: Workflow can retry on failure
- **Selective**: Can sync specific products or all

#### 1.3 Sync Workflow

**File**: `medusa/src/workflows/sanity-sync-products/index.ts`, `steps/sync.ts`

**Purpose**: Orchestrate the sync process with error handling and rollback.

**Workflow Definition**:

```typescript
export const sanitySyncProductsWorkflow = createWorkflow(
  { 
    name: "sanity-sync-products", 
    retentionTime: 10000    // Keep execution logs for debugging
  },
  function (input: SanitySyncProductsWorkflowInput) {
    const result = syncStep(input);
    return new WorkflowResponse(result);
  }
);
```

**Sync Step Implementation**:

```typescript
export const syncStep = createStep(
  { name: "sync-step", async: true },
  
  // Execution function
  async (input: { product_ids?: string[] }, { container }) => {
    const sanityModule = container.resolve(SANITY_MODULE);
    const query = container.resolve(ContainerRegistrationKeys.QUERY);
    
    let total = 0;
    const upsertMap = [];  // Track changes for rollback
    
    // Batch processing (200 products at a time)
    const batchSize = 200;
    let offset = 0;
    let hasMore = true;
    
    while (hasMore) {
      // Query products from Medusa
      const { data: products, metadata: { count } } = await query.graph({
        entity: "product",
        fields: ["id", "title", "sanity_product.*"],
        filters: input.product_ids ? { id: input.product_ids } : {},
        pagination: { skip: offset, take: batchSize, order: { id: "ASC" } }
      });
      
      // Sync each product to Sanity
      await promiseAll(
        products.map(async (prod) => {
          const after = await sanityModule.upsertSyncDocument("product", prod);
          upsertMap.push({ before: prod.sanity_product, after });
          return after;
        })
      );
      
      offset += batchSize;
      hasMore = offset < count;
      total += products.length;
    }
    
    return new StepResponse({ total }, upsertMap);
  },
  
  // Compensation function (rollback)
  async (upsertMap, { container }) => {
    if (!upsertMap) return;
    
    const sanityModule = container.resolve(SANITY_MODULE);
    
    // Rollback changes
    await promiseAll(
      upsertMap.map(({ before, after }) => {
        if (!before) {
          // Was created, so delete
          return sanityModule.delete(after._id);
        }
        // Was updated, restore old data
        const { _id: id, ...oldData } = before;
        return sanityModule.update(id, oldData);
      })
    );
  }
);
```

**Workflow Features**:

1. **Batch Processing**: Handles large product catalogs (200 at a time)
2. **Error Handling**: Permanent failure on errors
3. **Rollback Support**: Compensating function to undo changes
4. **Progress Tracking**: Returns total synced count
5. **Selective Sync**: Can sync specific products or all
6. **Relationship Loading**: Includes existing Sanity link data

**Execution Modes**:

| Mode | Trigger | Input |
|------|---------|-------|
| **Single Product** | Event subscriber | `{ product_ids: ["prod_123"] }` |
| **Bulk Sync** | Admin API | `{}` (all products) |
| **Manual Sync** | Admin button | `{ product_ids: ["prod_123"] }` |

#### 1.4 Link Definition

**File**: `medusa/src/links/product-sanity.ts`

**Purpose**: Define relationship between Medusa products and Sanity documents.

**Implementation**:

```typescript
import { defineLink } from "@medusajs/framework/utils";
import ProductModule from "@medusajs/medusa/product";
import { SANITY_MODULE } from "../modules/sanity";

defineLink(
  {
    linkable: ProductModule.linkable.product.id,  // Medusa product
    field: "id",                                    // Link field
  },
  {
    linkable: {
      serviceName: SANITY_MODULE,                  // Custom module
      alias: "sanity_product",                     // Query alias
      primaryKey: "id",                            // Link key
    },
  },
  {
    readOnly: true,                                // One-way only
  }
);
```

**Link Characteristics**:

- **One-way**: Medusa → Sanity only
- **Read-only**: Cannot create/update via link
- **Queryable**: Can include in Medusa queries
- **Not used in storefront**: Storefront fetches from Sanity directly

**Query Usage** (in Medusa Admin):

```typescript
// In workflow or admin API
const { data: products } = await query.graph({
  entity: "product",
  fields: [
    "id",
    "title",
    "sanity_product.*"    // Load linked Sanity document
  ]
});

// Result includes Sanity data
products[0].sanity_product = {
  _id: "prod_123",
  title: "Product Title",
  specs: [...]
}
```

**Why Link Exists**:
1. **Admin UI**: Display Sanity sync status
2. **Rollback**: Access previous state for compensation
3. **Future**: Could enable integrated queries
4. **Consistency**: Follow Medusa patterns

**Why Not Used in Storefront**:
1. Adds latency (Medusa as proxy)
2. Limited GROQ query capabilities
3. Sanity CDN benefits lost
4. More control with direct Sanity client

#### 1.5 Admin API Routes

**Sync All Products**: `POST /admin/sanity/syncs`

**File**: `medusa/src/api/admin/sanity/syncs/route.ts`

```typescript
export const POST = async (req: MedusaRequest, res: MedusaResponse) => {
  // Trigger workflow for ALL products
  const { transaction } = await sanitySyncProductsWorkflow(req.scope).run({
    input: {},  // Empty = all products
  });
  
  res.json({ transaction_id: transaction.transactionId });
};

export const GET = async (req: MedusaRequest, res: MedusaResponse) => {
  // List workflow executions for monitoring
  const workflowEngine = req.scope.resolve(Modules.WORKFLOW_ENGINE);
  
  const [executions, count] = await workflowEngine.listAndCountWorkflowExecutions({
    workflow_id: sanitySyncProductsWorkflow.getName(),
  }, { order: { created_at: "DESC" } });
  
  res.json({ workflow_executions: executions, count });
};
```

**Sync Single Product**: `POST /admin/sanity/documents/:id/sync`

**File**: `medusa/src/api/admin/sanity/documents/[id]/sync/route.ts`

```typescript
export const POST = async (req: MedusaRequest, res: MedusaResponse) => {
  // Trigger workflow for specific product
  const { transaction } = await sanitySyncProductsWorkflow(req.scope).run({
    input: { product_ids: [req.params.id] },
  });
  
  res.json({ transaction_id: transaction.transactionId });
};
```

**Get Sanity Document**: `GET /admin/sanity/documents/:id`

**File**: `medusa/src/api/admin/sanity/documents/[id]/route.ts`

```typescript
export const GET = async (req: MedusaRequest, res: MedusaResponse) => {
  const sanityModule = req.scope.resolve(SANITY_MODULE);
  
  // Fetch document from Sanity
  const sanityDocument = await sanityModule.retrieve(req.params.id);
  
  // Generate Studio link
  const url = sanityDocument ? 
    await sanityModule.getStudioLink(
      sanityDocument._type,
      sanityDocument._id,
      { explicit_type: true }
    ) : "";
  
  res.json({ sanity_document: sanityDocument, studio_url: url });
};
```

#### 1.6 Admin UI Widgets

**File**: `medusa/src/admin/widgets/sanity-product.tsx`

**Purpose**: Display Sanity sync status in product detail page.

**Features**:
- Show sync status badge (Synced / Not Synced)
- Manual sync button
- View Sanity document JSON
- Link to open in Sanity Studio

**Implementation**:

```typescript
const ProductWidget = ({ data }: DetailWidgetProps<AdminProduct>) => {
  const { mutateAsync, isPending } = useTriggerSanityProductSync(data.id);
  const { sanity_document, studio_url, isLoading } = useSanityDocument(data.id);
  
  return (
    <Container>
      <StatusBadge color={sanity_document?.title === data.title ? "green" : "red"}>
        {isLoading ? "Loading..." : sanity_document ? "Synced" : "Not Synced"}
      </StatusBadge>
      
      <Button onClick={handleSync}>Sync</Button>
      
      {studio_url && (
        <a href={studio_url} target="_blank">
          <Button>Open in Sanity Studio</Button>
        </a>
      )}
      
      <CodeBlock code={JSON.stringify(sanity_document, null, 2)} />
    </Container>
  );
};

export const config = defineWidgetConfig({
  zone: "product.details.after",  // Position in product detail page
});
```

### 2. Sanity Components

#### 2.1 Sanity Project Structure

```
Sanity Project (manage.sanity.io)
│
├── Datasets
│   ├── production     (default)
│   └── development    (optional)
│
├── API Tokens
│   ├── Editor Token   (for Medusa sync)
│   └── Viewer Token   (for storefront read)
│
├── CORS Origins
│   ├── http://localhost:9000   (Medusa)
│   └── http://localhost:8000   (Storefront)
│
└── Members & Roles
    ├── Admin (full access)
    ├── Editor (content only)
    └── Viewer (read-only)
```

#### 2.2 Content Lake

Sanity's **Content Lake** is a real-time, globally distributed document database.

**Features**:
- **Real-time sync**: Changes propagate instantly
- **Versioning**: Full history of all changes
- **GROQ queries**: Powerful query language
- **CDN delivery**: Fast global content delivery
- **Transactions**: ACID compliance
- **Rich data types**: References, arrays, objects

**Document Example**:

```json
{
  "_id": "prod_01ABCDEFG",
  "_type": "product",
  "_createdAt": "2024-01-15T10:00:00Z",
  "_updatedAt": "2024-01-16T14:30:00Z",
  "_rev": "abc123-xyz789",
  
  "title": "Premium Cotton T-Shirt",
  
  "specs": [
    {
      "_key": "en_001",
      "_type": "spec",
      "lang": "en",
      "title": "Premium Cotton T-Shirt",
      "content": "Made from 100% organic cotton, this premium t-shirt offers unmatched comfort and durability. Perfect for everyday wear."
    },
    {
      "_key": "es_002",
      "_type": "spec",
      "lang": "es",
      "title": "Camiseta de Algodón Premium",
      "content": "Hecha de algodón 100% orgánico, esta camiseta premium ofrece comodidad y durabilidad incomparables."
    }
  ],
  
  "addons": {
    "title": "Complete Your Look",
    "products": [
      { "_ref": "prod_jeans_001", "_type": "reference" },
      { "_ref": "prod_shoes_002", "_type": "reference" },
      { "_ref": "prod_hat_003", "_type": "reference" }
    ]
  }
}
```

### 3. Storefront Components

#### 3.1 Sanity Studio

**Location**: `storefront/studio` route (mounted at `/studio`)

**Configuration**: `storefront/sanity.config.ts`

```typescript
export default defineConfig({
  basePath: '/studio',
  projectId: process.env.NEXT_PUBLIC_SANITY_PROJECT_ID,
  dataset: 'production',
  
  schema: {
    types: [productSchema]  // Import your schemas
  },
  
  plugins: [
    structureTool({ structure }),  // Custom navigation
    visionTool()                    // GROQ query tool
  ]
})
```

**Features**:
- Embedded in Next.js app (no separate deployment)
- Real-time collaboration
- Custom input components
- Validation rules
- Preview panes
- Document actions

#### 3.2 Sanity Client (Storefront)

**File**: `storefront/src/sanity/lib/client.ts`

```typescript
import { createClient } from 'next-sanity'

export const client = createClient({
  projectId: process.env.NEXT_PUBLIC_SANITY_PROJECT_ID,
  dataset: 'production',
  apiVersion: '2024-01-01',
  useCdn: true  // Use CDN for production
})
```

**Usage in Pages**:

```typescript
// Fetch specific document
const sanity = await client.getDocument(productId)

// GROQ query
const addons = await client.fetch(
  `*[_type == "product" && _id == $id][0].addons`,
  { id: productId }
)
```

#### 3.3 Product Schema

**File**: `storefront/src/sanity/schemaTypes/documents/product.ts`

```typescript
const productSchema: DocumentDefinition = {
  name: "product",
  type: "document",
  title: "Product Page",
  
  fields: [
    {
      name: "title",
      type: "string",
      title: "Title",
      validation: Rule => Rule.required()
    },
    {
      name: "specs",
      type: "array",
      title: "Specifications",
      group: "content",
      of: [{
        type: "object",
        name: "spec",
        fields: [
          { name: "lang", type: "string", title: "Language" },
          { name: "title", type: "string", title: "Title" },
          { name: "content", type: "text", rows: 3, title: "Content" }
        ]
      }]
    },
    {
      name: "addons",
      type: "object",
      title: "Add-ons",
      fields: [
        { name: "title", type: "string" },
        {
          name: "products",
          type: "array",
          of: [{ 
            type: "reference", 
            to: [{ type: "product" }]
          }],
          validation: Rule => Rule.max(3)
        }
      ]
    }
  ],
  
  groups: [{
    name: "content",
    title: "Content",
    default: true
  }],
  
  preview: {
    select: { title: "title" }
  }
}
```

## Data Flow Architecture

### Sync Flow (Medusa → Sanity)

```
┌──────────────────────────────────────────────────────────────────┐
│                     PRODUCT SYNC FLOW                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Admin creates/updates product in Medusa                     │
│       ↓                                                           │
│  [2] Product saved to PostgreSQL                                 │
│       ↓                                                           │
│  [3] Event Bus emits event:                                      │
│      • product.created OR                                        │
│      • product.updated                                           │
│       ↓                                                           │
│  [4] Subscriber catches event                                    │
│      File: subscribers/sanity-product-sync.ts                    │
│       ↓                                                           │
│  [5] Trigger Workflow                                            │
│      Name: sanity-sync-products                                  │
│      Input: { product_ids: [id] }                                │
│       ↓                                                           │
│  [6] Workflow Step: syncStep                                     │
│      a) Query product from Medusa                                │
│         Fields: id, title, sanity_product.*                      │
│      b) Transform to Sanity format                               │
│      c) Call sanityModule.upsertSyncDocument()                   │
│       ↓                                                           │
│  [7] Sanity Module                                               │
│      a) Check if document exists (getDocument)                   │
│      b) If exists → update (patch)                               │
│         If not → create (create)                                 │
│       ↓                                                           │
│  [8] HTTP Request to Sanity API                                  │
│      POST https://[PROJECT_ID].api.sanity.io/v2024-01-01/data    │
│      Headers: Authorization: Bearer [TOKEN]                      │
│      Body: Sanity document                                       │
│       ↓                                                           │
│  [9] Sanity stores document in Content Lake                      │
│       ↓                                                           │
│  [10] Workflow returns success                                   │
│       { total: 1 }                                               │
│       ↓                                                           │
│  [11] Document available in Sanity Studio ✓                      │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Read Flow (Storefront)

```
┌──────────────────────────────────────────────────────────────────┐
│                     PRODUCT PAGE LOAD FLOW                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Customer visits /products/[handle]                          │
│       ↓                                                           │
│  [2] Next.js Product Page Component                              │
│      File: app/[countryCode]/(main)/products/[handle]/page.tsx   │
│       ↓                                                           │
│  [3] Fetch #1: Get product from Medusa                           │
│      const product = await listProducts({                        │
│        countryCode,                                              │
│        queryParams: { handle }                                   │
│      })                                                           │
│       ↓                                                           │
│  [4] HTTP GET to Medusa API                                      │
│      GET http://localhost:9000/store/products?handle=...         │
│       ↓                                                           │
│  [5] Medusa returns product data:                                │
│      • id, title, description                                    │
│      • variants, prices                                          │
│      • inventory                                                 │
│      • images (from Medusa)                                      │
│       ↓                                                           │
│  [6] Fetch #2: Get content from Sanity                           │
│      const sanity = await client.getDocument(product.id)         │
│       ↓                                                           │
│  [7] HTTP GET to Sanity API                                      │
│      GET https://[PROJECT_ID].api.sanity.io/v2024-01-01/data/    │
│          query/production?query=*[_id==$id][0]                   │
│       ↓                                                           │
│  [8] Sanity returns document:                                    │
│      • specs[] (localized content)                               │
│      • addons{}                                                  │
│       ↓                                                           │
│  [9] Extract specs for current language:                         │
│      const spec = sanity.specs.find(s => s.lang === 'en')       │
│       ↓                                                           │
│  [10] Pass both to template:                                     │
│       <ProductTemplate                                           │
│         product={product}      // From Medusa                    │
│         sanity={spec}          // From Sanity                    │
│       />                                                          │
│       ↓                                                           │
│  [11] Template merges data:                                      │
│       • Title: product.title (Medusa)                            │
│       • Description: sanity.content (Sanity) || product.desc     │
│       • Price: product.price (Medusa)                            │
│       • Variants: product.variants (Medusa)                      │
│       • Add to Cart: Medusa API                                  │
│       ↓                                                           │
│  [12] Render complete product page ✓                             │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

## Security Architecture

### API Tokens

```
Sanity API Tokens (manage.sanity.io → API → Tokens)
│
├── Editor Token (for Medusa Backend)
│   • Permissions: Editor (create, read, update, delete)
│   • Storage: Server-side .env only
│   • Usage: Sync workflow
│   • NEVER expose to frontend
│
└── Viewer Token (Optional, for Storefront)
    • Permissions: Viewer (read-only)
    • Storage: Can be public (if needed)
    • Usage: Private content access
    • Most content: Use CDN without token
```

### CORS Configuration

```
Sanity Project Settings → API → CORS Origins

Allowed Origins:
├── http://localhost:9000     (Medusa dev)
├── http://localhost:8000     (Storefront dev)
├── https://admin.yourdomain.com     (Medusa prod)
├── https://yourdomain.com           (Storefront prod)
└── https://studio.sanity.io         (Sanity Studio hosted)

Settings:
• Allow Credentials: Yes
• Allowed Methods: GET, POST, PUT, DELETE, PATCH
```

## Deployment Architecture

### Development

```
Local Machine
├── Medusa Backend (Port 9000)
│   • PostgreSQL (Docker or local)
│   • Connects to Sanity Cloud
│
├── Next.js + Studio (Port 8000)
│   • Studio at /studio
│   • Connects to Sanity Cloud
│
└── Sanity Cloud (SaaS)
    • Content Lake
    • API
    • CDN
```

### Production

```
Cloud Infrastructure
├── Medusa Backend
│   • Deploy: Railway, Render, AWS, etc.
│   • Database: Managed PostgreSQL
│   • Env: SANITY_API_TOKEN, SANITY_PROJECT_ID
│   • Outbound: HTTPS to Sanity API
│
├── Next.js Storefront
│   • Deploy: Vercel, Netlify, AWS, etc.
│   • Env: NEXT_PUBLIC_SANITY_PROJECT_ID
│   • Studio: Deployed at /studio route
│   • CDN: Automatic (Vercel/Netlify)
│
└── Sanity Cloud (SaaS)
    • Fully managed
    • Global CDN
    • Automatic scaling
    • 99.9% uptime SLA
```

## Performance Considerations

### Optimization Strategies

1. **Caching**:
   - Sanity CDN caches content globally
   - Next.js caches product pages
   - Implement stale-while-revalidate

2. **Batch Sync**:
   - Workflow processes 200 products at once
   - Reduces API calls
   - Faster bulk operations

3. **Selective Fetching**:
   - Only fetch needed Sanity fields
   - Use GROQ projections
   - Minimize data transfer

4. **Parallel Fetching**:
   - Fetch Medusa + Sanity concurrently
   - Use Promise.all()
   - Reduce total latency

## Next Steps

- **[Data Flow Details](./03-data-flow.md)** - See step-by-step scenarios
- **[Implementation Guide](./04-implementation.md)** - Build it yourself
- **[Pros & Cons](./05-pros-cons.md)** - Evaluate trade-offs

