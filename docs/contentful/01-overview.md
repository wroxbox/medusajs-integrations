# Overview: Contentful Integration with Medusa

## What is the Contentful Integration?

The Contentful integration is a **two-way sync architecture** that connects Medusa v1 (your e-commerce backend) with Contentful (an enterprise headless CMS). This allows you to:

- **Manage commerce in Medusa**: Products, variants, pricing, inventory, orders
- **Manage content in Contentful**: Localized descriptions, rich media, marketing content, multi-language support
- **Automatically sync data**: Product structure flows between Medusa and Contentful
- **Empower content teams**: Non-technical users can manage localized, rich content without touching code

### The Core Concept

```
Medusa (Commerce)  ◄────sync────►  Contentful (Content)
     ↓                                    ↓
     └──────────── Storefront ────────────┘
            Fetches from both
```

**Medusa owns commerce** - Products, pricing, inventory, orders, customers, and all transactional data.

**Contentful owns content** - Localized descriptions, rich media assets, marketing copy, editorial content, and multi-language translations.

**Two-way sync** - Changes in Medusa automatically flow to Contentful. Changes in Contentful can flow back to Medusa via webhooks.

**Storefront reads from both** - It fetches commerce data (prices, inventory, cart) from Medusa and content data (descriptions, images, localized copy) from Contentful.

## Why Use Contentful with Medusa?

### The Problem

Standard Medusa installations store basic product information:
- Product title
- Simple description (text field)
- Handle and metadata
- Thumbnail URL

But for **enterprise e-commerce**, you often need:
- Multi-language product content (English, Spanish, German, etc.)
- Rich media galleries with optimized delivery
- Editorial workflows with approval processes
- Scheduled content publishing
- Content versioning and rollback
- A/B testing of product descriptions
- Preview modes for content teams
- Global CDN for fast content delivery

### The Solution

Contentful provides an enterprise CMS layer for your commerce platform:

| Without Contentful | With Contentful |
|-------------------|-----------------|
| Basic text description | Rich text with markdown, embeds, formatting |
| Single language | Multi-language content management |
| Manual media optimization | Automatic image optimization and CDN delivery |
| All content in code/database | Visual admin panel for content teams |
| Developers manage all content | Content managers work independently |
| No content workflow | Advanced workflows (draft/review/publish) |
| Limited content structure | Flexible content modeling |
| No content preview | Preview mode before publishing |
| Manual content versioning | Automatic version history |

## What Gets Synced (and What Doesn't)

### Automatic Sync: Medusa → Contentful ✅

When you create or update entities in Medusa, they **automatically sync to Contentful**:

**Products:**
- Product ID (stored as reference in Contentful)
- Title, subtitle, handle
- Description (base content)
- Thumbnail URL
- Dimensions (height, width, length, weight)
- Material, origin country
- Status (published/draft)
- Metadata

**Variants:**
- Variant ID, title, SKU
- Options (size, color, etc.)
- Barcode, EAN, UPC
- Inventory info
- Material, origin country
- Metadata

**Collections:**
- Collection ID, title, handle
- Product relationships
- Metadata

**Categories:**
- Category ID, name, handle
- Hierarchy and relationships
- Description

**Regions:**
- Region ID, name
- Currency code
- Tax settings
- Payment providers
- Fulfillment providers
- Countries

### Two-Way Sync: Contentful ↔ Medusa ⚠️

**How it works:**
- Medusa → Contentful: **Automatic** (via event subscribers)
- Contentful → Medusa: **Via Webhooks** (must be configured)

**What syncs back from Contentful:**

| Field | Syncs Back? | Method | Notes |
|-------|-------------|--------|-------|
| Product Title | ✅ Yes | Webhook | If changed in Contentful |
| Product Description | ✅ Yes | Webhook | Base description field |
| Product Metadata | ✅ Yes | Webhook | Custom fields |
| Variant Title | ✅ Yes | Webhook | If changed in Contentful |
| **Localized Content** | ❌ No | N/A | **Stays in Contentful only** |
| **Rich Media** | ❌ No | N/A | **Managed in Contentful** |
| **Language Translations** | ❌ No | N/A | **Content layer only** |

**Why selective sync?** Medusa is the source of truth for commerce operations. Syncing everything back could break pricing, inventory, or order fulfillment logic. Contentful adds a content layer on top of commerce structure.

### Doesn't Sync At All ⛔

These are managed independently in each system:

**Medusa Only (Commerce):**
- Prices and currency conversions
- Inventory levels (real-time stock)
- Orders and carts
- Customers and authentication
- Shipping and fulfillment
- Payment processing
- Discount codes and promotions
- Sales channels

**Contentful Only (Content):**
- Localized descriptions (per language)
- Marketing copy and editorial content
- Media library assets (images, videos, PDFs)
- Content drafts and unpublished changes
- Publishing schedules
- Content workflows (review/approval)
- SEO fields (per locale)
- A/B test variations

## Technology Stack

### Core Components

**Medusa v1.8+**
- Node.js e-commerce platform
- PostgreSQL database
- Redis (event bus + caching)
- Express.js API server

**Contentful (SaaS)**
- Enterprise headless CMS
- Multi-language content management
- Global CDN (Fastly-powered)
- Content Management API
- Content Delivery API
- Preview API
- GraphQL API

**Integration Plugin**
- `medusa-plugin-contentful` (installed in Medusa)
- Event-driven synchronization
- Webhook handler for reverse sync
- Content model migrations

### Supporting Infrastructure

**Required:**
- Node.js v16+ (Medusa)
- PostgreSQL v12+ (Medusa database)
- Redis v6+ (for Medusa event bus)
- Contentful account (free tier available)

**Optional:**
- CDN (Contentful provides built-in CDN)
- Caching layer (Redis for API responses)
- Load balancer (for high-traffic storefronts)
- Monitoring (Contentful analytics dashboard)

### API Architecture

**Contentful provides 3 APIs:**

1. **Content Management API** (CMA)
   - Used by Medusa to create/update entries
   - Requires management tokens
   - Write operations
   - Used for sync: Medusa → Contentful

2. **Content Delivery API** (CDA)
   - Used by storefront to fetch published content
   - Requires delivery tokens
   - Read-only, cached
   - Fast, CDN-delivered

3. **Content Preview API** (CPA)
   - Used for draft content preview
   - Requires preview tokens
   - Shows unpublished changes
   - Used by content editors

## Two-Way Sync Explained

### Direction 1: Medusa → Contentful (Automatic)

**How it works:**

```
1. Product created in Medusa Admin
   ↓
2. Medusa emits event: "product.created"
   ↓
3. Redis event bus distributes to subscribers
   ↓
4. Contentful subscriber catches event
   ↓
5. Fetch full product data from Medusa
   ↓
6. Transform to Contentful format
   ↓
7. Authenticate with Contentful Management API
   ↓
8. Create entry in Contentful
   ↓
9. Success ✓
```

**Events that trigger sync:**
- `product.created` → Create in Contentful
- `product.updated` → Update in Contentful
- `product.deleted` → Delete in Contentful
- `product_variant.created` → Create variant in Contentful
- `product_variant.updated` → Update variant in Contentful
- `product_collection.created` → Create collection in Contentful
- `product_collection.updated` → Update collection in Contentful
- `product_category.created` → Create category in Contentful
- `region.created` → Create region in Contentful
- `region.updated` → Update region in Contentful

**Required:**
- Redis running (for event bus)
- Contentful Management API token configured
- Content models created in Contentful

### Direction 2: Contentful → Medusa (Via Webhooks)

**How it works:**

```
1. Content editor updates entry in Contentful UI
   ↓
2. Entry saved (and published)
   ↓
3. Contentful fires webhook
   ↓
4. POST request to: https://your-backend.com/hooks/contentful
   ↓
5. Medusa webhook handler receives request
   ↓
6. Validates webhook payload
   ↓
7. Extract changes and medusa_id reference
   ↓
8. Update product in Medusa database
   ↓
9. Success ✓
```

**Events that trigger webhook:**
- Entry published → Update in Medusa
- Entry unpublished → Update status in Medusa
- Entry deleted → Delete in Medusa
- Entry archived → Update status in Medusa

**Required:**
- Medusa backend deployed and publicly accessible
- Webhook configured in Contentful settings
- Webhook endpoint: `https://your-backend.com/hooks/contentful`

**Important**: Webhooks only work with deployed backends. Local development requires tools like ngrok or Cloudflare Tunnel.

## Content Model Structure

### Contentful Content Types

The integration uses these content types in Contentful:

| Content Type | Purpose | Key Fields |
|--------------|---------|------------|
| `product` | Main product | `medusaId`, `title`, `description`, `localizedDescription` (multi-locale) |
| `productVariant` | Product variations | `medusaId`, `title`, `sku`, `productRef` |
| `productCollection` | Product groupings | `medusaId`, `title`, `handle`, `products` |
| `productCategory` | Product taxonomy | `medusaId`, `name`, `handle`, `parent` |
| `region` | Geographic regions | `medusaId`, `name`, `currencyCode`, `countries` |

### Example: Product Content Type

```javascript
// In Contentful
{
  "name": "Product",
  "description": "E-commerce product with localized content",
  "displayField": "title",
  "fields": [
    {
      "id": "medusaId",
      "name": "Medusa ID",
      "type": "Symbol",
      "required": true,
      "unique": true,
      "localized": false
    },
    {
      "id": "title",
      "name": "Title",
      "type": "Symbol",
      "required": true,
      "localized": true  // Can be different per language
    },
    {
      "id": "description",
      "name": "Description",
      "type": "Text",
      "localized": true
    },
    {
      "id": "richDescription",
      "name": "Rich Description",
      "type": "RichText",
      "localized": true  // Full markdown/rich content per language
    },
    {
      "id": "images",
      "name": "Product Images",
      "type": "Array",
      "items": {
        "type": "Link",
        "linkType": "Asset"
      }
    },
    {
      "id": "seoTitle",
      "name": "SEO Title",
      "type": "Symbol",
      "localized": true
    },
    {
      "id": "seoDescription",
      "name": "SEO Description",
      "type": "Text",
      "localized": true
    },
    {
      "id": "productCollection",
      "name": "Collection",
      "type": "Link",
      "linkType": "Entry",
      "validations": [{
        "linkContentType": ["productCollection"]
      }]
    },
    {
      "id": "productCategories",
      "name": "Categories",
      "type": "Array",
      "items": {
        "type": "Link",
        "linkType": "Entry",
        "validations": [{
          "linkContentType": ["productCategory"]
        }]
      }
    }
  ]
}
```

### Localization in Contentful

**Key Feature**: Fields can be marked as `localized: true`, allowing different content per language:

```javascript
// English (en-US)
{
  "title": "Organic Cotton T-Shirt",
  "richDescription": "A comfortable t-shirt made from 100% organic cotton...",
  "seoTitle": "Organic Cotton T-Shirt | Sustainable Fashion"
}

// Spanish (es-ES)
{
  "title": "Camiseta de Algodón Orgánico",
  "richDescription": "Una camiseta cómoda hecha de algodón 100% orgánico...",
  "seoTitle": "Camiseta de Algodón Orgánico | Moda Sostenible"
}

// German (de-DE)
{
  "title": "Bio-Baumwolle T-Shirt",
  "richDescription": "Ein bequemes T-Shirt aus 100% Bio-Baumwolle...",
  "seoTitle": "Bio-Baumwolle T-Shirt | Nachhaltige Mode"
}
```

**Non-localized fields** (like `medusaId`, `sku`) remain the same across all languages.

## Quick Start Summary

### Installation Overview

**1. Set Up Contentful Space (15 minutes)**
```bash
# 1. Create account at https://www.contentful.com
# 2. Create new space (or use existing)
# 3. Note your Space ID (Settings → General Settings)
# 4. Create Management API token (Settings → API keys → Content management tokens)
```

**2. Install Medusa Plugin (5 minutes)**
```bash
cd /path/to/medusa-backend
npm install medusa-plugin-contentful

# Add to .env
CONTENTFUL_SPACE_ID=your_space_id_here
CONTENTFUL_ACCESS_TOKEN=your_management_token_here
CONTENTFUL_ENV=master
```

**3. Configure Plugin (5 minutes)**
```javascript
// medusa-config.js
module.exports = {
  plugins: [
    {
      resolve: `medusa-plugin-contentful`,
      options: {
        space_id: process.env.CONTENTFUL_SPACE_ID,
        access_token: process.env.CONTENTFUL_ACCESS_TOKEN,
        environment: process.env.CONTENTFUL_ENV,
      },
    },
  ],
}
```

**4. Create Content Models (15 minutes)**
```bash
# Run migration scripts to create content types
# See implementation guide for detailed scripts
node scripts/contentful-migrations.js
```

**5. Set Up Webhooks (10 minutes)**
```bash
# In Contentful dashboard:
# Settings → Webhooks → Add Webhook
# URL: https://your-backend.com/hooks/contentful
# Method: POST
# Content-Type: application/json
```

**6. Start Medusa (2 minutes)**
```bash
npx medusa develop
# Check logs for "Contentful integration initialized"
```

**7. Test Integration (5 minutes)**
```bash
# Create product in Medusa → Check Contentful
# Update content in Contentful → Check Medusa
```

**Total Time:** ~60 minutes for complete setup

For detailed step-by-step instructions, see [Implementation Guide](./04-implementation.md).

## Key Features

### 1. Automatic Data Synchronization

**Event-Driven Architecture:**
- Medusa emits events when data changes
- Plugin subscribers catch events
- Transform and send to Contentful API
- No manual sync needed

**Real-time Updates:**
- Product created → Appears in Contentful within seconds
- Variant updated → Synced to Contentful automatically
- Collection modified → Related products updated

### 2. Localization Built-In

**Multi-Language Support:**
- Define locales in Contentful (en-US, es-ES, de-DE, fr-FR, etc.)
- Content editors provide translations per locale
- Storefront fetches content in user's language
- Fallback to default locale if translation missing

**Example:**
```javascript
// Fetch product in user's language
const product = await contentful.getEntries({
  content_type: 'product',
  'fields.medusaId': 'prod_123',
  locale: 'es-ES'  // Spanish
});
```

### 3. Rich Media Management

**Contentful Assets:**
- Upload images, videos, PDFs to Contentful
- Automatic optimization and CDN delivery
- On-the-fly image transformations
- Responsive image URLs

**Image Transformations:**
```javascript
// Original
https://images.ctfassets.net/space/asset.jpg

// Resized to 800x600
https://images.ctfassets.net/space/asset.jpg?w=800&h=600

// WebP format
https://images.ctfassets.net/space/asset.jpg?fm=webp

// Quality 80%
https://images.ctfassets.net/space/asset.jpg?q=80
```

### 4. Content Workflows

**Draft → Review → Publish:**
- Content editors create drafts
- Reviewers approve changes
- Published content goes live
- Unpublished content hidden from storefront

**Version History:**
- Every change tracked
- Rollback to previous versions
- Compare versions
- Audit trail

### 5. Preview Mode

**Before Publishing:**
- Content editors preview changes
- Use Preview API to see drafts
- Test on staging storefront
- Publish when ready

### 6. Webhooks for Real-Time Sync

**Contentful → Medusa:**
- Entry published → Update Medusa
- Entry unpublished → Update status
- Entry deleted → Remove from Medusa
- Entry archived → Archive in Medusa

## Common Use Cases

### ✅ Perfect For

1. **Multi-Language E-commerce**
   - International stores with localized content
   - Product descriptions in multiple languages
   - Locale-specific marketing copy
   - Regional content variations

2. **Enterprise Content Management**
   - Large teams with content workflows
   - Editorial approval processes
   - Scheduled content publishing
   - Role-based access control

3. **Rich Product Content**
   - Detailed product descriptions
   - High-resolution image galleries
   - Product videos and demos
   - Downloadable resources (PDFs, manuals)

4. **Content-First Commerce**
   - Editorial-driven product pages
   - Storytelling and brand narratives
   - Magazine-style content
   - Blog integration

5. **Managed Infrastructure**
   - No DevOps for CMS
   - Global CDN included
   - Automatic scaling
   - Enterprise SLA

### ❌ Not Ideal For

1. **Budget Constraints**
   - Free tier is limited (10 req/s, 2 spaces)
   - Growing usage = growing costs
   - Bandwidth charges apply
   - Better alternatives: Strapi (FOSS), Payload (self-hosted)

2. **Simple Catalogs**
   - Basic products with minimal content
   - Single language only
   - No rich media requirements
   - Simple descriptions sufficient

3. **Vendor Lock-In Concerns**
   - Proprietary platform (not open source)
   - Data export available but complex
   - Migration to other CMS is work-intensive
   - Better alternatives: Strapi, Payload (open source)

4. **Full Self-Hosting Required**
   - Contentful is SaaS only (no self-hosted option)
   - Data lives in Contentful cloud
   - Better alternatives: Strapi, Payload

5. **Simple Content Needs**
   - Don't need localization
   - Don't need workflows
   - Don't need enterprise features
   - Medusa's built-in fields are enough

## Architecture Principles

### Separation of Concerns

**Medusa Handles:**
- Product catalog structure
- Pricing and currency
- Inventory management
- Order processing
- Customer management
- Payment processing
- Shipping and fulfillment

**Contentful Handles:**
- Localized product content
- Rich media assets
- Marketing copy
- Editorial workflows
- SEO optimization
- Content publishing
- Preview and versioning

**Why This Design?**
- Clear ownership boundaries
- Scalability (each system scales independently)
- Teams work independently (developers vs content editors)
- Best tool for each job (commerce vs content)

### Event-Driven Sync

**How It Works:**
1. Medusa emits event (e.g., "product.created")
2. Redis event bus distributes event
3. Contentful subscriber receives event
4. Fetches full data from Medusa
5. Transforms to Contentful format
6. Posts to Contentful Management API

**Benefits:**
- Asynchronous (non-blocking)
- Resilient (retries on failure)
- Decoupled (systems don't directly depend)
- Scalable (can process many events)

### Content Model Mapping

**Medusa ID as Reference:**
```javascript
// Medusa
{
  id: "prod_01HQ5K9ZFWXYZ123ABC",
  title: "Organic Cotton T-Shirt",
  // ... other fields
}

// Contentful
{
  medusaId: "prod_01HQ5K9ZFWXYZ123ABC",  // Reference to Medusa
  title: {
    "en-US": "Organic Cotton T-Shirt",
    "es-ES": "Camiseta de Algodón Orgánico"
  },
  // ... localized content
}
```

**Plugin handles translation automatically** - No manual mapping needed!

### Authentication Flow

**Medusa → Contentful:**
1. Plugin loads with management API token
2. Token stored securely in environment
3. Makes authenticated requests to Contentful
4. Token valid for long periods (no refresh needed)

**Contentful → Medusa:**
1. Webhook configured with optional secret
2. Contentful sends POST request to Medusa
3. Medusa validates request (optional: verify webhook signature)
4. Processes update

## What's Next?

Now that you understand the basics, explore:

- **[Architecture Deep Dive](./02-architecture.md)** - Technical implementation details
- **[Data Flow](./03-data-flow.md)** - Step-by-step synchronization scenarios
- **[Implementation Guide](./04-implementation.md)** - Set it up yourself
- **[Pros & Cons](./05-pros-cons.md)** - Decision-making guide
- **[FAQ](./FAQ.md)** - Common questions and answers

Or jump straight to:
- **[Quick Start in README](./README.md#quick-start)** - Get running in 1 hour
- **[Troubleshooting](./04-implementation.md#troubleshooting)** - Fix common issues

