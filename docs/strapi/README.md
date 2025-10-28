# Strapi CMS Integration with Medusa

**Complete documentation for integrating Strapi CMS with Medusa v1 for headless e-commerce content management.**

> ⚠️ **Legacy Notice**: This integration is built for Medusa v1 and Strapi v4. While functional, consider evaluating Sanity or Payload integrations for new projects on Medusa v2.

## Quick Overview

This integration combines:
- **Medusa v1** - Headless e-commerce platform (products, pricing, orders, inventory)
- **Strapi v4** - Open-source headless CMS (rich content, editorial management)
- **Storefront** - Fetches commerce data from Medusa and content from Strapi

### Why This Architecture?

Traditional e-commerce stores all content in one database. This integration **separates commerce from content**, giving you:

✅ **Full Content Control** - Rich editorial capabilities for product content  
✅ **Open Source Freedom** - No vendor lock-in, self-hosted FOSS solution  
✅ **Powerful CMS Features** - Strapi's robust content types and relationships  
✅ **Automatic Syncing** - Product data flows from Medusa to Strapi automatically  
✅ **Content Team Independence** - Non-technical users can manage content without touching code  

### The Trade-off

While powerful, this adds complexity:

❌ Two separate systems to manage  
❌ Additional infrastructure (Strapi server + PostgreSQL database)  
❌ Legacy versions (Medusa v1 + Strapi v4)  
❌ Data synchronization to maintain  
❌ Dual data fetching in storefront  
❌ Limited community support compared to newer integrations  

**Use this integration if**: You need powerful content management, want a FOSS solution, and are running Medusa v1 or can maintain legacy systems.

**Skip this integration if**: You're starting fresh with Medusa v2, have simple content needs, or want the latest supported integrations (consider Sanity or Payload instead).

## Documentation Structure

### 📚 [1. Overview](./01-overview.md)
**Start here** - Understand what this integration does, why it exists, and key benefits.

- What is Strapi integration?
- Why use Strapi with Medusa?
- What gets synced (and what doesn't)
- Technology stack overview
- Legacy considerations
- Quick start summary

### 🏗️ [2. Architecture Deep Dive](./02-architecture.md)
**Technical details** - How the integration works under the hood.

- System components explained
- Medusa Strapi Plugin (`medusa-plugin-strapi-ts`)
- Strapi Medusa Plugin (`strapi-plugin-medusajs`)
- Event subscribers and sync mechanisms
- Strapi content types and schemas
- Data fetching patterns (proxy vs direct)
- Authentication and security

### 🔄 [3. Data Flow](./03-data-flow.md)
**See it in action** - Step-by-step walkthroughs of common scenarios.

- Creating a new product (Medusa → Strapi)
- Enriching content in Strapi Admin
- Customer viewing product (storefront fetch)
- Product updates and syncing
- Product deletion
- Two-way sync limitations
- Manual sync operations

### 🛠️ [4. Implementation Guide](./04-implementation.md)
**Build it yourself** - Step-by-step setup instructions.

- Prerequisites and requirements
- Part 1: Set up Strapi v4 server
- Part 2: Configure Strapi plugins
- Part 3: Set up Medusa backend
- Part 4: Configure Medusa plugin
- Part 5: Connect the systems
- Part 6: Test the integration
- Troubleshooting common issues
- Production deployment guide

### ⚖️ [5. Pros & Cons](./05-pros-cons.md)
**Make the decision** - Detailed analysis of trade-offs.

- When to use this integration
- When NOT to use it
- Detailed pros (FOSS, control, features)
- Detailed cons (legacy, complexity, maintenance)
- Comparison with Sanity integration
- Comparison with Payload integration
- Decision matrix and scoring
- Migration considerations

### ❓ [FAQ](./FAQ.md)
**Common questions** - Quick answers to frequent questions.

- General questions
- Technical questions
- Troubleshooting
- Performance and scaling
- Legacy version concerns
- Migration paths

### 📝 [SUMMARY](./SUMMARY.md)
**Quick reference** - Cheat sheet for developers.

- Architecture diagram
- Data flow summary
- API endpoints
- Key files and their purposes
- Common commands
- Quick troubleshooting

## Quick Start

### Prerequisites

- Node.js v16+
- PostgreSQL (for both Medusa and Strapi)
- Redis (for event bus and caching)
- Medusa v1.8+ backend
- Basic understanding of Medusa and Strapi

### 5-Minute Setup Overview

**⚠️ Important**: Choose which repository to clone based on your needs. See [Package Versions Guide](./PACKAGE-VERSIONS.md).

#### Option A: Original (Simple Setup)

```bash
# 1. Clone original Strapi template
git clone https://github.com/SGFGOV/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi
cp .env.test .env
# Configure environment variables (see detailed guide)

# 2. Build Strapi packages
yarn install
yarn build

# 3. Start Strapi server
yarn develop  # Port 1337

# 4. Install Medusa plugin (from npm)
cd /path/to/medusa-backend
yarn add medusa-plugin-strapi-ts

# 5. Configure Medusa plugin
# Add to medusa-config.js (see implementation guide)
# Set environment variables

# 6. Start Medusa
npx medusa develop  # Port 9000

# 7. Test integration
# Create product in Medusa → Check Strapi Admin at http://localhost:1337/admin
```

#### Option B: Fork (Enhanced Setup)

```bash
# 1. Clone fork Strapi template (with extra features)
git clone https://github.com/dannycdannydo/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi
cp .env.test .env
# Configure environment variables (see detailed guide)

# 2. Build Strapi packages
yarn install
yarn build

# 3. Start Strapi server
yarn dev  # Port 1337 (note: different script name)

# 4. Install Medusa plugin (from npm - works with both)
cd /path/to/medusa-backend
yarn add medusa-plugin-strapi-ts

# 5-7. Same as Option A
```

**For detailed instructions**: See [Implementation Guide](./04-implementation.md)

## 🚨 Important: Repository & Package Clarification

**CRITICAL**: Before proceeding, understand which packages you can install from npm vs which must be cloned locally.

### Repository Information

- **Original**: [SGFGOV/medusa-strapi-repo](https://github.com/SGFGOV/medusa-strapi-repo) - 719 commits
- **Fork**: [dannycdannydo/medusa-strapi-repo](https://github.com/dannycdannydo/medusa-strapi-repo) - 764 commits (+45 additional commits)

### Package Installation Guide

| Package | Can Use npm? | Notes |
|---------|--------------|-------|
| `medusa-plugin-strapi-ts` | ✅ **YES** | Identical in both repos, use `yarn add medusa-plugin-strapi-ts` |
| `medusa-strapi` (Strapi app) | ❌ **NO** | Must clone repository, not published to npm |
| `strapi-plugin-medusajs` | ✅ **YES** | Can use npm or included in monorepo |
| `strapi-plugin-sso-medusa` | ✅ **YES** | Can use npm or included in monorepo |

**See**: **[Complete Package Installation Guide](./PACKAGE-VERSIONS.md)** for detailed instructions on npm vs local installation.

## Production Examples & Enhanced Fork

While this documentation references the official [SGFGOV/medusa-strapi-repo](https://github.com/SGFGOV/medusa-strapi-repo), 
a **production-ready fork with 45+ additional commits** and enhanced features is available at [dannycdannydo/medusa-strapi-repo](https://github.com/dannycdannydo/medusa-strapi-repo).

### Additional Features in Production Fork

**Upload & Email Integrations**:
- ✅ **Cloudinary** image hosting (alternative to AWS S3)
- ✅ **Mailchimp** marketing automation integration
- ✅ **SendGrid** transactional email plugin

**Content Management Extensions**:
- ✅ Custom homepage content types (banners, features, offers)
- ✅ Policy & legal page management (terms, privacy)
- ✅ Team member profile pages
- ✅ Website maintenance mode toggle
- ✅ Enhanced image management types

**Technical Enhancements**:
- ✅ TypeScript environment-specific configurations (dev/prod)
- ✅ Updated PostgreSQL driver (8.12.0)
- ✅ Production-tested with real project ("Marshmallow")
- ✅ 14 additional custom content types

### When to Use Which Version

**Use Original Repository** (`SGFGOV/medusa-strapi-repo`) **if**:
- 🎓 You're learning the Medusa-Strapi integration
- 🚀 You want the minimal, clean setup
- 📦 You prefer installing plugins as needed
- 📖 You're following official documentation

**Use Production Fork** (`medusa-with-strapi-dannycdannydo`) **if**:
- 🏭 You need production-ready examples
- 🎨 You want custom homepage/marketing content types
- 🔌 You need multiple upload providers (S3 + Cloudinary)
- 📧 You want email marketing integration (Mailchimp)
- 💼 You're building a content-heavy e-commerce site
- 🛠️ You want TypeScript environment configs

**See**: [Production Examples & Advanced Features](./ANALYSIS.md) for detailed comparison and implementation guides.

## Visual Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────┐
│                  MASTER-SLAVE ARCHITECTURE               │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────┐              ┌─────────────────┐ │
│  │   MEDUSA v1      │              │   STRAPI v4     │ │
│  │   (Master)       │──────sync───►│   (Slave)       │ │
│  │                  │              │                 │ │
│  │  • Products      │              │  • Product CMS  │ │
│  │  • Variants      │  Limited     │  • Rich Content │ │
│  │  • Pricing       │  ◄─sync back─│  • Editorial    │ │
│  │  • Inventory     │              │  • Content Mgmt │ │
│  │  • Orders        │              │                 │ │
│  │  • Collections   │              │  PostgreSQL     │ │
│  └────────┬─────────┘              └────────┬────────┘ │
│           │                                  │          │
│           │         ┌──────────────┐        │          │
│           └────────►│  STOREFRONT  │◄───────┘          │
│                     │  (Next.js)   │                   │
│                     │              │                   │
│                     │  Fetches:    │                   │
│                     │  • Commerce  │                   │
│                     │  • Content   │                   │
│                     └──────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

### Data Synchronization Flow

```
┌──────────────────────────────────────────────────────────┐
│  1. Create Product in Medusa Admin                       │
│     ↓                                                     │
│  2. Event: "product.created" (via Event Bus)             │
│     ↓                                                     │
│  3. Strapi Subscriber catches event                      │
│     ↓                                                     │
│  4. Query product data from Medusa                       │
│     - Variants, options, prices                          │
│     - Collections, categories, tags                      │
│     - Images, metadata                                   │
│     ↓                                                     │
│  5. Transform to Strapi format                           │
│     - Rename fields (collection → product_collection)    │
│     - Convert IDs to medusa_id references                │
│     - Structure relations                                │
│     ↓                                                     │
│  6. Authenticate with Strapi                             │
│     - Service account JWT token                          │
│     ↓                                                     │
│  7. HTTP POST to Strapi API                              │
│     - /api/products                                      │
│     ↓                                                     │
│  8. Strapi stores in PostgreSQL ✓                        │
│     - Creates product with medusa_id                     │
│     - Links variants, collections, categories            │
│                                                           │
│  9. Content team enriches in Strapi Admin                │
│     - Add rich descriptions                              │
│     - Upload high-res images                             │
│     - Create marketing content                           │
│     - Organize related products                          │
│     ↓                                                     │
│  10. Limited sync back to Medusa                         │
│      - Only basic fields (title, subtitle)               │
│      - Most content stays in Strapi                      │
│                                                           │
│  11. Storefront fetches:                                 │
│      a) Product data from Medusa API                     │
│      b) Content from Strapi (via proxy or direct)        │
│      ↓                                                    │
│  12. Merge and display to customer ✓                     │
└──────────────────────────────────────────────────────────┘
```

## Key Concepts

### What Syncs Automatically (Medusa → Strapi)

| Event | Direction | What | When |
|-------|-----------|------|------|
| Product Created | Medusa → Strapi | Full product structure (variants, options, collections, categories, tags) | Immediately on creation |
| Product Updated | Medusa → Strapi | Updated product data | Immediately on update |
| Product Deleted | Medusa → Strapi | Delete command | Immediately on deletion |
| Variant Created | Medusa → Strapi | New variant with prices, options | Immediately |
| Variant Updated | Medusa → Strapi | Updated variant data | Immediately |
| Collection Created/Updated | Medusa → Strapi | Collection data and product relationships | Immediately |
| Category Created/Updated | Medusa → Strapi | Category data and hierarchy | Immediately |
| Region Created/Updated | Medusa → Strapi | Region with payment/fulfillment providers | Immediately |
| Metafields Created/Updated | Medusa → Strapi | Product metadata | Immediately |

### What Syncs Back (Strapi → Medusa) - LIMITED

| Field | Direction | What | Why Limited |
|-------|-----------|------|-------------|
| Product Title | Strapi → Medusa | Title changes | Basic field sync |
| Product Subtitle | Strapi → Medusa | Subtitle changes | Basic field sync |
| Variant Title | Strapi → Medusa | Variant title changes | Basic field sync |
| Region Name | Strapi → Medusa | Region name changes | Basic field sync |
| **Rich Content** | **Strapi ONLY** | **Descriptions, images, editorial content** | **Content managed independently** |
| **Media Assets** | **Strapi ONLY** | **Uploaded images, videos** | **CMS feature** |
| **Custom Fields** | **Strapi ONLY** | **Extra content fields** | **CMS extension** |

### What Doesn't Sync

| Change | System | Why |
|--------|--------|-----|
| Rich descriptions | Strapi only | Content management feature |
| Marketing copy | Strapi only | Editorial content |
| Media library | Strapi only | Asset management |
| Prices | Medusa only | Commerce operation |
| Inventory | Medusa only | Real-time stock levels |
| Orders | Medusa only | Transaction data |
| Customers | Medusa only | User data |

### Storefront Data Sources

When a customer views a product page:

| Element | Source | Fetch Method | Why |
|---------|--------|--------------|-----|
| Product ID | Medusa | Medusa API | Commerce identifier |
| Title | Medusa (or Strapi if synced) | Medusa API | Primary title |
| **Rich Description** | **Strapi** | **Strapi API or Proxy** | **Editorial content** |
| **Hero Images** | **Strapi** | **Strapi API or Proxy** | **CMS media library** |
| **Marketing Copy** | **Strapi** | **Strapi API or Proxy** | **Content management** |
| Price | Medusa | Medusa API | Dynamic, region-specific |
| Inventory | Medusa | Medusa API | Real-time |
| Variants | Medusa | Medusa API | Commerce structure |
| Collections | Medusa | Medusa API | Catalog organization |
| Add to Cart | Medusa | Medusa API | Commerce operation |

**Key Difference**: Storefront makes **TWO API calls** - one to Medusa for commerce, one to Strapi for content (or uses Medusa proxy endpoint `/strapi/content/`).

## Common Use Cases

### ✅ Perfect For

1. **Editorial E-commerce**
   - Rich product stories and narratives
   - Magazine-style product content
   - Brand storytelling
   - Content-first commerce

2. **Complex Product Information**
   - Detailed specifications
   - Multiple content sections
   - Technical documentation
   - User guides and manuals

3. **Content Team Workflow**
   - Non-technical content managers
   - Approval workflows
   - Scheduled publishing
   - Content versioning (with plugins)

4. **FOSS Requirement**
   - No vendor lock-in
   - Self-hosted solution
   - Full source code access
   - Community-driven development

5. **Medusa v1 Legacy Systems**
   - Existing Medusa v1 deployments
   - Not ready to migrate to v2
   - Stable, proven setup

### ❌ Not Ideal For

1. **New Medusa v2 Projects**
   - Built for legacy Medusa v1
   - Newer integrations available (Sanity, Payload)
   - Migration overhead

2. **Simple Catalogs**
   - Basic products with minimal content
   - Small product count
   - Simple descriptions sufficient

3. **Fast Time-to-Market**
   - Two systems to set up and configure
   - Additional infrastructure
   - Learning curve for two platforms

4. **Limited DevOps Resources**
   - Need to maintain Strapi server
   - PostgreSQL database management
   - Double the deployment complexity

5. **Frequently Changing Content Models**
   - Strapi schema changes require manual migration
   - Field mappings need updating
   - Sync logic may need adjustment

## Comparison with Other CMS Options

**Looking to compare Strapi with other CMS options?**

See the **[Complete CMS Comparison Guide](../README.md)** for a detailed comparison of Strapi, Payload, Sanity, and Contentful, including:
- Feature comparison tables
- Cost analysis
- Use case recommendations
- Migration paths
- Decision matrix

**Quick comparison:** Strapi is the go-to FOSS solution for Medusa v1 with comprehensive CMS features, but requires significant DevOps resources. For new Medusa v2 projects, consider Payload or Sanity instead.

**⚠️ Important:** This integration is built for Medusa v1. Migrating to Medusa v2 requires significant effort. See the [comparison guide](../README.md) for modern alternatives on v2.

## Troubleshooting Quick Reference

### Product not syncing to Strapi?

1. Check Medusa logs for event bus errors
2. Verify Redis is running (required for event bus)
3. Check `STRAPI_SECRET` matches between Medusa and Strapi
4. Ensure Strapi service account is created
5. Test Strapi health: `curl http://localhost:1337/_health`
6. Check subscriber is loaded: Look for "Strapi Subscriber Initialized" in Medusa logs

### Strapi content not showing on storefront?

1. Verify product exists in Strapi (`/admin/content-manager/collectionType/api::product.product`)
2. Check storefront is calling Strapi API or Medusa proxy endpoint
3. Verify CORS settings in Strapi config
4. Check authentication tokens
5. Inspect Network tab for API errors

### Strapi server won't start?

1. Verify PostgreSQL database is accessible
2. Check all required environment variables are set
3. Ensure `APP_KEYS`, `API_TOKEN_SALT`, `JWT_SECRET`, `ADMIN_JWT_SECRET` are set
4. Run migrations: `yarn strapi admin:reset-user-password` (creates admin)
5. Check port 1337 is not in use

### Authentication failing?

1. Verify `MEDUSA_STRAPI_SECRET` matches `STRAPI_SECRET`
2. Check super admin credentials are correct
3. Ensure service account user is created in Strapi
4. Check JWT token expiration settings
5. Review authentication logs in both systems

**For detailed troubleshooting**: See [Implementation Guide - Troubleshooting](./04-implementation.md#troubleshooting)

## Support & Resources

### Official Documentation

- [Medusa v1 Documentation](https://docs.medusajs.com/v1/)
- [Strapi v4 Documentation](https://docs.strapi.io/dev-docs/intro)
- [Medusa Strapi Plugin (Community)](https://docs.medusajs.com/v1/plugins/cms/strapi)
- [GitHub Repository (Original)](https://github.com/SGFGOV/medusa-strapi-repo) - 719 commits
- [GitHub Repository (Fork)](https://github.com/dannycdannydo/medusa-strapi-repo) - 764 commits (+45 additional)
- [Package Installation Guide](./PACKAGE-VERSIONS.md) - **Critical**: npm vs local installation
- [npm: medusa-plugin-strapi-ts](https://www.npmjs.com/package/medusa-plugin-strapi-ts)

### Community

- [Medusa Discord](https://discord.gg/medusajs) - #plugins channel
- [Strapi Discord](https://discord.strapi.io)
- [GitHub Issues](https://github.com/SGFGOV/medusa-strapi-repo/issues)
- Plugin Maintainer: @govdiw on Discord

### Learn More

- [Medusa v1 Events Guide](https://docs.medusajs.com/development/events/events-list)
- [Strapi Content-Type Builder](https://docs.strapi.io/dev-docs/backend-customization/models)
- [Strapi API Documentation](https://docs.strapi.io/dev-docs/api/rest)
- [Monorepo Organization (Lerna)](https://lerna.js.org/)

## Real-World Considerations

### Performance

**Pros:**
- Direct PostgreSQL queries (fast)
- Cacheable content via Strapi
- Separated read concerns (commerce vs content)

**Cons:**
- Two API calls per product page (if not using proxy)
- Additional latency from second service
- Database connections for both systems

**Optimization tips**: Use Medusa proxy endpoint, implement Redis caching, CDN for Strapi media, consider GraphQL layer.

### Security

- Store API keys securely (environment variables)
- Use different PostgreSQL databases for Medusa and Strapi
- Implement proper CORS settings
- Rotate JWT secrets regularly
- Use read-only tokens for public frontends
- Keep both systems updated (security patches)

### Scalability

- Both Medusa and Strapi can scale horizontally
- PostgreSQL can be clustered
- Redis can be clustered for event bus
- Consider managed PostgreSQL for production
- Load balance Strapi instances if needed

## Migration Considerations

### From Medusa v1 Strapi to Medusa v2

**Current State**: This integration is built for Medusa v1. Migrating to Medusa v2 requires:

1. **Major Code Changes**:
   - Medusa v2 has completely different architecture
   - Event system changed
   - Service layer redesigned
   - Plugin structure updated

2. **Options**:
   - **Stay on v1**: Continue using this integration (stable)
   - **Migrate to Sanity**: Modern v2 integration available
   - **Migrate to Payload**: Modern v2 integration available
   - **Custom Development**: Port this plugin to v2 (significant work)

3. **Recommendation**: For new projects, use Sanity or Payload with Medusa v2.

## Contributing

Found an issue or want to improve this integration?

1. Check [GitHub repository](https://github.com/SGFGOV/medusa-strapi-repo)
2. Review existing issues and PRs
3. Test changes thoroughly (integration tests included)
4. Update documentation to reflect changes
5. Submit pull request with clear description

**Financial Support**: Consider [sponsoring the maintainer](https://github.com/sponsors/SGFGOV) to ensure continued development.

## License

This integration is open source (MIT License) and provided as-is for reference and production use.

---

**Ready to start?** → Begin with [Overview](./01-overview.md) or jump to [Implementation Guide](./04-implementation.md)

**Need help deciding?** → See [Pros & Cons](./05-pros-cons.md) for detailed comparison

**Have questions?** → Check [FAQ](./FAQ.md) for common answers

