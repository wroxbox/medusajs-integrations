# Summary: Quick Reference Guide

This is a condensed reference for developers working with the Medusa-Strapi integration.

## At a Glance

**What:** Connects Medusa v1 (e-commerce) with Strapi v4 (CMS) for rich content management.

**How:** Event-driven sync from Medusa to Strapi via HTTP API.

**Who:** Content-rich e-commerce stores on Medusa v1 needing FOSS CMS.

**When:** Legacy solution (v1 only), consider Sanity/Payload for v2.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                  MEDUSA v1 (Master)                      │
│  ┌────────────────────────────────────────────────────┐ │
│  │  medusa-plugin-strapi-ts                           │ │
│  │  • Event Subscribers → Listen for changes          │ │
│  │  • UpdateStrapiService → Transform & sync          │ │
│  │  • UpdateMedusaService → Receive from Strapi       │ │
│  └────────────────────────────────────────────────────┘ │
│          PostgreSQL              Redis Event Bus         │
└────────────────────┬────────────────────────────────────┘
                     │ HTTP/REST + JWT Auth
                     ▼
┌─────────────────────────────────────────────────────────┐
│                   STRAPI v4 (Slave)                      │
│  ┌────────────────────────────────────────────────────┐ │
│  │  strapi-plugin-medusajs                            │ │
│  │  • Bootstrap → Create service account              │ │
│  │  • Sync Endpoint → Bulk operations                 │ │
│  │  • Content Types → Medusa schema mirror            │ │
│  └────────────────────────────────────────────────────┘ │
│          PostgreSQL (Separate)       Admin Panel         │
└─────────────────────────────────────────────────────────┘
                     ▲
                     │ REST/GraphQL API
                     │
              ┌──────┴──────┐
              │  STOREFRONT  │
              │  (Next.js)   │
              └──────────────┘
```

---

## Quick Setup

### 1. Strapi Server (30 min)

```bash
# Clone template
git clone https://github.com/SGFGOV/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi

# Configure
cp .env.test .env
# Edit .env with your settings (see implementation guide)

# Install & build
yarn install
yarn build

# Start
yarn develop  # http://localhost:1337/admin
```

### 2. Medusa Plugin (10 min)

```bash
# Install
cd /path/to/medusa-backend
yarn add medusa-plugin-strapi-ts

# Configure medusa-config.js
plugins: [
  {
    resolve: "medusa-plugin-strapi-ts",
    options: {
      strapi_protocol: "http",
      strapi_host: "localhost",
      strapi_port: 1337,
      strapi_secret: process.env.STRAPI_SECRET,  # MUST MATCH Strapi JWT
      strapi_default_user: {
        username: "medusa",
        password: process.env.STRAPI_MEDUSA_PASSWORD,
        email: "medusa@service.local"
      },
      strapi_admin: {
        username: process.env.STRAPI_SUPER_USERNAME,
        password: process.env.STRAPI_SUPER_PASSWORD,
        email: process.env.STRAPI_SUPER_USER_EMAIL
      },
      auto_start: true
    }
  }
]

# Start
npx medusa develop  # Port 9000
```

### 3. Test (5 min)

```bash
# Create product in Medusa Admin
# Check appears in Strapi Admin
# Edit content in Strapi
# Verify on storefront
```

---

## Key Files & Locations

### Medusa Side

```
medusa-backend/
├── medusa-config.js              # Plugin configuration
├── .env                          # Secrets (STRAPI_SECRET, etc.)
└── node_modules/
    └── medusa-plugin-strapi-ts/
        ├── src/
        │   ├── subscribers/
        │   │   └── strapi.ts     # Event listeners
        │   ├── services/
        │   │   ├── update-strapi.ts   # Medusa → Strapi sync
        │   │   └── update-medusa.ts   # Strapi → Medusa sync
        │   ├── api/
        │   │   └── routes/       # /strapi/content/* endpoints
        │   └── utils/
        │       └── transformations.ts  # Data format conversion
```

### Strapi Side

```
medusa-strapi/
├── config/
│   ├── database.js              # PostgreSQL connection
│   ├── server.js                # Port 1337, JWT settings
│   ├── plugins.js               # Enable medusajs plugin
│   └── middlewares.js           # CORS configuration
├── .env                         # Secrets (JWT_SECRET, etc.)
├── src/
│   ├── api/                     # Content types
│   │   ├── product/
│   │   ├── product-variant/
│   │   ├── product-collection/
│   │   └── ... (20+ types)
└── node_modules/
    └── strapi-plugin-medusajs/
        └── server/
            ├── bootstrap.ts     # Create service account
            └── controllers/
                └── setup.ts     # Bulk sync endpoint
```

---

## Environment Variables Checklist

### Medusa `.env`

```bash
# ✅ CRITICAL: Must match Strapi
STRAPI_SECRET=your_jwt_secret

# Strapi connection
STRAPI_PROTOCOL=http
STRAPI_SERVER_HOSTNAME=localhost
STRAPI_PORT=1337

# Service account
STRAPI_MEDUSA_USER=medusa
STRAPI_MEDUSA_PASSWORD=SecurePassword123!
STRAPI_MEDUSA_EMAIL=medusa@service.local

# Super admin (same as Strapi)
STRAPI_SUPER_USERNAME=SuperUser
STRAPI_SUPER_PASSWORD=AdminPassword123!
STRAPI_SUPER_USER_EMAIL=admin@yourdomain.com

# Redis (REQUIRED)
REDIS_URL=redis://localhost:6379
```

### Strapi `.env`

```bash
# ✅ CRITICAL: Must match Medusa STRAPI_SECRET
JWT_SECRET=your_jwt_secret
MEDUSA_STRAPI_SECRET=your_jwt_secret

# Security keys (generate random)
APP_KEYS=key1,key2,key3,key4
API_TOKEN_SALT=random_salt
ADMIN_JWT_SECRET=random_secret

# Medusa connection
MEDUSA_BACKEND_URL=http://localhost:9000
MEDUSA_BACKEND_ADMIN=http://localhost:7001

# Super admin
SUPERUSER_EMAIL=admin@yourdomain.com
SUPERUSER_USERNAME=SuperUser
SUPERUSER_PASSWORD=AdminPassword123!

# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=strapi_medusa
DATABASE_USERNAME=strapi_user
DATABASE_PASSWORD=your_db_password
```

---

## API Endpoints

### Medusa Endpoints

```bash
# Proxy to Strapi content
GET /strapi/content/:contentType/:medusaId
# Example: /strapi/content/products/prod_123

# Trigger manual sync (if implemented)
POST /admin/strapi/sync
```

### Strapi Endpoints

```bash
# Authentication
POST /api/auth/local
Body: { "identifier": "user@email.com", "password": "..." }
Response: { "jwt": "...", "user": {...} }

# Products
GET /api/products
GET /api/products/:id
GET /api/products?filters[medusa_id][$eq]=prod_123
POST /api/products  # Requires auth
PUT /api/products/:id  # Requires auth
DELETE /api/products/:id  # Requires auth

# Variants
GET /api/product-variants
GET /api/product-variants?filters[medusa_id][$eq]=variant_123

# Collections
GET /api/product-collections

# Bulk sync
POST /strapi-plugin-medusajs/synchronise-medusa-tables
# Requires super admin token
```

---

## Common Commands

### Development

```bash
# Start Strapi (dev mode)
cd medusa-strapi
yarn develop

# Start Medusa (dev mode)
cd medusa-backend
npx medusa develop

# Check Redis
redis-cli ping

# Check PostgreSQL
psql -U postgres -d strapi_medusa -c "SELECT COUNT(*) FROM products;"
```

### Production

```bash
# Build Strapi
yarn build

# Start Strapi (production)
NODE_ENV=production yarn start

# Start Medusa (production)
npx medusa start

# Check health
curl http://localhost:1337/_health  # Strapi
curl http://localhost:9000/health   # Medusa
```

### Troubleshooting

```bash
# Test Strapi connection from Medusa server
curl http://localhost:1337/_health

# Test authentication
curl -X POST http://localhost:1337/api/auth/local \
  -H "Content-Type: application/json" \
  -d '{"identifier":"medusa@service.local","password":"your_password"}'

# Check Redis
redis-cli KEYS "*_ignore_*"
redis-cli GET prod_123_ignore_strapi

# Check Strapi database
psql -U strapi_user -d strapi_medusa
\dt  # List tables
SELECT medusa_id, title FROM products LIMIT 5;
```

---

## Data Sync Reference

### What Syncs (Medusa → Strapi)

| Entity | Fields | Frequency |
|--------|--------|-----------|
| **Product** | medusa_id, title, subtitle, description, handle, thumbnail, weight, dimensions, metadata | On create/update |
| **Variant** | medusa_id, title, sku, material, weight, origin_country | On create/update |
| **Collection** | medusa_id, title, handle | On create/update |
| **Category** | medusa_id, name, handle, parent | On create/update |
| **Region** | medusa_id, name, currency, tax, providers | On create/update |
| **Prices** | amount, currency_code, region | With variant |
| **Options** | name, values | With product |

### What Doesn't Sync (Stays in Strapi)

- Rich text descriptions
- Media library images
- Custom CMS fields
- Content drafts
- SEO metadata
- Editorial notes

### What Doesn't Sync (Stays in Medusa)

- Orders & carts
- Customers
- Payments
- Inventory levels
- Discounts & promotions
- Shipping rules

---

## Event Reference

### Medusa Events (Trigger Sync)

```typescript
// Products
"product.created"       → createProductInStrapi()
"product.updated"       → updateProductInStrapi()
"product.deleted"       → deleteProductInStrapi()

// Variants
"product_variant.created"  → createProductVariantInStrapi()
"product_variant.updated"  → updateProductVariantInStrapi()
"product_variant.deleted"  → deleteProductVariantInStrapi()

// Collections
"product_collection.created"       → createCollectionInStrapi()
"product_collection.updated"       → updateCollectionInStrapi()
"product_collection.products_added"    → updateProductsWithinCollectionInStrapi()
"product_collection.products_removed"  → updateProductsWithinCollectionInStrapi()

// Categories
"product_category.created"  → createCategoryInStrapi()
"product_category.updated"  → updateCategoryInStrapi()

// Regions
"region.created"  → createRegionInStrapi()
"region.updated"  → updateRegionInStrapi()
"region.deleted"  → deleteRegionInStrapi()

// Metafields
"product.metafields.create"  → createProductMetafieldInStrapi()
"product.metafields.update"  → updateProductMetafieldInStrapi()
```

---

## Storefront Integration Patterns

### Pattern 1: Dual Fetch (Direct)

```typescript
// Fetch from both APIs
const product = await medusa.products.retrieve(id);
const content = await strapi.getProduct(id);

const enriched = {
  ...product,
  richDescription: content.rich_description,
  images: content.images
};
```

**Pros:** Full control, flexible queries  
**Cons:** Two network calls, higher latency

### Pattern 2: Medusa Proxy

```typescript
// Single call via Medusa
const product = await fetch(
  `${MEDUSA_URL}/strapi/content/products/${id}`
);
```

**Pros:** Single request, simpler frontend  
**Cons:** Additional hop, less flexible

### Pattern 3: Server-Side Merge

```typescript
// Next.js server component
async function ProductPage({ params }) {
  const [product, content] = await Promise.all([
    medusa.products.retrieve(params.id),
    strapi.getProduct(params.id)
  ]);
  
  return <Product {...product} {...content} />;
}
```

**Pros:** Parallel fetching, optimized  
**Cons:** Server-side only

---

## Troubleshooting Checklist

### Products Not Syncing

- [ ] Redis running? `redis-cli ping`
- [ ] Strapi reachable? `curl http://localhost:1337/_health`
- [ ] Secrets match? Check `STRAPI_SECRET` vs `JWT_SECRET`
- [ ] Subscriber loaded? Check logs for "Strapi Subscriber Initialized"
- [ ] Event bus configured? Check `redis_url` in `medusa-config.js`

### Authentication Errors

- [ ] Service account exists in Strapi Admin → Users?
- [ ] Super admin credentials correct?
- [ ] Can log in manually via curl?
- [ ] Token not expired? (3 min cache)

### Slow Sync

- [ ] Network latency? `ping strapi-server`
- [ ] Database indexes? Check `medusa_id` indexes
- [ ] Rate limiting? Check for 429 errors
- [ ] Redis connection? Check latency

### Storefront Issues

- [ ] CORS configured? Check `middlewares.js`
- [ ] Content exists in Strapi? Check admin panel
- [ ] API calls successful? Check browser Network tab
- [ ] Proper error handling? Check console logs

---

## Quick Comparison

### vs. No CMS (Medusa Only)

| Aspect | Medusa Only | + Strapi |
|--------|-------------|----------|
| Setup | ✅ Simple | ❌ Complex |
| Cost | ✅ Low | ❌ High |
| Content | ❌ Basic | ✅ Rich |
| Team | ❌ Devs only | ✅ Independent |
| Maintenance | ✅ Easy | ❌ Hard |

### vs. Sanity (Medusa v2)

| Aspect | Strapi | Sanity |
|--------|--------|--------|
| Medusa v2 | ❌ No | ✅ Yes |
| FOSS | ✅ Yes | ❌ No |
| Self-Hosted | ✅ Yes | ❌ No |
| Managed | ❌ No | ✅ Yes |
| Setup | ❌ Hard | ✅ Easy |

### vs. Payload (Medusa v2)

| Aspect | Strapi | Payload |
|--------|--------|---------|
| Medusa v2 | ❌ No | ✅ Yes |
| Deployment | Separate | Embedded |
| Database | Separate | Shared |
| Cost | 2x infra | 1x infra |
| TypeScript | Old | Native |

---

## Performance Metrics

### Sync Performance

- Single product: ~335ms
- Bulk sync: ~50-100 products/min
- API latency: ~150-200ms
- Database write: ~50-80ms

### Storefront Performance

- Medusa-only fetch: ~150ms
- Dual fetch: ~350ms (+200ms)
- Proxy fetch: ~250ms (+100ms)

### Scalability

- Tested with: 10,000+ products ✓
- Concurrent syncs: 5-10/sec
- Database size: ~1GB per 10k products
- API rate limit: Configurable (Strapi)

---

## Resource Links

### Official Documentation

- [Medusa v1 Docs](https://docs.medusajs.com/v1/)
- [Medusa Strapi Plugin Docs](https://docs.medusajs.com/v1/plugins/cms/strapi)
- [Strapi v4 Docs](https://docs.strapi.io/dev-docs/intro)
- [GitHub Repository](https://github.com/SGFGOV/medusa-strapi-repo)

### Community

- [Medusa Discord](https://discord.gg/medusajs) - #plugins channel
- [Strapi Discord](https://discord.strapi.io)
- Plugin Maintainer: @govdiw on Discord

### Related Integrations

- [Sanity Integration (v2)](https://github.com/medusajs/examples/tree/main/sanity-integration)
- [Payload Integration (v2)](https://github.com/medusajs/examples/tree/main/payload-integration)

---

## Decision Matrix

### Use Strapi If:

1. ✅ On Medusa v1 (not v2)
2. ✅ Need rich content management
3. ✅ Have dedicated content team
4. ✅ Require FOSS solution
5. ✅ Have DevOps resources
6. ✅ Budget allows extra infrastructure
7. ✅ Content is competitive advantage

### Don't Use Strapi If:

1. ❌ On Medusa v2 (use Sanity/Payload)
2. ❌ Simple product catalog
3. ❌ Solo developer
4. ❌ Tight budget/timeline
5. ❌ Need quick MVP
6. ❌ Prefer managed services

---

## Version Compatibility

| Software | Version | Status |
|----------|---------|--------|
| Node.js | v16+ | Required |
| Medusa | v1.8+ | Required |
| Strapi | v4.13+ | Required |
| PostgreSQL | v12+ | Required |
| Redis | v6+ | Required |
| Medusa v2 | 2.x | ❌ Not supported |
| Strapi v5 | 5.x | ❌ Not tested |

---

## What's Next?

- **Need details?** → [Overview](./01-overview.md)
- **Want technical depth?** → [Architecture](./02-architecture.md)
- **See examples?** → [Data Flow](./03-data-flow.md)
- **Ready to build?** → [Implementation](./04-implementation.md)
- **Making decision?** → [Pros & Cons](./05-pros-cons.md)
- **Have questions?** → [FAQ](./FAQ.md)
- **Full guide** → [README](./README.md)

---

**Quick Start:** Clone repo → Configure → Start Strapi → Install plugin → Test (~1 hour)

**Best For:** Content-rich e-commerce on Medusa v1 needing FOSS CMS

**Legacy Warning:** Built for v1, not v2. Plan migration path.

