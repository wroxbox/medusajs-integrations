# Sanity CMS Integration - Overview

## What is This Integration?

This integration connects **Medusa v2** (headless e-commerce platform) with **Sanity** (headless content management system) to create a powerful hybrid architecture where:

- **Medusa** manages commerce operations (products, pricing, inventory, orders, payments, shipping)
- **Sanity** manages content and localization (product specifications, multilingual content, product relationships)
- **Next.js Storefront** fetches data from **both** systems independently and merges them for display

## Why Use This Architecture?

### Traditional E-commerce Approach

In typical e-commerce setups, product content (descriptions, specifications) is stored directly in the e-commerce database alongside commerce data. This works but has limitations:

- Limited rich content capabilities
- No built-in localization
- Difficult content collaboration
- No version control or preview workflows
- Content changes require developer involvement

### Hybrid CMS + E-commerce Approach

This integration separates concerns:

```
Traditional (Single System):
┌─────────────────────────┐
│   E-COMMERCE SYSTEM     │
│  • Products & Pricing   │
│  • Inventory & Orders   │
│  • Descriptions         │  ← Everything in one place
│  • Content              │
│  • Localization         │
└─────────────────────────┘

Hybrid (Separated):
┌───────────────────┐       ┌────────────────────┐
│   MEDUSA          │       │   SANITY           │
│   (Commerce)      │       │   (Content)        │
│                   │       │                    │
│  • Products       │──────►│  • Specifications  │
│  • Variants       │ sync  │  • Localized Text  │
│  • Pricing        │       │  • Relationships   │
│  • Inventory      │       │  • Rich Content    │
│  • Orders         │       │                    │
└─────────┬─────────┘       └──────────┬─────────┘
          │                             │
          │        ┌──────────┐        │
          └───────►│ STOREFRONT│◄──────┘
                   │(Dual Fetch)│
                   └────────────┘
```

## Key Benefits

### ✅ For Content Teams

1. **Sanity Studio Interface**
   - Intuitive, modern content editor
   - Real-time collaborative editing
   - No code deployment needed for content updates
   - Visual content organization

2. **Localization Support**
   - Native multi-language content structure
   - Localized specifications per product
   - Language-specific product details
   - Easy to add new languages

3. **Structured Content**
   - Define custom schemas
   - Repeatable content blocks
   - Reference other documents
   - Validation and required fields

4. **Content Relationships**
   - Link products as addons/accessories
   - Create product bundles
   - Cross-sell recommendations
   - Related content management

### ✅ For Developers

1. **Separation of Concerns**
   - Commerce logic isolated in Medusa
   - Content management in Sanity
   - Clear boundaries and responsibilities
   - Independent scaling

2. **Powerful Query Language (GROQ)**
   - Fetch exactly what you need
   - Complex content relationships
   - Efficient data retrieval
   - Type-safe queries

3. **Real-time Updates**
   - Content changes reflect immediately
   - No need to restart Medusa
   - Live preview capabilities
   - Instant content delivery

4. **Flexible Content Modeling**
   - Easy to extend schemas
   - Custom field types
   - Complex nested structures
   - Validation at CMS level

### ✅ For Business

1. **Content Velocity**
   - Non-technical teams can update content
   - No developer bottleneck for copy changes
   - Faster time to market
   - Agile content management

2. **Global Reach**
   - Serve multiple languages from one product
   - Localized shopping experiences
   - Cultural adaptation
   - Regional content variations

3. **Scalability**
   - Sanity handles content delivery via CDN
   - Medusa focuses on commerce logic
   - Each system scales independently
   - Optimized for their specific roles

4. **Content Quality**
   - Preview before publish
   - Editorial workflows
   - Version history
   - Collaboration features

## What Gets Synced?

### Medusa → Sanity (Automatic, One-way)

When products are created or updated in Medusa, the integration automatically syncs:

**Product Core Data:**
```typescript
{
  _id: product.id,           // Medusa product ID as Sanity document ID
  _type: "product",          // Document type
  title: product.title,       // Product title
  specs: [                    // Array of localized specifications
    {
      _key: product.id,       // Unique key for array item
      _type: "spec",
      lang: "en",             // Language code
      title: product.title,   // Title in this language
      content: ""             // Empty initially - filled in Sanity
    }
  ]
}
```

**Triggered by:**
- `product.created` event
- `product.updated` event

**Sync method:**
- Upsert (creates if new, updates if exists)
- Only updates `title` field on updates
- Preserves existing Sanity content

### Sanity Exclusive (Manual Management)

These fields are ONLY managed in Sanity and never synced back to Medusa:

**Localized Specifications:**
```typescript
specs: [
  {
    lang: "en",
    title: "Premium Cotton T-Shirt",
    content: "Made from 100% organic cotton..."
  },
  {
    lang: "es",
    title: "Camiseta de Algodón Premium",
    content: "Hecha de algodón 100% orgánico..."
  },
  {
    lang: "fr",
    title: "T-Shirt en Coton Premium",
    content: "Fabriqué à partir de coton 100% bio..."
  }
]
```

**Product Addons/Cross-sells:**
```typescript
addons: {
  title: "Complete the Look",
  products: [
    { _ref: "prod_123..." },  // Reference to another Sanity product
    { _ref: "prod_456..." },
    { _ref: "prod_789..." }
  ]
}
```

### NOT Synced (Medusa Exclusive)

The following remain exclusively in Medusa and are never sent to Sanity:

| Data Type | Why Not Synced |
|-----------|----------------|
| Pricing | Dynamic, region-specific, commerce concern |
| Inventory levels | Real-time, transactional data |
| Variants | Commerce structure, pricing related |
| Product images | Medusa handles media |
| Collections | Catalog organization |
| Categories | Medusa taxonomy |
| Tags | Commerce metadata |
| Sales channels | Multi-channel management |
| Shipping profiles | Fulfillment configuration |
| Order data | Transactional data |
| Customer data | Private, sensitive information |

## Architecture Overview

### Infrastructure Components

```
┌─────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────┐        ┌────────────────────┐    │
│  │ PostgreSQL       │        │ Sanity Content Lake│    │
│  │ Medusa Database  │        │ (Sanity Cloud)     │    │
│  └────────┬─────────┘        └────────┬───────────┘    │
│           │                            │                │
│           │                            │                │
│  ┌────────▼─────────────┐    ┌────────▼──────────────┐ │
│  │  Medusa Backend      │    │  Sanity API           │ │
│  │  (Port 9000)         │───►│  (api.sanity.io)      │ │
│  │                      │HTTP│                       │ │
│  │  • REST API          │    │  • REST API           │ │
│  │  • Admin Dashboard   │    │  • GROQ Query Engine  │ │
│  │  • Event Bus         │    │  • Real-time Sync     │ │
│  │  • Sanity Module     │    │  • CDN Delivery       │ │
│  │  • Workflows         │    │                       │ │
│  └──────────────────────┘    └──────────┬────────────┘ │
│                                          │               │
│           ┌──────────────────────────────┘               │
│           │                                              │
│  ┌────────▼──────────────┐                              │
│  │  Next.js + Studio     │                              │
│  │  (Port 8000)          │                              │
│  │                       │                              │
│  │  • Storefront         │                              │
│  │  • Sanity Studio      │  /studio                     │
│  │  • Dual API Fetching  │                              │
│  │    - Medusa SDK       │                              │
│  │    - Sanity Client    │                              │
│  └───────────────────────┘                              │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Key Components

#### 1. Medusa Sanity Module

**Location**: `medusa/src/modules/sanity/`

**Purpose**: HTTP client for Sanity API

**Key Features**:
- Connects to Sanity using `@sanity/client`
- Provides methods for CRUD operations
- Transforms Medusa products to Sanity format
- Handles upsert logic

**Configuration**:
```typescript
{
  resolve: "./src/modules/sanity",
  options: {
    api_token: process.env.SANITY_API_TOKEN,      // Editor token
    project_id: process.env.SANITY_PROJECT_ID,    // Sanity project ID
    api_version: "2024-01-01",                     // API version
    dataset: "production",                          // Dataset name
    studio_url: "http://localhost:8000/studio",    // Studio URL
    type_map: {
      product: "product"                            // Medusa → Sanity type
    }
  }
}
```

#### 2. Event Subscriber

**Location**: `medusa/src/subscribers/sanity-product-sync.ts`

**Purpose**: Listen for product events and trigger sync

**Events Handled**:
- `product.created`
- `product.updated`

**Action**: Triggers `sanitySyncProductsWorkflow`

#### 3. Sync Workflow

**Location**: `medusa/src/workflows/sanity-sync-products/`

**Purpose**: Orchestrate product sync to Sanity

**Steps**:
1. Query product data from Medusa
2. Transform to Sanity document format
3. Upsert to Sanity via module
4. Handle errors and rollback if needed

#### 4. Link Definition

**Location**: `medusa/src/links/product-sanity.ts`

**Purpose**: Define relationship between Medusa products and Sanity documents

**Note**: Link is defined but NOT used for data fetching in storefront (unlike Payload integration). Storefront fetches from Sanity directly.

```typescript
defineLink(
  ProductModule.linkable.product,    // Medusa product
  {
    linkable: {
      serviceName: SANITY_MODULE,
      alias: "sanity_product"
    }
  },
  { readOnly: true }                  // One-way relationship
)
```

#### 5. Sanity Studio

**Location**: `storefront/studio` (mounted at `/studio` route)

**Purpose**: Web-based content management interface

**Features**:
- Visual document editor
- Real-time collaboration
- Custom schemas
- Content preview
- Validation

#### 6. Sanity Schema

**Location**: `storefront/src/sanity/schemaTypes/documents/product.ts`

**Purpose**: Define structure of product documents in Sanity

**Schema**:
```typescript
{
  name: "product",
  type: "document",
  fields: [
    { name: "title", type: "string" },
    {
      name: "specs",
      type: "array",
      of: [{
        type: "object",
        fields: [
          { name: "lang", type: "string" },     // Language code
          { name: "title", type: "string" },    // Localized title
          { name: "content", type: "text" }     // Localized content
        ]
      }]
    },
    {
      name: "addons",
      type: "object",
      fields: [
        { name: "title", type: "string" },
        {
          name: "products",
          type: "array",
          of: [{ type: "reference", to: [{ type: "product" }] }]
        }
      ]
    }
  ]
}
```

## Technology Stack

### Medusa Backend
- **Framework**: Medusa v2
- **Language**: TypeScript
- **Database**: PostgreSQL
- **Sanity Client**: `@sanity/client` v6+

### Sanity CMS
- **Version**: Latest
- **Dataset**: Production (customizable)
- **API**: REST + GROQ
- **Studio**: Web-based (embedded in Next.js)

### Storefront
- **Framework**: Next.js 14+ (App Router)
- **Language**: TypeScript
- **Styling**: Tailwind CSS
- **Data Fetching**: 
  - **Medusa SDK** for commerce data
  - **Sanity Client** (`next-sanity`) for content

## Data Fetching Pattern (Important!)

**Unlike Payload integration**, this integration uses **dual fetching**:

```typescript
// On product page
export default async function ProductPage({ params }) {
  // 1. Fetch commerce data from Medusa
  const product = await listProducts({
    countryCode: params.countryCode,
    queryParams: { handle: params.handle }
  }).then(({ response }) => response.products[0])

  // 2. Fetch content data DIRECTLY from Sanity
  const sanity = await client.getDocument(product.id)
  
  // 3. Pass both to template
  return <ProductTemplate product={product} sanity={sanity?.specs[0]} />
}
```

**Why Dual Fetch?**
- Sanity content may be large or complex
- GROQ allows precise content queries
- Avoids Medusa as middleman for content
- Leverages Sanity's CDN for content delivery

## Quick Start Summary

1. **Create Sanity Project**: Get project ID and API token
2. **Configure Medusa**: Add Sanity module with credentials
3. **Define Schema**: Configure Sanity content structure
4. **Setup Studio**: Embed Sanity Studio in Next.js
5. **Test Sync**: Create product in Medusa → Verify in Sanity Studio
6. **Add Content**: Enhance products with localized specs in Studio
7. **Fetch in Storefront**: Implement dual fetch pattern

## When to Use This Integration

### ✅ Perfect For:

- **Multi-language stores** requiring localized content
- **Content-rich products** with detailed specifications
- **International businesses** serving multiple regions
- **Content team collaboration** with real-time editing
- **Structured content** with relationships and references
- **Large content volumes** needing CDN delivery

### ❌ Avoid If:

- **Simple catalogs** with basic descriptions
- **Single-language stores** with minimal content
- **Budget constraints** prefer avoiding external services
- **Minimal content updates** don't justify CMS overhead
- **Want integrated fetching** (consider Payload instead)
- **Self-hosting preference** (Sanity is SaaS)

## Comparison to Other Approaches

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| **Medusa Only** | Simple, integrated | Limited content features | Basic stores |
| **Medusa + Sanity** | Localization, collaboration, CDN | Dual fetch, external dependency | Multi-language, content-rich |
| **Medusa + Payload** | Self-hosted, integrated fetch | No real-time collab | Full control, privacy-focused |
| **Medusa + Contentful** | Enterprise features | Expensive, complex | Large organizations |

## Next Steps

Ready to dive deeper? Continue with:

- **[Architecture Deep Dive](./02-architecture.md)** - Technical implementation details
- **[Data Flow](./03-data-flow.md)** - See how data moves through the system
- **[Implementation Guide](./04-implementation.md)** - Build it yourself step-by-step
- **[Pros & Cons](./05-pros-cons.md)** - Make the right decision for your project

## Questions?

Before proceeding, consider:

1. **Do you need multi-language support?** → Sanity excels here
2. **Is real-time collaboration important?** → Sanity provides this
3. **Can you accept dual API calls?** → Sanity requires this
4. **Do you want self-hosting?** → Consider Payload instead
5. **Is content versioning needed?** → Sanity includes this

If still unsure, see **[FAQ](./FAQ.md)** or **[Pros & Cons](./05-pros-cons.md)** for detailed comparisons.

