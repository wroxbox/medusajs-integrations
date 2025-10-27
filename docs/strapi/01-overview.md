# Overview: Strapi Integration with Medusa

## What is the Strapi Integration?

The Strapi integration is a **master-slave architecture** that connects Medusa v1 (your e-commerce backend) with Strapi v4 (a powerful open-source headless CMS). This allows you to:

- **Manage commerce in Medusa**: Products, variants, pricing, inventory, orders
- **Manage content in Strapi**: Rich descriptions, editorial content, media assets, marketing copy
- **Automatically sync data**: Product structure flows from Medusa to Strapi
- **Empower content teams**: Non-technical users can enrich product content without touching code

### The Core Concept

```
Medusa (Master)  ────sync───► Strapi (Slave)  
Commerce Data                  Content Layer
     ↓                              ↓
     └──────── Storefront ──────────┘
           Fetches from both
```

**Medusa is the master** - It owns all commerce data and product structure. When you create/update products in Medusa, those changes automatically flow to Strapi.

**Strapi is the slave** - It receives product structure from Medusa and adds a content layer on top. Content teams enrich products with descriptions, images, and marketing content.

**Storefront reads from both** - It fetches commerce data (prices, inventory, cart) from Medusa and content data (descriptions, images) from Strapi.

## Why Use Strapi with Medusa?

### The Problem

Standard Medusa installations store basic product information:
- Product title
- Simple description (text field)
- Handle and metadata
- Thumbnail URL

But for **content-rich e-commerce**, you often need:
- Long-form product narratives
- Multiple high-resolution images with captions
- Structured content sections (Features, Specifications, Care Instructions)
- Marketing copy and SEO content
- Editorial workflows and publishing schedules
- Media asset management

### The Solution

Strapi provides a professional CMS layer for your commerce platform:

| Without Strapi | With Strapi |
|----------------|-------------|
| Basic text description | Rich text editor with formatting |
| Single thumbnail | Media library with multiple images |
| All content in code/database | Visual admin panel for content teams |
| Developers manage all content | Content managers work independently |
| No content workflow | Draft/publish workflows |
| Limited content structure | Custom content types and fields |

## What Gets Synced (and What Doesn't)

### Automatic Sync: Medusa → Strapi ✅

When you create or update entities in Medusa, they **automatically sync to Strapi**:

**Products:**
- Product ID (stored as `medusa_id` in Strapi)
- Title, subtitle, handle
- Description (base content)
- Thumbnail URL
- Dimensions (height, width, length, weight)
- Metadata

**Variants:**
- Variant ID, title, SKU
- Options (size, color, etc.)
- Prices (money amounts)
- Inventory info
- Material, origin country

**Collections:**
- Collection ID, title, handle
- Product relationships

**Categories:**
- Category ID, name, handle
- Hierarchy and relationships

**Regions:**
- Region ID, name, tax info
- Currency
- Payment providers
- Fulfillment providers

**Metafields:**
- Custom product metadata
- Key-value pairs

### Limited Sync Back: Strapi → Medusa ⚠️

Only **basic fields** sync back from Strapi to Medusa:

| Field | Syncs Back? | Notes |
|-------|-------------|-------|
| Product Title | ✅ Yes | If changed in Strapi |
| Product Subtitle | ✅ Yes | If changed in Strapi |
| Variant Title | ✅ Yes | If changed in Strapi |
| Region Name | ✅ Yes | If changed in Strapi |
| **Rich Content** | ❌ No | Stays in Strapi only |
| **Images** | ❌ No | Managed in Strapi |
| **Custom Fields** | ❌ No | CMS-only content |

**Why limited?** Medusa is the source of truth for commerce operations. Syncing too much back could break pricing, inventory, or order fulfillment logic.

### Doesn't Sync At All ⛔

These are managed independently in each system:

**Medusa Only (Commerce):**
- Prices and currency
- Inventory levels
- Orders and carts
- Customers and authentication
- Shipping and fulfillment
- Payment processing
- Discount codes and promotions

**Strapi Only (Content):**
- Rich text descriptions
- Marketing copy
- Editorial content
- Media library assets
- Content drafts
- Publishing schedules
- Custom content types
- SEO fields (via plugins)

## Technology Stack

### Core Components

**Medusa v1.8+**
- Node.js e-commerce platform
- PostgreSQL database
- Redis (event bus + caching)
- Express.js API server

**Strapi v4.13+**
- Node.js headless CMS
- PostgreSQL database (separate from Medusa)
- Built-in admin panel
- REST/GraphQL API

**Integration Plugins**
- `medusa-plugin-strapi-ts` (installed in Medusa)
- `strapi-plugin-medusajs` (installed in Strapi)

### Supporting Infrastructure

**Required:**
- Node.js v16+ (both systems)
- PostgreSQL v12+ (two databases: one for Medusa, one for Strapi)
- Redis v6+ (for Medusa event bus)

**Optional:**
- Media storage (S3 or local)
- Search (MeiliSearch via Strapi plugin)
- Monitoring (Sentry via Strapi plugin)
- CDN (for Strapi media delivery)

### Monorepo Structure

The official implementation uses Lerna monorepo with 5 packages:

```
medusa-strapi-repo/
├── packages/
│   ├── medusa-plugin-strapi-ts/      # Medusa plugin (sync engine)
│   ├── medusa-strapi/                # Strapi server template
│   ├── strapi-plugin-medusajs/       # Strapi plugin (receives data)
│   ├── strapi-plugin-sso-medusa/     # SSO between systems
│   └── strapi-plugin-multi-country-select/  # Admin UI enhancement
```

## Legacy Considerations

### ⚠️ Important: This is a Legacy Integration

**Built For:**
- Medusa v1 (released 2021-2022)
- Strapi v4 (released 2021)

**Current Status:**
- Medusa v2 released in 2024 with breaking changes
- Strapi v5 in development
- Community-maintained (not official Medusa plugin)
- Limited ongoing development

**Migration Challenges:**

| Aspect | Difficulty | Notes |
|--------|------------|-------|
| Medusa v1 → v2 | ⚠️ Hard | Complete architecture change |
| Strapi v4 → v5 | 🟡 Medium | Breaking changes expected |
| To Sanity | 🟡 Medium | Different data model |
| To Payload | 🟡 Medium | Different architecture |

**Recommendation:**
- ✅ **Use if**: Already on Medusa v1 and need CMS now
- ✅ **Use if**: Require FOSS solution with full control
- ⚠️ **Caution if**: Starting new project (consider Sanity/Payload)
- ❌ **Avoid if**: On Medusa v2 (use newer integrations)

## Quick Start Summary

### Installation Overview

**1. Set Up Strapi (30 minutes)**
```bash
# Clone template with pre-configured content types
git clone https://github.com/SGFGOV/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi

# Configure environment
cp .env.test .env
# Edit .env with your settings

# Install and build
yarn install
yarn build

# Start Strapi
yarn develop  # http://localhost:1337/admin
```

**2. Install Medusa Plugin (10 minutes)**
```bash
cd /path/to/medusa-backend
yarn add medusa-plugin-strapi-ts

# Configure in medusa-config.js
# Set environment variables
```

**3. Connect Systems (5 minutes)**
- Start both servers
- Service account auto-created in Strapi
- Initial sync triggered automatically

**4. Test Integration (5 minutes)**
- Create product in Medusa Admin
- Check Strapi Admin for synced product
- Enrich content in Strapi
- View on storefront

**Total Time:** ~1 hour for basic setup

For detailed step-by-step instructions, see [Implementation Guide](./04-implementation.md).

## Key Features

### Automatic Data Synchronization

**Event-Driven Architecture:**
- Medusa emits events when data changes
- Plugin subscribers catch events
- Transform and send to Strapi API
- No manual sync needed

**Synced Events:**
```
product.created      → Create in Strapi
product.updated      → Update in Strapi
product.deleted      → Delete in Strapi
variant.created      → Create in Strapi
variant.updated      → Update in Strapi
collection.updated   → Update in Strapi
category.created     → Create in Strapi
region.updated       → Update in Strapi
```

### Service Account Management

**Automatic Setup:**
- Super admin created on first boot
- Service account auto-registered
- JWT authentication handled automatically
- Token refresh managed by plugin

**Security:**
- Encrypted credentials
- Shared JWT secret between systems
- Role-based access in Strapi

### Content Type Generation

**Pre-Built Strapi Content Types:**
- Products (with all commerce fields)
- Product Variants
- Product Collections
- Product Categories
- Product Types
- Product Tags
- Product Options
- Product Option Values
- Regions
- Money Amounts
- Currencies
- Countries
- Fulfillment Providers
- Payment Providers
- Shipping Options

**All Mapped to Medusa Structure** - No manual content type creation needed!

### Two-Way Communication

**Medusa → Strapi (Primary):**
- Full product structure
- Commerce metadata
- Relationships preserved

**Strapi → Medusa (Limited):**
- Basic field updates (title, subtitle)
- Prevents commerce data corruption
- Content stays in CMS layer

**API Endpoint in Medusa:**
```bash
GET /strapi/content/:contentType/:medusaId
```
Proxy endpoint to fetch Strapi content through Medusa.

## Common Use Cases

### 1. Editorial E-commerce

**Scenario:** Fashion brand with rich product storytelling

**Implementation:**
- Create products in Medusa (SKUs, prices, inventory)
- Content team adds narratives in Strapi
- High-res lookbook images in Strapi media library
- Styling tips and care instructions as custom fields
- Seasonal collection marketing content

**Benefit:** Designers and copywriters manage content without developer involvement.

### 2. Technical Product Catalog

**Scenario:** Electronics store with detailed specifications

**Implementation:**
- Products and variants in Medusa
- Technical specs table in Strapi
- Multiple product images (angle views, closeups)
- User manuals uploaded to Strapi media library
- Comparison charts for product categories

**Benefit:** Technical writers maintain accurate specs independently.

### 3. Content-First Commerce

**Scenario:** Artisanal goods marketplace with maker stories

**Implementation:**
- Basic product info in Medusa
- Maker biographies in Strapi
- Process photos and videos
- Sustainability credentials
- Related blog posts and articles

**Benefit:** Editorial team creates compelling brand narratives.

### 4. Multi-Brand Store

**Scenario:** Retailer selling multiple brands

**Implementation:**
- All products in Medusa
- Brand-specific content templates in Strapi
- Brand pages and lookbooks
- Unique styling per brand
- Centralized inventory management

**Benefit:** Separate content management per brand, unified commerce.

## Architecture Principles

### Master-Slave Relationship

**Medusa is Master:**
- Owns product structure
- Controls commerce operations
- Source of truth for IDs
- Triggers synchronization

**Strapi is Slave:**
- Receives product structure
- Adds content layer
- Stores references to Medusa IDs
- Enriches with editorial content

**Why This Design?**
- Clear ownership boundaries
- Prevents commerce data corruption
- Content can be rebuilt from Medusa
- Reduces synchronization conflicts

### Event-Driven Sync

**How It Works:**
1. Medusa emits event (e.g., "product.created")
2. Redis event bus distributes event
3. Strapi subscriber receives event
4. Fetches full data from Medusa
5. Transforms to Strapi format
6. Posts to Strapi API

**Benefits:**
- Asynchronous (non-blocking)
- Resilient (retries on failure)
- Decoupled (systems don't directly depend)
- Scalable (can process many events)

### Data Model Mapping

**Medusa uses underscores:**
```javascript
{
  collection: {...},
  type: {...},
  tags: [...]
}
```

**Strapi uses product_ prefix:**
```javascript
{
  product_collection: {...},
  product_type: {...},
  product_tags: [...]
}
```

**Plugin handles translation automatically** - No manual mapping needed!

### Authentication Flow

**Initial Setup:**
1. Strapi super admin created (via env vars)
2. Plugin registers Medusa service account
3. Service account gets JWT token
4. Token cached in memory

**Ongoing Operations:**
1. Plugin retrieves cached token
2. Makes authenticated API calls to Strapi
3. Token auto-refreshes on expiration
4. Falls back to re-authentication if needed

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

