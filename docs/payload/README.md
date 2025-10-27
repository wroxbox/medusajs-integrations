# Payload CMS Integration with Medusa

**Complete documentation for integrating Payload CMS with Medusa v2 for headless e-commerce content management.**

## Quick Overview

This integration combines:
- **Medusa** - Headless e-commerce platform (products, pricing, orders, inventory)
- **Payload CMS** - Modern content management system (descriptions, images, SEO)
- **Next.js** - Storefront that fetches data from both systems

### Why This Architecture?

Traditional e-commerce stores everything in one database. This integration **separates commerce from content**, giving you:

✅ Rich content editor for product descriptions  
✅ Professional media management  
✅ SEO optimization tools  
✅ Content team independence from developers  
✅ Scalable architecture  

### The Trade-off

While powerful, this adds complexity:

❌ Two systems to manage  
❌ Higher infrastructure costs  
❌ Steeper learning curve  

**Use this integration if**: Content quality is a competitive advantage and you have a dedicated content team.

**Skip this integration if**: You have simple products, small team, or tight budget/timeline.

## Documentation Structure

### 📚 [1. Overview](./01-overview.md)
**Start here** - Understand what this integration does, why it exists, and key benefits.

- What is Payload integration?
- Why use hybrid CMS + e-commerce?
- What gets synced (and what doesn't)
- Technology stack overview
- Quick start summary

### 🏗️ [2. Architecture Deep Dive](./02-architecture.md)
**Technical details** - How the integration works under the hood.

- System components explained
- Medusa Payload Module
- Event subscribers and workflows
- Payload collections and access control
- API communication patterns
- Database architecture

### 🔄 [3. Data Flow](./03-data-flow.md)
**See it in action** - Step-by-step walkthroughs of common scenarios.

- Creating a new product (Medusa → Payload)
- Enriching content in Payload
- Customer viewing product (reads from both)
- Variant updates and syncing
- Product deletion
- Bulk sync operations

### 🛠️ [4. Implementation Guide](./04-implementation.md)
**Build it yourself** - Step-by-step setup instructions.

- Prerequisites and requirements
- Part 1: Set up Medusa backend
- Part 2: Set up Next.js + Payload
- Part 3: Connect the systems
- Part 4: Integrate storefront
- Troubleshooting common issues
- Production deployment guide

### ⚖️ [5. Pros & Cons](./05-pros-cons.md)
**Make the decision** - Detailed analysis of trade-offs.

- When to use this integration
- When NOT to use it
- Detailed pros (content management, productivity, scalability)
- Detailed cons (complexity, costs, learning curve)
- Decision matrix and scoring
- Migration paths

## Quick Start

### Prerequisites

- Node.js v20+
- PostgreSQL
- Basic understanding of Medusa and Next.js

### 5-Minute Setup Overview

```bash
# 1. Set up Medusa with Payload module
cd medusa
npm install
# Add Payload module, workflows, subscribers
npx medusa db:setup
npm run dev  # Port 9000

# 2. Set up Next.js + Payload
cd storefront
npm install
# Add Payload config and collections
npm run dev  # Port 8000

# 3. Generate Payload API key
# Visit http://localhost:8000/admin
# Create user → Enable API Key

# 4. Configure Medusa with API key
# Update .env: PAYLOAD_API_KEY=your-key
# Restart Medusa

# 5. Test integration
# Create product in Medusa → Appears in Payload
# Edit in Payload → Shows on storefront
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
│  │   MEDUSA         │              │   PAYLOAD CMS   │ │
│  │   (Commerce)     │◄────sync────►│   (Content)     │ │
│  │                  │              │                 │ │
│  │  • Products      │              │  • Descriptions │ │
│  │  • Variants      │              │  • Images       │ │
│  │  • Pricing       │              │  • SEO          │ │
│  │  • Inventory     │              │  • Rich Content │ │
│  │  • Orders        │              │                 │ │
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

### Data Flow

```
┌──────────────────────────────────────────────────────────┐
│  1. Create Product in Medusa                             │
│     ↓                                                     │
│  2. Event: "product.created"                             │
│     ↓                                                     │
│  3. Subscriber triggers workflow                         │
│     ↓                                                     │
│  4. Query product data from Medusa                       │
│     ↓                                                     │
│  5. Transform to Payload format                          │
│     ↓                                                     │
│  6. HTTP POST to Payload API                             │
│     ↓                                                     │
│  7. Payload stores product                               │
│     ↓                                                     │
│  8. Return Payload ID                                    │
│     ↓                                                     │
│  9. Store Payload ID in Medusa metadata                  │
│     ↓                                                     │
│  10. Link established ✓                                  │
│                                                           │
│  11. Content Manager enriches in Payload                 │
│     • Rich descriptions                                  │
│     • Upload images                                      │
│     • Add SEO metadata                                   │
│     (NO sync back to Medusa)                             │
│                                                           │
│  12. Storefront fetches from Medusa with                 │
│      fields=*payload_product                             │
│     ↓                                                     │
│  13. Medusa returns merged data:                         │
│     • Commerce data from Medusa                          │
│     • Content data from Payload (via link)               │
│     ↓                                                     │
│  14. Customer sees rich product page ✓                   │
└──────────────────────────────────────────────────────────┘
```

## Key Concepts

### What Syncs Automatically

| Event | Direction | What | When |
|-------|-----------|------|------|
| Product Created | Medusa → Payload | Basic product info, structure | Immediately |
| Product Deleted | Medusa → Payload | Delete command | Immediately |
| Variant Created | Medusa → Payload | New variant details | Immediately |
| Variant Updated | Medusa → Payload | Updated variant | Immediately |
| Option Created | Medusa → Payload | New option | Immediately |

### What Doesn't Sync

| Change | System | Why |
|--------|--------|-----|
| Content updates | Payload only | Medusa is source of truth for commerce |
| Images | Payload only | Media library in Payload |
| SEO | Payload only | Content management feature |
| Prices | Medusa only | Commerce operation |
| Inventory | Medusa only | Real-time stock levels |

### Storefront Data Sources

When a customer views a product:

| Element | Source | Why |
|---------|--------|-----|
| Title | Payload (fallback: Medusa) | Content team can optimize |
| Description | Payload | Rich formatting |
| Images | Payload | Managed media |
| SEO | Payload | Optimization |
| **Price** | **Medusa** | Dynamic, region-specific |
| **Inventory** | **Medusa** | Real-time |
| **Add to Cart** | **Medusa** | Commerce operation |

## Common Use Cases

### ✅ Perfect For

1. **Fashion & Apparel**
   - Detailed product descriptions
   - Multiple lifestyle images
   - Size guides and care instructions
   - Seasonal collections

2. **Home Goods**
   - Rich styling content
   - Room inspiration images
   - Material specifications
   - Design stories

3. **Electronics**
   - Technical specifications
   - Multiple product angles
   - Feature comparison tables
   - User guides

4. **Artisanal Products**
   - Maker stories
   - Process photos
   - Material sourcing
   - Craftsmanship details

### ❌ Not Ideal For

1. **Commodity Products**
   - Simple items (screws, basics)
   - Minimal descriptions
   - Auto-generated specs

2. **MVP/Small Catalogs**
   - < 50 products
   - Quick launch needed
   - Limited resources

3. **B2B Catalogs**
   - Technical specs only
   - PIM system better fit
   - Bulk operations focus

## Troubleshooting Quick Reference

### Product not syncing to Payload?

1. Check Medusa logs for errors
2. Verify `PAYLOAD_API_KEY` is correct
3. Ensure Payload is running at `PAYLOAD_SERVER_URL`
4. Test API key: `curl http://localhost:8000/api/products -H "Authorization: users API-Key YOUR_KEY"`

### Payload data not showing on storefront?

1. Verify query includes `*payload_product` in fields
2. Check product metadata has `payload_id`
3. Ensure link definition is loaded (restart Medusa)
4. Test: `curl "http://localhost:9000/store/products/handle?fields=*payload_product"`

### Images not loading?

1. Check Next.js image configuration allows localhost
2. Verify image URLs are correct
3. Check CORS configuration
4. Test: `curl http://localhost:8000/media/image.jpg`

**For detailed troubleshooting**: See [Implementation Guide - Troubleshooting](./04-implementation.md#troubleshooting)

## Support & Resources

### Official Documentation

- [Medusa Documentation](https://docs.medusajs.com)
- [Payload Documentation](https://payloadcms.com/docs)
- [Next.js Documentation](https://nextjs.org/docs)

### Community

- [Medusa Discord](https://discord.gg/medusajs)
- [Payload Discord](https://discord.gg/payload)
- [GitHub Issues](https://github.com/medusajs/medusa)

### Learn More

- [Medusa Modules Guide](https://docs.medusajs.com/learn/basics/modules)
- [Medusa Workflows Guide](https://docs.medusajs.com/learn/basics/workflows)
- [Payload Collections Guide](https://payloadcms.com/docs/configuration/collections)
- [Payload Hooks Guide](https://payloadcms.com/docs/hooks/overview)

## Contributing

Found an issue or want to improve this documentation?

1. Check existing examples in `https://github.com/medusajs/examples/medusa-examples/payload-integration`
2. Test changes thoroughly
3. Update documentation to reflect changes
4. Submit pull request with clear description

## License

This integration example is provided as-is for reference and learning purposes.

---

**Ready to start?** → Begin with [Overview](./01-overview.md) or jump to [Implementation Guide](./04-implementation.md)

