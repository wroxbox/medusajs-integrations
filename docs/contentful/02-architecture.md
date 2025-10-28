# Architecture Deep Dive

This document provides technical details on how the Medusa-Contentful integration works under the hood.

## System Components

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    INTEGRATION ARCHITECTURE                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  MEDUSA BACKEND (Commerce)               Port 9000         │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  medusa-plugin-contentful                           │  │  │
│  │  │                                                      │  │  │
│  │  │  ┌────────────────┐    ┌───────────────────────┐   │  │  │
│  │  │  │ Subscribers    │────│ ContentfulService     │   │  │  │
│  │  │  │ (Event         │    │ • Transform data      │   │  │  │
│  │  │  │  Listeners)    │    │ • HTTP client         │   │  │  │
│  │  │  └────────────────┘    │ • API authentication  │   │  │  │
│  │  │                        │ • Retry logic         │   │  │  │
│  │  │                        └───────────────────────┘   │  │  │
│  │  │                                                      │  │  │
│  │  │  ┌────────────────┐    ┌───────────────────────┐   │  │  │
│  │  │  │ Webhook        │────│ WebhookHandler        │   │  │  │
│  │  │  │ Routes         │    │ • Receive from        │   │  │  │
│  │  │  │ /hooks/*       │    │   Contentful          │   │  │  │
│  │  │  └────────────────┘    │ • Validate & update   │   │  │  │
│  │  │                        └───────────────────────┘   │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                            │                               │  │
│  │  ┌─────────────────────────┼────────────────────────────┐ │  │
│  │  │  PostgreSQL             │  Redis Event Bus           │ │  │
│  │  │  (Commerce Data)        │  (Event Distribution)      │ │  │
│  │  └─────────────────────────┴────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              │ HTTPS (REST/GraphQL)              │
│                              │ Management API + Webhooks         │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  CONTENTFUL (SaaS CMS)                                    │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Content Management                                  │  │  │
│  │  │                                                       │  │  │
│  │  │  ┌────────────────────────────────────────────────┐ │  │  │
│  │  │  │ Content Types (Schema)                         │ │  │  │
│  │  │  │ • Product, Variant, Collection, Category       │ │  │  │
│  │  │  │ • All mapped to Medusa structure               │ │  │  │
│  │  │  │ • Localized fields for multi-language          │ │  │  │
│  │  │  └────────────────────────────────────────────────┘ │  │  │
│  │  │                                                       │  │  │
│  │  │  ┌────────────────────────────────────────────────┐ │  │  │
│  │  │  │ Three API Layers:                              │ │  │  │
│  │  │  │ 1. Management API (CMA) - Write operations     │ │  │  │
│  │  │  │ 2. Delivery API (CDA) - Published content      │ │  │  │
│  │  │  │ 3. Preview API (CPA) - Draft content           │ │  │  │
│  │  │  └────────────────────────────────────────────────┘ │  │  │
│  │  │                                                       │  │  │
│  │  │  ┌────────────────────────────────────────────────┐ │  │  │
│  │  │  │ Webhooks (Outgoing)                            │ │  │  │
│  │  │  │ • Entry published → POST to Medusa             │ │  │  │
│  │  │  │ • Entry updated → POST to Medusa               │ │  │  │
│  │  │  │ • Entry deleted → POST to Medusa               │ │  │  │
│  │  │  └────────────────────────────────────────────────┘ │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                            │                               │  │
│  │  ┌─────────────────────────┼────────────────────────────┐ │  │
│  │  │  Contentful Cloud        │  Global CDN (Fastly)       │ │  │
│  │  │  (Data Storage)          │  (Content Delivery)        │ │  │
│  │  └─────────────────────────┴────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  STOREFRONT (Three Patterns)                              │  │
│  │                                                            │  │
│  │  Pattern 1: Direct Dual Fetch                             │  │
│  │    ├─→ Medusa API (commerce)                              │  │
│  │    └─→ Contentful Delivery API (content)                  │  │
│  │                                                            │  │
│  │  Pattern 2: Medusa Proxy                                  │  │
│  │    └─→ Medusa API (fetches from Contentful internally)    │  │
│  │                                                            │  │
│  │  Pattern 3: Server-Side Merge                             │  │
│  │    └─→ Server fetches both, merges, sends to client       │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Component Breakdown

#### 1. Medusa Plugin: `medusa-plugin-contentful`

**Location:** Installed in Medusa backend (`node_modules/medusa-plugin-contentful`)

**Main Files:**
```
medusa-plugin-contentful/
├── src/
│   ├── subscribers/
│   │   └── contentful.ts              # Event listeners
│   ├── services/
│   │   ├── contentful.ts               # Main service (sync logic)
│   │   └── index.ts
│   ├── api/
│   │   ├── routes/
│   │   │   └── webhooks.ts             # Webhook endpoint
│   │   └── middleware/
│   │       └── validate-webhook.ts     # Webhook validation
│   ├── types/
│   │   └── index.ts                    # TypeScript definitions
│   └── utils/
│       ├── transformations.ts          # Data format conversion
│       └── contentful-client.ts        # API client wrapper
```

**Key Responsibilities:**
- Listen to Medusa events via event bus
- Transform Medusa data format to Contentful format
- Authenticate with Contentful Management API
- POST/PUT/DELETE to Contentful
- Handle incoming webhooks from Contentful
- Retry failed requests
- Log sync operations

#### 2. Contentful Space

**Location:** Hosted on Contentful Cloud

**Structure:**
```
Contentful Space (your_space_id)
├── Environments
│   ├── master (production)
│   ├── staging (optional)
│   └── development (optional)
│
├── Content Types (Schema)
│   ├── Product
│   ├── Product Variant
│   ├── Product Collection
│   ├── Product Category
│   └── Region
│
├── Entries (Data)
│   ├── Products (with localized content)
│   ├── Variants
│   ├── Collections
│   └── Categories
│
├── Assets (Media Library)
│   ├── Images
│   ├── Videos
│   └── Documents
│
├── Locales
│   ├── en-US (default)
│   ├── es-ES
│   ├── de-DE
│   └── fr-FR (configurable)
│
└── Webhooks
    └── Medusa Webhook (→ /hooks/contentful)
```

**Key Responsibilities:**
- Store content entries with localization
- Provide Content Management API (write)
- Provide Content Delivery API (read published)
- Provide Preview API (read drafts)
- Send webhooks on entry changes
- Manage media assets with CDN
- Handle content workflows

## Data Flow Architecture

### Event-Driven Synchronization (Medusa → Contentful)

#### Step-by-Step: Product Creation Flow

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. User creates product in Medusa Admin                           │
│    POST /admin/products                                            │
│    { title, description, variants, ... }                           │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 2. Medusa ProductService.create() executes                        │
│    • Validates data                                                │
│    • Saves to PostgreSQL                                           │
│    • Returns product object                                        │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 3. Medusa emits event: "product.created"                          │
│    EventBusService.emit("product.created", { id: "prod_123" })    │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 4. Redis propagates event to all subscribers                      │
│    (Enables horizontal scaling of Medusa instances)               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 5. ContentfulSubscriber.onProductCreated() fires                  │
│    Listener registered in plugin initialization                   │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 6. ContentfulService.createProductInContentful() called           │
│    • Fetch full product data with relations:                      │
│      productService.retrieve(id, {                                 │
│        relations: ['options', 'variants', 'type',                  │
│                    'collection', 'categories', 'tags', 'images']   │
│      })                                                            │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 7. Transform data to Contentful format                            │
│    • Map Medusa fields to Contentful fields                       │
│    • Create entry structure with sys metadata                     │
│    • Prepare localized fields (default locale only initially)     │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 8. Authenticate with Contentful Management API                    │
│    • Use management access token from config                      │
│    • Headers: { Authorization: "Bearer <token>" }                 │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 9. HTTP POST to Contentful Management API                         │
│    POST https://api.contentful.com/spaces/{space_id}/             │
│         environments/{env}/entries                                 │
│    Headers: {                                                      │
│      Authorization: "Bearer <token>",                              │
│      Content-Type: "application/vnd.contentful.management.v1+json"│
│      X-Contentful-Content-Type: "product"                          │
│    }                                                               │
│    Body: {                                                         │
│      fields: {                                                     │
│        medusaId: { "en-US": "prod_123" },                          │
│        title: { "en-US": "Example Product" },                      │
│        ...                                                         │
│      }                                                             │
│    }                                                               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 10. Contentful stores and returns entry                           │
│     • Validates against content type schema                       │
│     • Creates entry in database                                   │
│     • Returns entry with Contentful ID                            │
│     • Entry is in draft state (not published yet)                 │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 11. Optional: Auto-publish entry                                  │
│     PUT https://api.contentful.com/spaces/{space_id}/             │
│         environments/{env}/entries/{entry_id}/published           │
│     (if auto_publish option enabled)                              │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 12. Success ✓                                                     │
│     Product now exists in both systems:                            │
│     • Medusa: Full commerce data                                  │
│     • Contentful: Product structure + empty localized fields      │
└──────────────────────────────────────────────────────────────────┘
```

### Webhook-Driven Sync (Contentful → Medusa)

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. Content editor updates entry in Contentful UI                  │
│    • Open product entry                                            │
│    • Edit title or description                                     │
│    • Click "Publish"                                               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 2. Contentful saves changes                                        │
│    • Updates entry in database                                     │
│    • Creates new version                                           │
│    • Changes entry state to "published"                            │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 3. Contentful webhook triggers                                     │
│    • Webhook configured: "Entry.publish" event                     │
│    • POST to: https://your-backend.com/hooks/contentful            │
│    • Payload includes:                                             │
│      - sys.id (Contentful entry ID)                                │
│      - sys.contentType                                             │
│      - fields.medusaId (reference to Medusa product)               │
│      - Updated field values                                        │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 4. Medusa webhook handler receives POST                            │
│    Route: POST /hooks/contentful                                   │
│    Handler: ContentfulWebhookHandler.handleWebhook()              │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 5. Validate webhook request                                        │
│    • Check webhook secret (if configured)                          │
│    • Verify payload structure                                      │
│    • Extract medusaId from fields                                  │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 6. Determine entity type and action                                │
│    • sys.contentType.sys.id = "product"                            │
│    • Extract changed fields                                        │
│    • Determine if update or delete                                 │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 7. Update Medusa product                                           │
│    await productService.update(                                    │
│      fields.medusaId["en-US"], // prod_123                         │
│      {                                                             │
│        title: fields.title["en-US"],  // Only sync specific fields│
│        description: fields.description["en-US"]                    │
│      }                                                             │
│    )                                                               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 8. Prevent sync loop                                               │
│    • Set flag to ignore next product.updated event                │
│    • Flag prevents re-syncing to Contentful                        │
│    • Flag expires after short period                               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 9. Success ✓                                                       │
│    • Changes synced back to Medusa                                 │
│    • Sync loop prevented                                           │
│    • Localized content stayed in Contentful                        │
└──────────────────────────────────────────────────────────────────┘
```

## Contentful API Architecture

### Three API Layers

#### 1. Content Management API (CMA)

**Purpose:** Write operations (create, update, delete entries)

**Used By:** Medusa plugin (for sync Medusa → Contentful)

**Authentication:** Management access token

**Base URL:** `https://api.contentful.com/spaces/{space_id}/environments/{env}/`

**Key Endpoints:**
```
POST   /entries                      # Create entry
GET    /entries/{entry_id}           # Read entry
PUT    /entries/{entry_id}           # Update entry
DELETE /entries/{entry_id}           # Delete entry
PUT    /entries/{entry_id}/published # Publish entry
DELETE /entries/{entry_id}/published # Unpublish entry
```

**Example Request:**
```http
POST /spaces/abc123/environments/master/entries HTTP/1.1
Host: api.contentful.com
Authorization: Bearer CFPAT-management-token
Content-Type: application/vnd.contentful.management.v1+json
X-Contentful-Content-Type: product

{
  "fields": {
    "medusaId": {
      "en-US": "prod_01HQ5K9ZFWXYZ123ABC"
    },
    "title": {
      "en-US": "Organic Cotton T-Shirt",
      "es-ES": "Camiseta de Algodón Orgánico"
    },
    "description": {
      "en-US": "A comfortable t-shirt...",
      "es-ES": "Una camiseta cómoda..."
    }
  }
}
```

#### 2. Content Delivery API (CDA)

**Purpose:** Read published content (for production storefronts)

**Used By:** Storefront (to fetch published content)

**Authentication:** Delivery access token (read-only)

**Base URL:** `https://cdn.contentful.com/spaces/{space_id}/environments/{env}/`

**Key Endpoints:**
```
GET /entries                    # Query entries
GET /entries/{entry_id}         # Read single entry
GET /entries?content_type=X     # Filter by content type
GET /entries?fields.X=Y         # Filter by field value
GET /entries?locale=es-ES       # Fetch in specific locale
GET /assets                     # Query assets
GET /assets/{asset_id}          # Read single asset
```

**Example Request:**
```http
GET /spaces/abc123/entries?content_type=product&fields.medusaId=prod_123&locale=es-ES HTTP/1.1
Host: cdn.contentful.com
Authorization: Bearer delivery-token
```

**Response:**
```json
{
  "sys": { "type": "Array" },
  "total": 1,
  "skip": 0,
  "limit": 100,
  "items": [
    {
      "sys": {
        "id": "contentful-entry-id",
        "type": "Entry",
        "contentType": { "sys": { "id": "product" } }
      },
      "fields": {
        "medusaId": "prod_123",
        "title": "Camiseta de Algodón Orgánico",
        "description": "Una camiseta cómoda..."
      }
    }
  ]
}
```

#### 3. Content Preview API (CPA)

**Purpose:** Read draft/unpublished content (for content editors)

**Used By:** Storefront in preview mode

**Authentication:** Preview access token

**Base URL:** `https://preview.contentful.com/spaces/{space_id}/environments/{env}/`

**Same endpoints as CDA, but returns:**
- Published entries (like CDA)
- Draft entries (not yet published)
- Updated entries (pending publish)

**Use Case:**
```javascript
// Content editor wants to preview changes before publishing
const previewClient = contentful.createClient({
  space: 'abc123',
  accessToken: 'preview-token',
  host: 'preview.contentful.com'  // Use preview API
});

const product = await previewClient.getEntries({
  content_type: 'product',
  'fields.medusaId': 'prod_123'
});
// Returns draft version with unpublished changes
```

## Authentication & Security

### API Token Types

Contentful provides 3 token types:

#### 1. Management Access Token (Write)

**Purpose:** Create, update, delete entries (used by Medusa)

**Permissions:** Full read/write access

**Creation:**
- Contentful Dashboard → Settings → API keys
- Content management tokens → Generate personal access token
- Copy token (shown once)

**Security:**
- Store in environment variable: `CONTENTFUL_ACCESS_TOKEN`
- Never commit to version control
- Never expose in frontend
- Rotate periodically

**Usage:**
```javascript
// In medusa-config.js
{
  resolve: "medusa-plugin-contentful",
  options: {
    access_token: process.env.CONTENTFUL_ACCESS_TOKEN  // Management token
  }
}
```

#### 2. Content Delivery API Token (Read Published)

**Purpose:** Read published content (used by storefront)

**Permissions:** Read-only, published entries only

**Creation:**
- Contentful Dashboard → Settings → API keys
- Add API key → Copy "Content Delivery API - access token"

**Security:**
- Can be exposed in frontend (read-only)
- Limited to published content
- Per-environment (different tokens for dev/prod)

**Usage:**
```javascript
// In storefront
const contentful = require('contentful');
const client = contentful.createClient({
  space: process.env.CONTENTFUL_SPACE_ID,
  accessToken: process.env.CONTENTFUL_DELIVERY_TOKEN  // Safe for frontend
});
```

#### 3. Content Preview API Token (Read Drafts)

**Purpose:** Preview unpublished changes (used by content editors)

**Permissions:** Read-only, includes drafts

**Creation:**
- Same location as delivery token
- Copy "Content Preview API - access token"

**Security:**
- Should not be exposed in production frontend
- Use in preview/staging environments only

### Webhook Security

**Option 1: No Authentication (Simple)**
```
Webhook URL: https://your-backend.com/hooks/contentful
Method: POST
No authentication
```

**Risks:** Anyone can POST to your webhook endpoint

**Option 2: Shared Secret (Recommended)**
```javascript
// Contentful webhook configuration
{
  headers: [
    {
      key: "X-Contentful-Webhook-Secret",
      value: "your-secret-here"
    }
  ]
}

// Medusa webhook handler
const webhookSecret = req.headers['x-contentful-webhook-secret'];
if (webhookSecret !== process.env.CONTENTFUL_WEBHOOK_SECRET) {
  return res.status(401).json({ error: "Unauthorized" });
}
```

**Option 3: HMAC Signature (Most Secure)**
```javascript
// Contentful includes X-Contentful-Webhook-Name and X-Contentful-Webhook-Signature
const crypto = require('crypto');
const signature = req.headers['x-contentful-webhook-signature'];
const body = JSON.stringify(req.body);
const hash = crypto
  .createHmac('sha256', process.env.CONTENTFUL_WEBHOOK_SECRET)
  .update(body)
  .digest('hex');

if (hash !== signature) {
  return res.status(401).json({ error: "Invalid signature" });
}
```

## Database Architecture

### Separate Databases

**Medusa PostgreSQL:**
```
Database: medusa_db
Tables:
  - product
  - product_variant
  - product_collection
  - product_category
  - region
  - order
  - customer
  - cart
  - ... (50+ commerce tables)
```

**Contentful Cloud:**
```
Managed by Contentful (not directly accessible)
Stores:
  - Entries (products, variants, etc.)
  - Assets (images, videos, files)
  - Locales and translations
  - Version history
  - Webhooks configuration
```

**Why Separate?**
- **Medusa**: Optimized for transactions (orders, cart, inventory)
- **Contentful**: Optimized for content delivery (CDN, localization)
- **Independence**: Each system scales independently
- **Isolation**: Failures don't cascade

**Trade-off:** Cannot use database JOINs across systems, must fetch separately.

### Data Relationships

**Medusa (Source):**
```
product
  ├─ id: "prod_123" (PK)
  ├─ title
  ├─ collection_id (FK → product_collection)
  ├─ type_id (FK → product_type)
  └─ ... (commerce fields)

product_variant
  ├─ id: "variant_456" (PK)
  ├─ product_id (FK → product)
  ├─ title
  └─ ... (variant fields)
```

**Contentful (Mirror + Content):**
```
Product Entry
  ├─ sys.id: "contentful-entry-id" (Contentful PK)
  ├─ fields.medusaId: "prod_123" (ref to Medusa)
  ├─ fields.title (localized):
  │   ├─ en-US: "Organic Cotton T-Shirt"
  │   ├─ es-ES: "Camiseta de Algodón Orgánico"
  │   └─ de-DE: "Bio-Baumwolle T-Shirt"
  ├─ fields.richDescription (localized, Contentful-only)
  ├─ fields.images (Asset references)
  └─ fields.productCollection (Entry reference)
```

**Key Pattern:**
- Contentful uses `medusaId` field to reference Medusa entities
- Relations in Contentful mirror Medusa relations
- Localized fields exist only in Contentful
- `sys.id` is Contentful's internal ID (not used by Medusa)

## Storefront Integration Patterns

### Pattern 1: Direct Dual Fetch (Recommended for Flexibility)

**Architecture:**
```
Storefront
  ├─→ Medusa API (commerce data)
  └─→ Contentful Delivery API (content data)
```

**Implementation:**
```typescript
// storefront/lib/data.ts

import { Medusa } from "@medusajs/medusa-js";
import { createClient } from "contentful";

const medusa = new Medusa({ baseUrl: MEDUSA_URL });
const contentful = createClient({
  space: CONTENTFUL_SPACE_ID,
  accessToken: CONTENTFUL_DELIVERY_TOKEN
});

export async function getProduct(handle: string, locale: string = 'en-US') {
  // Fetch commerce data from Medusa
  const { product } = await medusa.products.list({ handle });
  
  // Fetch content from Contentful
  const contentfulData = await contentful.getEntries({
    content_type: 'product',
    'fields.medusaId': product.id,
    locale: locale
  });
  
  // Merge
  return {
    ...product,
    richDescription: contentfulData.items[0]?.fields.richDescription,
    images: contentfulData.items[0]?.fields.images,
    seoTitle: contentfulData.items[0]?.fields.seoTitle,
    seoDescription: contentfulData.items[0]?.fields.seoDescription
  };
}
```

**Pros:**
- Full API flexibility (use all Contentful features)
- Parallel requests (Promise.all for performance)
- Better caching control (cache each API separately)
- Use GraphQL if needed (Contentful supports it)

**Cons:**
- Two API clients to maintain
- CORS configuration for both APIs
- Two sets of error handling
- Slightly more complex code

### Pattern 2: Medusa Proxy (Recommended for Simplicity)

**Architecture:**
```
Storefront → Medusa API → Contentful API
```

**Medusa Implementation:**
```typescript
// medusa-backend/src/api/routes/store/products/[id]/contentful.ts

export async function GET(req, res) {
  const { id } = req.params;
  const locale = req.query.locale || 'en-US';
  
  // Fetch from Contentful
  const contentfulService = req.scope.resolve("contentfulService");
  const content = await contentfulService.getProductContent(id, locale);
  
  res.json(content);
}
```

**Storefront Implementation:**
```typescript
// storefront/lib/data.ts

export async function getProduct(handle: string, locale: string = 'en-US') {
  // Single call to Medusa
  const response = await fetch(
    `${MEDUSA_URL}/store/products/${handle}?expand=contentful&locale=${locale}`
  );
  const { product } = await response.json();
  
  // Already merged by Medusa
  return product;
}
```

**Pros:**
- Single API client in storefront
- Centralized error handling
- Can add caching layer in Medusa
- Simpler frontend code
- Single CORS configuration

**Cons:**
- Additional hop (latency)
- Medusa must handle Contentful errors
- Less flexibility in querying
- Medusa becomes single point of failure

### Pattern 3: Server-Side Merge (Recommended for Performance & SEO)

**Architecture:**
```
Browser → Next.js Server → [Medusa API + Contentful API] → Merged Response
```

**Implementation:**
```typescript
// storefront/pages/products/[handle].tsx

export async function getServerSideProps({ params, locale }) {
  const { handle } = params;
  
  // Parallel fetch from both APIs (server-side)
  const [productRes, contentRes] = await Promise.all([
    fetch(`${MEDUSA_URL}/store/products?handle=${handle}`),
    fetch(`${CONTENTFUL_URL}/entries?content_type=product&fields.handle=${handle}&locale=${locale}`, {
      headers: { Authorization: `Bearer ${CONTENTFUL_DELIVERY_TOKEN}` }
    })
  ]);
  
  const { products } = await productRes.json();
  const { items } = await contentRes.json();
  
  const product = {
    ...products[0],
    ...items[0]?.fields
  };
  
  return {
    props: { product }
  };
}

export default function ProductPage({ product }) {
  // Product already merged, no client-side fetching
  return <ProductTemplate product={product} />;
}
```

**Pros:**
- Parallel fetching (fastest)
- No client-side loading (better UX)
- Better SEO (content in initial HTML)
- Can use server-only tokens
- Single round-trip from browser

**Cons:**
- Longer TTFB (server waits for both APIs)
- Server load (processing for each request)
- Not suitable for client-side navigation
- Need ISR or caching for performance

**Recommendation:**
- Use **Pattern 3** for product pages (SEO critical)
- Use **Pattern 1** for dynamic features (cart, search)
- Use **Pattern 2** for simplicity (smaller teams)

## Performance Considerations

### Bottlenecks

1. **Event Bus Latency (Medusa → Contentful)**
   - Redis pub/sub ~1-5ms
   - Subscriber processing ~10-50ms
   - Contentful API call ~100-300ms
   - **Total:** ~150-350ms per sync

2. **HTTP Requests**
   - Network round-trip to Contentful (varies by region)
   - API authentication overhead
   - JSON serialization/deserialization

3. **Contentful API Rate Limits**
   - Free tier: 10 req/s (requests per second)
   - Team tier: 98 req/s
   - Enterprise: Custom

### Optimizations

**1. Caching (Critical)**
```typescript
// Cache Contentful responses in Redis
const cached = await redis.get(`product:${id}:${locale}`);
if (cached) {
  return JSON.parse(cached);
}

const content = await contentful.getEntries({...});
await redis.set(`product:${id}:${locale}`, JSON.stringify(content), 'EX', 3600);  // 1 hour
```

**2. Parallel Fetching**
```typescript
// Don't do this (sequential)
const product = await medusa.products.retrieve(id);
const content = await contentful.getEntries({...});

// Do this (parallel)
const [product, content] = await Promise.all([
  medusa.products.retrieve(id),
  contentful.getEntries({...})
]);
```

**3. Contentful CDN**
```typescript
// Use CDN-backed Delivery API (not Management API)
const client = contentful.createClient({
  space: SPACE_ID,
  accessToken: DELIVERY_TOKEN,  // Uses cdn.contentful.com (fast)
  // NOT Management API (api.contentful.com - slower)
});
```

**4. GraphQL for Complex Queries**
```graphql
# Fetch only needed fields
query {
  productCollection(where: { medusaId: "prod_123" }, locale: "es-ES", limit: 1) {
    items {
      medusaId
      title
      richDescription
      imagesCollection {
        items {
          url
          title
        }
      }
    }
  }
}
```

**5. Selective Syncing**
```typescript
// Only sync if relevant fields changed
const updateFields = ['title', 'description', 'thumbnail'];
const found = data.fields?.find(f => updateFields.includes(f));
if (!found) {
  return;  // Skip sync (e.g., price change doesn't need content sync)
}
```

## Monitoring & Debugging

### Logging

**Medusa Logs:**
```bash
# Plugin initialization
info: Contentful plugin initialized
info: Space ID: abc123, Environment: master

# Sync operations
info: Syncing product prod_123 to Contentful
info: Successfully created product in Contentful (entry_id: xyz789)

# Errors
error: Failed to sync product prod_123: API rate limit exceeded
error: Contentful webhook validation failed: Invalid signature
```

**Contentful Logs:**
- View in Contentful Dashboard → Settings → Webhooks → Webhook activity log
- Shows webhook deliveries, HTTP status, response time
- Can replay failed webhooks

### Health Checks

**Contentful API:**
```bash
# Check API is accessible
curl -H "Authorization: Bearer ${TOKEN}" \
  https://api.contentful.com/spaces/${SPACE_ID}
# Returns space details if healthy
```

**Medusa Webhook Endpoint:**
```bash
# Test webhook endpoint is accessible
curl -X POST https://your-backend.com/hooks/contentful \
  -H "Content-Type: application/json" \
  -d '{"test": "payload"}'
# Should return 200 OK or validation error
```

### Debugging Tips

**1. Check Event Emission**
```typescript
// In Medusa, verify events are emitted
eventBusService.emit("product.created", { id: "prod_123" });
// Check Medusa logs for "Event emitted: product.created"
```

**2. Verify Subscriber Registration**
```typescript
// Check subscriber is loaded
// Look for log: "Contentful subscriber initialized"
```

**3. Test Contentful API Manually**
```bash
# Create test entry
curl -X POST https://api.contentful.com/spaces/${SPACE_ID}/environments/master/entries \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/vnd.contentful.management.v1+json" \
  -H "X-Contentful-Content-Type: product" \
  -d '{
    "fields": {
      "medusaId": {"en-US": "test-123"},
      "title": {"en-US": "Test Product"}
    }
  }'
```

**4. Check Webhook Delivery**
- Contentful Dashboard → Settings → Webhooks
- Click on webhook → View details
- See all deliveries, status codes, response times
- Replay failed webhooks

## What's Next?

Explore other documentation:
- **[Data Flow Scenarios](./03-data-flow.md)** - See specific sync examples
- **[Implementation Guide](./04-implementation.md)** - Set it up step-by-step
- **[Pros & Cons](./05-pros-cons.md)** - Evaluate trade-offs
- **[FAQ](./FAQ.md)** - Common questions

