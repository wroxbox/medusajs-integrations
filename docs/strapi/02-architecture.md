# Architecture Deep Dive

This document provides technical details on how the Medusa-Strapi integration works under the hood.

## System Components

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      INTEGRATION ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  MEDUSA BACKEND (Master)                 Port 9000        │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  medusa-plugin-strapi-ts                           │  │  │
│  │  │                                                     │  │  │
│  │  │  ┌───────────────┐    ┌────────────────────────┐  │  │  │
│  │  │  │ Subscribers   │────│ UpdateStrapiService    │  │  │  │
│  │  │  │ (Event        │    │ • Transform data       │  │  │  │
│  │  │  │  Listeners)   │    │ • HTTP client          │  │  │  │
│  │  │  └───────────────┘    │ • JWT authentication   │  │  │  │
│  │  │                       │ • Retry logic          │  │  │  │
│  │  │                       └────────────────────────┘  │  │  │
│  │  │                                                     │  │  │
│  │  │  ┌───────────────┐    ┌────────────────────────┐  │  │  │
│  │  │  │ API Routes    │────│ UpdateMedusaService    │  │  │  │
│  │  │  │ /strapi/*     │    │ • Receive from Strapi  │  │  │  │
│  │  │  └───────────────┘    │ • Validate & update    │  │  │  │
│  │  │                       └────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │                           │                             │  │
│  │  ┌────────────────────────┼──────────────────────────┐ │  │
│  │  │  PostgreSQL            │  Redis Event Bus         │ │  │
│  │  │  (Commerce Data)       │  (Event Distribution)    │ │  │
│  │  └────────────────────────┴──────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              │ HTTP/HTTPS                       │
│                              │ REST API                         │
│                              │ JWT Auth                         │
│                              ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  STRAPI SERVER (Slave)                   Port 1337       │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  strapi-plugin-medusajs                            │  │  │
│  │  │                                                     │  │  │
│  │  │  ┌───────────────┐    ┌────────────────────────┐  │  │  │
│  │  │  │ Bootstrap     │────│ User Management        │  │  │  │
│  │  │  │ (Startup      │    │ • Create service acct  │  │  │  │
│  │  │  │  Hooks)       │    │ • Verify JWT secrets   │  │  │  │
│  │  │  └───────────────┘    │ • Sync endpoint        │  │  │  │
│  │  │                       └────────────────────────┘  │  │  │
│  │  │                                                     │  │  │
│  │  │  ┌───────────────────────────────────────────────┐ │  │  │
│  │  │  │ Content-Type Definitions (Schema)             │ │  │  │
│  │  │  │ • Product, Variant, Collection, Category      │ │  │  │
│  │  │  │ • Region, Currency, Country                   │ │  │  │
│  │  │  │ • All mapped to Medusa structure              │ │  │  │
│  │  │  └───────────────────────────────────────────────┘ │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │                           │                             │  │
│  │  ┌────────────────────────┼──────────────────────────┐ │  │
│  │  │  PostgreSQL (Separate) │  Admin Panel UI          │ │  │
│  │  │  (Content Data)        │  /admin                  │ │  │
│  │  └────────────────────────┴──────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  STOREFRONT                                              │  │
│  │                                                          │  │
│  │  Fetches Data:                                           │  │
│  │  1. Commerce from Medusa API                             │  │
│  │  2. Content from Strapi API (or Medusa proxy)            │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Component Breakdown

#### 1. Medusa Plugin: `medusa-plugin-strapi-ts`

**Location:** Installed in Medusa backend (`node_modules/medusa-plugin-strapi-ts`)

**Main Files:**
```
medusa-plugin-strapi-ts/
├── src/
│   ├── subscribers/
│   │   └── strapi.ts              # Event listeners
│   ├── services/
│   │   ├── update-strapi.ts       # Sync Medusa → Strapi
│   │   └── update-medusa.ts       # Sync Strapi → Medusa
│   ├── api/
│   │   ├── routes/               # API endpoints
│   │   ├── controllers/          # Request handlers
│   │   └── middleware/           # Auth & validation
│   ├── types/
│   │   └── globals.ts            # TypeScript definitions
│   └── utils/
│       ├── transformations.ts    # Data format conversion
│       └── redis-key-manager.ts  # Ignore flags
```

**Key Responsibilities:**
- Listen to Medusa events via event bus
- Transform Medusa data format to Strapi format
- Authenticate with Strapi (JWT)
- POST/PUT/DELETE to Strapi REST API
- Provide proxy endpoints (`/strapi/content/*`)
- Handle authentication and token refresh
- Retry failed requests with exponential backoff

#### 2. Strapi Plugin: `strapi-plugin-medusajs`

**Location:** Installed in Strapi (`node_modules/strapi-plugin-medusajs`)

**Main Files:**
```
strapi-plugin-medusajs/
├── server/
│   ├── bootstrap.ts              # Startup initialization
│   ├── controllers/
│   │   └── setup.ts              # Sync endpoints
│   ├── services/
│   │   ├── setup.ts              # User & sync management
│   │   └── index.ts
│   ├── routes/
│   │   └── index.ts              # Route definitions
│   └── config/
│       └── index.ts              # Plugin configuration
└── admin/
    └── src/                      # Admin UI extensions
```

**Key Responsibilities:**
- Create super admin on first boot
- Register Medusa service account user
- Verify JWT secret matches Medusa
- Provide sync endpoint for bulk operations
- Manage content types for Medusa entities

#### 3. Strapi Server: `medusa-strapi`

**Location:** Standalone Strapi application

**Key Files:**
```
medusa-strapi/
├── config/
│   ├── database.js               # PostgreSQL connection
│   ├── server.js                 # Port, host, JWT settings
│   ├── plugins.js                # Enable medusajs plugin
│   └── middlewares.js            # CORS, security
├── src/
│   ├── api/                      # Content types (auto-generated)
│   │   ├── product/
│   │   │   ├── content-types/
│   │   │   │   └── product/
│   │   │   │       └── schema.json
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   └── routes/
│   │   ├── product-variant/
│   │   ├── product-collection/
│   │   ├── product-category/
│   │   ├── region/
│   │   └── ... (20+ content types)
│   ├── components/               # Reusable content components
│   └── policies/                 # Access control
└── data/
    └── uploads/                  # Media library
```

**Key Responsibilities:**
- Store content data in PostgreSQL
- Provide REST API for CRUD operations
- Admin panel for content management
- Media library for asset management
- Authentication and authorization

## Data Flow Architecture

### Event-Driven Synchronization

#### Step-by-Step: Product Creation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. User creates product in Medusa Admin                          │
│    POST /admin/products                                           │
│    { title, description, variants, ... }                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. Medusa ProductService.create() executes                        │
│    • Validates data                                               │
│    • Saves to PostgreSQL                                          │
│    • Returns product object                                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. Medusa emits event: "product.created"                         │
│    EventBusService.emit("product.created", { id: "prod_123" })   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. Redis propagates event to all subscribers                     │
│    (Enables horizontal scaling of Medusa instances)              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. StrapiSubscriber.onProductCreated() fires                     │
│    File: subscribers/strapi.ts                                    │
│    Listener registered: eventBus_.subscribe(ProductService.      │
│                          Events.CREATED, handler)                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 6. UpdateStrapiService.createProductInStrapi() called            │
│    • Fetch full product data with relations:                     │
│      productService.retrieve(id, {                                │
│        relations: ['options', 'variants', 'type',                 │
│                    'collection', 'categories', 'tags', 'images']  │
│      })                                                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 7. Transform data to Strapi format                               │
│    • Rename fields:                                               │
│      - product.type → product.product_type                        │
│      - product.collection → product.product_collection            │
│      - product.tags → product.product_tags                        │
│    • Convert IDs to medusa_id references                          │
│    • Remove Medusa-specific fields (created_at, updated_at)       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 8. Authenticate with Strapi                                      │
│    • Retrieve cached JWT token (if valid)                        │
│    • Or login: POST /api/auth/local                              │
│      { identifier: "medusa@service.local",                        │
│        password: "..." }                                          │
│    • Cache token with timestamp                                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 9. HTTP POST to Strapi API                                       │
│    POST https://strapi:1337/api/products                          │
│    Headers: { Authorization: "Bearer <jwt>" }                     │
│    Body: {                                                        │
│      data: {                                                      │
│        medusa_id: "prod_123",                                     │
│        title: "Example Product",                                  │
│        product_variants: [...],                                   │
│        product_collection: { medusa_id: "pcol_456" },             │
│        ...                                                        │
│      }                                                            │
│    }                                                              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 10. Strapi receives and processes                                │
│     • Validates against content-type schema                      │
│     • Resolves relations (creates if don't exist)                │
│     • Saves to Strapi PostgreSQL                                 │
│     • Returns created document with Strapi ID                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 11. Plugin stores mapping (optional, in memory)                  │
│     medusaId → strapiId mapping cached for future updates        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 12. Success ✓                                                    │
│     Product now exists in both systems:                           │
│     • Medusa: Full commerce data                                 │
│     • Strapi: Product structure + empty content fields           │
└─────────────────────────────────────────────────────────────────┘
```

### Reverse Sync: Strapi → Medusa (Limited)

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Content manager updates product title in Strapi Admin         │
│    PUT /api/products/:id                                          │
│    { data: { title: "New Title" } }                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. Strapi saves to database                                      │
│    Updates PostgreSQL record                                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. Strapi plugin could trigger webhook (not implemented)         │
│    Would need custom implementation                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. Currently: Manual API call to Medusa required                 │
│    POST /admin/strapi/update-product                              │
│    { strapiId: 123, medusaId: "prod_123", title: "New Title" }   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. UpdateMedusaService.sendStrapiProductToMedusa()               │
│    • Checks "ignore" flag (prevents sync loop)                   │
│    • Validates allowed fields                                    │
│    • Updates only whitelisted fields (title, subtitle)           │
│    • Sets "ignore" flag for Strapi sync                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 6. Medusa ProductService.update() executes                       │
│    Updates PostgreSQL with new title                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 7. Success ✓ (with safeguards)                                  │
│    • Title synced back to Medusa                                 │
│    • Ignore flag prevents re-sync to Strapi (loop prevention)    │
│    • Rich content stayed in Strapi only                          │
└─────────────────────────────────────────────────────────────────┘
```

## Authentication & Security

### JWT Authentication Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ INITIAL SETUP (First Time)                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ 1. Strapi Starts                                                 │
│    └──► strapi-plugin-medusajs/bootstrap.ts runs                 │
│                                                                   │
│ 2. Create Super Admin (if doesn't exist)                         │
│    POST /admin/register-admin                                     │
│    {                                                              │
│      email: env.SUPERUSER_EMAIL,                                  │
│      password: env.SUPERUSER_PASSWORD,                            │
│      username: env.SUPERUSER_USERNAME                             │
│    }                                                              │
│                                                                   │
│ 3. Login as Super Admin                                          │
│    POST /admin/login                                              │
│    Returns: { token: "...", user: {...} }                        │
│                                                                   │
│ 4. Create Service Account User                                   │
│    POST /strapi-plugin-medusajs/create-medusa-user                │
│    Headers: { Authorization: "Bearer <admin_token>" }             │
│    {                                                              │
│      email: env.STRAPI_MEDUSA_EMAIL,                              │
│      password: env.STRAPI_MEDUSA_PASSWORD,                        │
│      username: env.STRAPI_MEDUSA_USER,                            │
│      confirmed: true,                                             │
│      blocked: false                                               │
│    }                                                              │
│                                                                   │
│ 5. Medusa Plugin Starts                                          │
│    └──► medusa-plugin-strapi-ts loads                             │
│    └──► Checks Strapi health: GET /_health                       │
│    └──► Calls startInterface() if auto_start=true                │
│                                                                   │
│ 6. Login as Service Account                                      │
│    POST /api/auth/local                                           │
│    {                                                              │
│      identifier: "medusa@service.local",                          │
│      password: "shared_secret"                                    │
│    }                                                              │
│    Returns: { jwt: "...", user: {...} }                          │
│                                                                   │
│ 7. Cache JWT Token                                               │
│    Store in memory: userTokens[email] = {                         │
│      token: "...",                                                │
│      time: Date.now(),                                            │
│      user: {...}                                                  │
│    }                                                              │
│                                                                   │
│ 8. Optional: Trigger Initial Sync                                │
│    POST /strapi-plugin-medusajs/synchronise-medusa-tables        │
│    Syncs all existing Medusa data to Strapi                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ ONGOING OPERATIONS (Every Request)                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ 1. Plugin needs to call Strapi API                               │
│    └──► Check if token cached and valid                          │
│                                                                   │
│ 2. Token Validation                                              │
│    if (cachedToken && (now - cachedTime) < 180000) {             │
│      // Token still valid (< 3 min old)                          │
│      return cachedToken                                           │
│    }                                                              │
│                                                                   │
│ 3. Token Refresh (if expired)                                    │
│    POST /api/auth/local                                           │
│    { identifier, password }                                       │
│    Cache new token                                                │
│                                                                   │
│ 4. Make Authenticated Request                                    │
│    POST/PUT/DELETE /api/:contentType/:id?                         │
│    Headers: {                                                     │
│      Authorization: "Bearer <jwt>",                               │
│      Content-Type: "application/json"                             │
│    }                                                              │
│                                                                   │
│ 5. Error Handling                                                │
│    if (response.status === 401) {                                │
│      // Token expired, retry with fresh token                    │
│      throw LoginTokenExpiredError                                 │
│      // Caught by retry logic                                    │
│    }                                                              │
│    if (response.status === 429) {                                │
│      // Rate limited, exponential backoff                        │
│      wait(retryDelay)                                             │
│      retry()                                                      │
│    }                                                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Shared Secret Configuration

**Purpose:** Both systems must share the same JWT secret to verify tokens.

**Medusa Configuration:**
```javascript
// medusa-config.js
module.exports = {
  projectConfig: {
    jwt_secret: process.env.JWT_SECRET  // e.g., "supersecret123"
  },
  plugins: [
    {
      resolve: "medusa-plugin-strapi-ts",
      options: {
        strapi_secret: process.env.STRAPI_SECRET  // MUST MATCH Strapi
      }
    }
  ]
}
```

**Strapi Configuration:**
```javascript
// .env
MEDUSA_STRAPI_SECRET=supersecret123  // MUST MATCH Medusa JWT_SECRET
JWT_SECRET=supersecret123
```

**Security Best Practices:**
1. Use strong, random secrets (32+ characters)
2. Store in environment variables (never in code)
3. Rotate secrets periodically
4. Use different secrets per environment (dev/staging/prod)
5. Never commit secrets to version control

### Access Control

**Strapi Roles:**
- **Super Admin** - Full access to Strapi admin panel and API
- **Editor** - Can create/edit content
- **Author** - Can create content (limited)
- **Service Account** - Medusa plugin user (programmatic access)

**Permissions:**
```javascript
// Service account needs:
- create: products, variants, collections, categories, regions
- update: products, variants, collections, categories, regions
- delete: products, variants
- find: all content types
- findOne: all content types
```

**Strapi Public API:**
- By default, API requires authentication
- Can expose specific endpoints publicly if needed
- Use find/findOne permissions to control access

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

**Strapi PostgreSQL:**
```
Database: strapi_db
Tables:
  - products (custom content type)
  - product_variants (custom content type)
  - product_collections (custom content type)
  - product_categories (custom content type)
  - regions (custom content type)
  - upload_files (media library)
  - upload_folders
  - admin_users
  - admin_permissions
  - ... (Strapi core tables)
```

**Why Separate?**
- **Independence:** Each system can scale independently
- **Isolation:** Failures don't cascade
- **Clarity:** Clear ownership of data
- **Backup:** Separate backup strategies

**Trade-off:** Cannot use database JOINs across systems, must fetch separately.

### Data Relationships

**Medusa (Source):**
```
product
  ├─ product_id (PK)
  ├─ title
  ├─ collection_id (FK → product_collection)
  ├─ type_id (FK → product_type)
  └─ ... (commerce fields)

product_variant
  ├─ variant_id (PK)
  ├─ product_id (FK → product)
  ├─ title
  └─ ... (variant fields)
```

**Strapi (Mirror + Content):**
```
products
  ├─ id (Strapi PK)
  ├─ medusa_id (ref to Medusa product_id)
  ├─ title
  ├─ rich_description (rich text, CMS-only)
  ├─ product_collection (relation via medusa_id)
  ├─ product_type (relation via medusa_id)
  └─ ... (content fields)

product_variants
  ├─ id (Strapi PK)
  ├─ medusa_id (ref to Medusa variant_id)
  ├─ product (relation to products via medusa_id)
  ├─ title
  └─ ... (variant fields)
```

**Key Pattern:**
- Strapi uses `medusa_id` field to reference Medusa entities
- Relations in Strapi mirror Medusa relations
- Additional CMS-only fields don't exist in Medusa

## Content Type Schemas

### Example: Product Content Type

```json
{
  "kind": "collectionType",
  "collectionName": "products",
  "info": {
    "singularName": "product",
    "pluralName": "products",
    "displayName": "Product"
  },
  "options": {
    "draftAndPublish": false
  },
  "attributes": {
    "medusa_id": {
      "type": "string",
      "required": true,
      "unique": true
    },
    "title": {
      "type": "string"
    },
    "subtitle": {
      "type": "string"
    },
    "description": {
      "type": "text"
    },
    "handle": {
      "type": "string",
      "unique": true
    },
    "thumbnail": {
      "type": "string"
    },
    "weight": {
      "type": "decimal"
    },
    "length": {
      "type": "decimal"
    },
    "height": {
      "type": "decimal"
    },
    "width": {
      "type": "decimal"
    },
    "metadata": {
      "type": "json"
    },
    "product_variants": {
      "type": "relation",
      "relation": "oneToMany",
      "target": "api::product-variant.product-variant",
      "mappedBy": "product"
    },
    "product_collection": {
      "type": "relation",
      "relation": "manyToOne",
      "target": "api::product-collection.product-collection",
      "inversedBy": "products"
    },
    "product_categories": {
      "type": "relation",
      "relation": "manyToMany",
      "target": "api::product-category.product-category",
      "inversedBy": "products"
    },
    "product_type": {
      "type": "relation",
      "relation": "manyToOne",
      "target": "api::product-type.product-type",
      "inversedBy": "products"
    },
    "product_tags": {
      "type": "relation",
      "relation": "manyToMany",
      "target": "api::product-tag.product-tag",
      "inversedBy": "products"
    },
    "product_options": {
      "type": "relation",
      "relation": "oneToMany",
      "target": "api::product-option.product-option",
      "mappedBy": "product"
    }
  }
}
```

### All Content Types

The integration provides these pre-built content types:

| Content Type | Purpose | Relations |
|--------------|---------|-----------|
| `product` | Main product entity | variants, collection, categories, type, tags, options |
| `product-variant` | Product variations | product, prices, option values |
| `product-collection` | Product groupings | products |
| `product-category` | Product taxonomy | products, parent category |
| `product-type` | Product classification | products |
| `product-tag` | Product tags | products |
| `product-option` | Variant options (size, color) | product, option values |
| `product-option-value` | Option values (S, M, L) | option, variants |
| `product-metafield` | Custom metadata | (key-value pairs) |
| `region` | Geographic regions | currency, countries, payment/fulfillment providers |
| `currency` | Currency definitions | regions |
| `country` | Country definitions | regions |
| `money-amount` | Prices | variant, region, currency |
| `payment-provider` | Payment methods | regions |
| `fulfillment-provider` | Shipping methods | regions |
| `shipping-option` | Shipping options | region, requirements, profile |
| `shipping-profile` | Shipping profiles | shipping options |
| `store` | Store configuration | currencies, payment/fulfillment providers |
| `image` | Product images | (media library integration) |

## Retry Logic & Error Handling

### Axios Retry Strategy

```typescript
// Automatic retry on HTTP 429 (rate limiting)
axiosRetry(axios, {
  retries: 100,
  retryDelay: (retryCount, error) => {
    // Check for X-Retry-After header
    const retryAfter = error.response.headers['x-retry-after'];
    const rateLimitReset = error.response.headers['x-ratelimit-reset'];
    
    if (retryAfter) {
      return parseInt(retryAfter) * 1000;  // Convert to ms
    }
    
    if (rateLimitReset) {
      const currentTime = Date.now() / 1000;
      const waitTime = (parseInt(rateLimitReset) - currentTime + 2) * 1000;
      return waitTime;
    }
    
    // Default: 400 seconds
    return 400_000;
  },
  retryCondition: (error) => {
    return error.response.status === 429;
  }
});
```

### Sync Loop Prevention

**Problem:** Updates could ping-pong between systems infinitely.

**Solution:** Redis ignore flags with TTL.

```typescript
// Before syncing to Strapi, check ignore flag
const ignore = await shouldIgnore_(productId, 'strapi');
if (ignore) {
  return;  // Skip sync
}

// After successful sync, set ignore flag
await addIgnore_(productId, 'strapi', 3);  // 3 second TTL

// Strapi → Medusa sync does the opposite
await addIgnore_(productId, 'medusa', 3);
```

**Redis Keys:**
```
"prod_123_ignore_strapi"  → "1" (expires in 3s)
"prod_123_ignore_medusa"  → "1" (expires in 3s)
```

## Storefront Integration

### Option 1: Dual Fetch (Direct)

```typescript
// storefront/lib/data.ts

// Fetch commerce data from Medusa
const product = await fetch(`${MEDUSA_URL}/store/products/${handle}`)
  .then(res => res.json());

// Fetch content from Strapi
const content = await fetch(
  `${STRAPI_URL}/api/products?filters[medusa_id][$eq]=${product.id}`
).then(res => res.json());

// Merge
const enrichedProduct = {
  ...product,
  richDescription: content.data[0]?.rich_description,
  images: content.data[0]?.images,
  // ... other content fields
};
```

**Pros:**
- Direct access to both systems
- Can leverage Strapi's query capabilities (filters, populate)
- Better for complex queries

**Cons:**
- Two network requests (higher latency)
- Need to manage two API clients
- CORS configuration for both

### Option 2: Medusa Proxy

```typescript
// storefront/lib/data.ts

// Fetch via Medusa proxy endpoint
const product = await fetch(
  `${MEDUSA_URL}/strapi/content/products/${productId}`
).then(res => res.json());

// Already merged by Medusa backend
```

**Medusa Endpoint:**
```typescript
// medusa-plugin-strapi-ts/src/api/routes/index.ts
router.get(
  "/strapi/content/:contentType/:medusaId",
  async (req, res) => {
    const { contentType, medusaId } = req.params;
    
    // Fetch from Strapi
    const strapiData = await updateStrapiService.getEntitiesFromStrapi({
      strapiEntityType: contentType,
      id: medusaId,
      authInterface: defaultAuthInterface
    });
    
    res.json(strapiData);
  }
);
```

**Pros:**
- Single network request from storefront
- Centralized error handling
- Can add caching layer in Medusa
- Simpler frontend code

**Cons:**
- Additional hop (storefront → Medusa → Strapi)
- Slightly higher latency
- Less flexible querying

## Performance Considerations

### Bottlenecks

1. **Event Bus Latency**
   - Redis pub/sub ~1-5ms
   - Subscriber processing ~10-50ms
   - Strapi API call ~50-200ms
   - **Total:** ~100-250ms per sync

2. **HTTP Requests**
   - Network round-trip to Strapi
   - Authentication (JWT validation)
   - Database writes

3. **Database Operations**
   - Strapi relation resolution
   - Index lookups on `medusa_id`

### Optimizations

**1. Token Caching**
```typescript
// Don't fetch new token every request
if (cachedToken && (now - tokenTime) < 180_000) {
  return cachedToken;  // Use cached
}
```

**2. Batch Operations**
```typescript
// Instead of syncing variants one-by-one
for (const variant of variants) {
  await syncVariant(variant);  // ❌ Slow
}

// Batch sync
await Promise.all(
  variants.map(variant => syncVariant(variant))  // ✅ Parallel
);
```

**3. Selective Syncing**
```typescript
// Only sync if relevant fields changed
const updateFields = ['title', 'description', 'thumbnail'];
const found = data.fields?.find(f => updateFields.includes(f));
if (!found) {
  return;  // Skip sync
}
```

**4. Database Indexing**
```sql
-- Strapi database
CREATE INDEX idx_products_medusa_id ON products(medusa_id);
CREATE INDEX idx_variants_medusa_id ON product_variants(medusa_id);
```

## Monitoring & Debugging

### Logging

**Medusa Logs:**
```bash
# Plugin initialization
info: Checking Strapi Health
info: Strapi is healthy
info: Strapi Subscriber Initialized

# Sync operations
info: creating product in strapi - {...}
info: Successfully updated collection col_123 in Strapi

# Errors
error: unable to create collection col_123 Connection timeout
```

**Strapi Logs:**
```bash
# Startup
[2024-01-15 10:30:15] info: Strapi plugin medusajs initialized
[2024-01-15 10:30:16] info: Super admin user created
[2024-01-15 10:30:17] info: Medusa service account registered

# API requests
[2024-01-15 10:35:20] info: POST /api/products - 201 Created
```

### Health Checks

**Strapi Health:**
```bash
curl http://localhost:1337/_health
# Returns 204 No Content if healthy
```

**Medusa Health:**
```bash
curl http://localhost:9000/health
# Returns {"status": "ok"}
```

**Integration Health:**
```bash
# Check if Medusa can reach Strapi
curl http://localhost:9000/admin/strapi/health
# Custom endpoint to verify connection
```

## What's Next?

Explore other documentation:
- **[Data Flow Scenarios](./03-data-flow.md)** - See specific sync examples
- **[Implementation Guide](./04-implementation.md)** - Set it up step-by-step
- **[Pros & Cons](./05-pros-cons.md)** - Evaluate trade-offs
- **[FAQ](./FAQ.md)** - Common questions

