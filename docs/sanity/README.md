# Sanity CMS Integration with Medusa

**Complete documentation for integrating Sanity CMS with Medusa v2 for headless e-commerce content management.**

## Quick Overview

This integration combines:
- **Medusa** - Headless e-commerce platform (products, pricing, orders, inventory)
- **Sanity** - Headless CMS platform (product content, localization, addons)
- **Next.js** - Storefront that fetches data from both systems

### Why This Architecture?

Traditional e-commerce stores all content in one database. This integration **separates commerce from content**, giving you:

✅ Powerful content modeling and localization  
✅ Real-time collaboration in Sanity Studio  
✅ Structured content with GROQ queries  
✅ Content versioning and publishing workflows  
✅ Multi-language product specifications  

### The Trade-off

While powerful, this adds complexity:

❌ Two separate systems to manage  
❌ Additional infrastructure (Sanity project required)  
❌ Data synchronization to maintain  
❌ Dual data fetching in storefront  

**Use this integration if**: You need advanced content management, localization, or structured content beyond basic product descriptions.

**Skip this integration if**: You have simple products with minimal content needs or want to minimize infrastructure complexity.

## Documentation Structure

### 📚 [1. Overview](./01-overview.md)
**Start here** - Understand what this integration does, why it exists, and key benefits.

- What is Sanity integration?
- Why use Sanity with Medusa?
- What gets synced (and what doesn't)
- Technology stack overview
- Quick start summary

### 🏗️ [2. Architecture Deep Dive](./02-architecture.md)
**Technical details** - How the integration works under the hood.

- System components explained
- Medusa Sanity Module
- Event subscribers and workflows
- Sanity schema and documents
- Link definitions
- Data fetching patterns

### 🔄 [3. Data Flow](./03-data-flow.md)
**See it in action** - Step-by-step walkthroughs of common scenarios.

- Creating a new product (Medusa → Sanity)
- Enriching content in Sanity Studio
- Customer viewing product (dual fetch)
- Product updates and syncing
- Product deletion
- Bulk sync operations
- Manual sync from admin

### 🛠️ [4. Implementation Guide](./04-implementation.md)
**Build it yourself** - Step-by-step setup instructions.

- Prerequisites and requirements
- Part 1: Create Sanity project
- Part 2: Set up Medusa backend
- Part 3: Set up Next.js + Sanity Studio
- Part 4: Connect the systems
- Part 5: Test the integration
- Troubleshooting common issues
- Production deployment guide

### ⚖️ [5. Pros & Cons](./05-pros-cons.md)
**Make the decision** - Detailed analysis of trade-offs.

- When to use this integration
- When NOT to use it
- Detailed pros (content management, localization, collaboration)
- Detailed cons (complexity, costs, dual fetching)
- Comparison with Payload integration
- Decision matrix and scoring
- Migration paths

### ❓ [FAQ](./FAQ.md)
**Common questions** - Quick answers to frequent questions.

- General questions
- Technical questions
- Troubleshooting
- Performance and scaling
- Cost considerations

### 📝 [Summary](./SUMMARY.md)
**Quick reference** - Cheat sheet for developers.

- Architecture diagram
- Data flow summary
- API endpoints
- Key files and their purposes
- Common commands
- Quick troubleshooting

## Quick Start

### Prerequisites

- Node.js v20+
- PostgreSQL
- Sanity account (free tier available)
- Basic understanding of Medusa and Next.js

### 5-Minute Setup Overview

```bash
# 1. Create Sanity project
npm create sanity@latest
# Follow prompts, note your PROJECT_ID

# 2. Generate Sanity API token
# Visit manage.sanity.io → Your Project → API → Tokens
# Create token with "Editor" permissions

# 3. Set up Medusa with Sanity module
cd medusa
npm install @sanity/client
# Add Sanity module to medusa-config.ts
# Set SANITY_API_TOKEN and SANITY_PROJECT_ID in .env
npx medusa db:setup
npm run dev  # Port 9000

# 4. Set up Next.js + Sanity Studio
cd storefront
npm install sanity next-sanity
# Configure sanity.config.ts and schema
npm run dev  # Port 8000

# 5. Test integration
# Create product in Medusa → Appears in Sanity
# Edit in Sanity Studio (localhost:8000/studio)
# View on storefront → Shows Sanity content
```

**For detailed instructions**: See [Implementation Guide](./04-implementation.md)

## Visual Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────┐
│                    HYBRID ARCHITECTURE                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────┐              ┌─────────────────┐ │
│  │   MEDUSA         │              │   SANITY CMS    │ │
│  │   (Commerce)     │──────sync───►│   (Content)     │ │
│  │                  │              │                 │ │
│  │  • Products      │              │  • Specs        │ │
│  │  • Variants      │              │  • Localization │ │
│  │  • Pricing       │              │  • Addons       │ │
│  │  • Inventory     │              │  • References   │ │
│  │  • Orders        │              │                 │ │
│  └────────┬─────────┘              └────────┬────────┘ │
│           │                                  │          │
│           │         ┌──────────────┐        │          │
│           └────────►│  STOREFRONT  │◄───────┘          │
│                     │  (Next.js)   │                   │
│                     │              │                   │
│                     │  Dual Fetch: │                   │
│                     │  • Medusa    │                   │
│                     │  • Sanity    │                   │
│                     └──────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

### Data Synchronization Flow

```
┌──────────────────────────────────────────────────────────┐
│  1. Create Product in Medusa Admin                       │
│     ↓                                                     │
│  2. Event: "product.created" or "product.updated"        │
│     ↓                                                     │
│  3. Subscriber triggers workflow                         │
│     ↓                                                     │
│  4. Workflow queries product data                        │
│     ↓                                                     │
│  5. Transform to Sanity document format                  │
│     • _id: product.id                                    │
│     • _type: "product"                                   │
│     • title: product.title                               │
│     • specs: [{ lang, title, content }]                  │
│     ↓                                                     │
│  6. Upsert to Sanity via @sanity/client                  │
│     ↓                                                     │
│  7. Sanity stores document ✓                             │
│                                                           │
│  8. Content team edits in Sanity Studio                  │
│     • Add localized specs (EN, ES, FR, etc.)            │
│     • Write rich content                                 │
│     • Link related products as addons                    │
│     (NO sync back to Medusa)                             │
│                                                           │
│  9. Storefront fetches:                                  │
│     a) Product data from Medusa API                      │
│     b) Content specs from Sanity API                     │
│     ↓                                                     │
│  10. Merge and display to customer ✓                     │
└──────────────────────────────────────────────────────────┘
```

## Key Concepts

### What Syncs Automatically (Medusa → Sanity)

| Event | Direction | What | When |
|-------|-----------|------|------|
| Product Created | Medusa → Sanity | Basic product info (id, title, empty specs) | Immediately on creation |
| Product Updated | Medusa → Sanity | Updated title | Immediately on update |

### What Doesn't Sync

| Change | System | Why |
|--------|--------|-----|
| Content updates | Sanity only | Content managed independently |
| Localized specs | Sanity only | Localization feature |
| Product addons | Sanity only | Content relationship |
| Prices | Medusa only | Commerce operation |
| Inventory | Medusa only | Real-time stock levels |
| Variants | Medusa only | Commerce structure |
| Collections | Medusa only | Catalog organization |

### Storefront Data Sources

When a customer views a product page:

| Element | Source | Fetch Method | Why |
|---------|--------|--------------|-----|
| Product ID | Medusa | Medusa API | Commerce identifier |
| Title | Medusa | Medusa API | Primary title |
| **Content/Specs** | **Sanity** | **Sanity API** | Localized content |
| **Addons** | **Sanity** | **Sanity API** | Content relationships |
| Price | Medusa | Medusa API | Dynamic, region-specific |
| Inventory | Medusa | Medusa API | Real-time |
| Variants | Medusa | Medusa API | Commerce structure |
| Add to Cart | Medusa | Medusa API | Commerce operation |

**Key Difference**: Storefront makes **TWO separate API calls** - one to Medusa, one to Sanity.

## Common Use Cases

### ✅ Perfect For

1. **Multi-Language E-commerce**
   - Product specs in multiple languages
   - Localized content by region
   - Language-specific descriptions
   - Cultural adaptation

2. **Content-Rich Products**
   - Detailed specifications
   - Structured product information
   - Rich editorial content
   - Complex product stories

3. **Product Relationships**
   - Cross-sell via addons
   - Related product suggestions
   - Product bundles
   - Accessory recommendations

4. **Content Team Collaboration**
   - Real-time multi-user editing
   - Content workflows
   - Version control
   - Preview before publish

### ❌ Not Ideal For

1. **Simple Catalogs**
   - Basic products with short descriptions
   - Single-language stores
   - Minimal content needs

2. **High-Frequency Updates**
   - Products with constantly changing descriptions
   - Real-time content requirements
   - Immediate synchronization needs

3. **Budget-Conscious Projects**
   - Tight infrastructure budget
   - Avoiding external dependencies
   - Minimizing API calls

## Comparison: Sanity vs Payload Integration

| Feature | Sanity | Payload |
|---------|--------|---------|
| **Deployment** | External SaaS | Embedded in Next.js |
| **Data Storage** | Sanity Cloud | Your PostgreSQL |
| **Admin UI** | Sanity Studio | Payload Admin |
| **Pricing** | Free tier + usage-based | Self-hosted (free) |
| **Content Model** | Lightweight (specs, addons) | Comprehensive (images, SEO, rich content) |
| **Localization** | Built-in support | Custom implementation |
| **Data Fetching** | Dual fetch (Medusa + Sanity) | Single fetch via Medusa link |
| **Real-time Collab** | Yes | No |
| **Setup Complexity** | Medium (external account) | Medium (embedded) |
| **Vendor Lock-in** | Yes (Sanity) | No (self-hosted) |

**Choose Sanity if**: You need localization, real-time collaboration, or prefer managed CMS.  
**Choose Payload if**: You want full control, integrated data fetching, or self-hosting.

## Troubleshooting Quick Reference

### Product not syncing to Sanity?

1. Check Medusa logs for errors
2. Verify `SANITY_API_TOKEN` has Editor permissions
3. Ensure `SANITY_PROJECT_ID` is correct
4. Test Sanity connection: `curl https://YOUR_PROJECT_ID.api.sanity.io/v2024-01-01/data/query/production`
5. Check subscriber is loaded: Look for "Connected to Sanity" in logs

### Sanity content not showing on storefront?

1. Verify product exists in Sanity (check Studio at `/studio`)
2. Check storefront is calling `client.getDocument(productId)`
3. Inspect Network tab for Sanity API call
4. Verify `NEXT_PUBLIC_SANITY_PROJECT_ID` is set
5. Check CORS settings in Sanity project

### Sanity Studio not loading?

1. Verify `sanity.config.ts` exists with correct projectId
2. Check `/studio` route exists in Next.js app
3. Ensure `sanity` package is installed
4. Clear Next.js cache: `rm -rf .next && npm run dev`
5. Check browser console for errors

**For detailed troubleshooting**: See [Implementation Guide - Troubleshooting](./04-implementation.md#troubleshooting)

## Support & Resources

### Official Documentation

- [Medusa Documentation](https://docs.medusajs.com)
- [Sanity Documentation](https://www.sanity.io/docs)
- [Next.js Documentation](https://nextjs.org/docs)
- [Sanity + Next.js Guide](https://www.sanity.io/docs/nextjs)

### Community

- [Medusa Discord](https://discord.gg/medusajs)
- [Sanity Slack](https://slack.sanity.io)
- [GitHub Issues](https://github.com/medusajs/medusa)

### Learn More

- [Medusa Modules Guide](https://docs.medusajs.com/learn/basics/modules)
- [Medusa Workflows Guide](https://docs.medusajs.com/learn/basics/workflows)
- [Sanity Content Lake](https://www.sanity.io/docs/datastore)
- [GROQ Query Language](https://www.sanity.io/docs/groq)
- [Sanity Studio](https://www.sanity.io/docs/sanity-studio)

## Real-World Examples

This integration pattern is used by:

- **Multi-region e-commerce** requiring localized product content
- **Fashion brands** with seasonal collections and editorial content
- **Technical products** with detailed, structured specifications
- **International stores** serving multiple languages

## Performance Considerations

### Pros
- Content delivery via Sanity's CDN (fast global access)
- Efficient GROQ queries for complex content
- Real-time updates without Medusa restart

### Cons
- Two API calls per product page (Medusa + Sanity)
- Additional latency from external service
- API usage limits on Sanity free tier

**Optimization tips**: Use caching, batch requests, and CDN for images.

## Security Notes

- Store Sanity API token securely (never in frontend)
- Use read-only tokens for public frontends
- Implement proper CORS settings
- Monitor API usage and set alerts

## Contributing

Found an issue or want to improve this documentation?

1. Check existing examples in `https://github.com/medusajs/examples/tree/main/sanity-integration`
2. Test changes thoroughly
3. Update documentation to reflect changes
4. Submit pull request with clear description

## License

This integration example is provided as-is for reference and learning purposes.

---

**Ready to start?** → Begin with [Overview](./01-overview.md) or jump to [Implementation Guide](./04-implementation.md)

**Need help deciding?** → See [Pros & Cons](./05-pros-cons.md) for detailed comparison

