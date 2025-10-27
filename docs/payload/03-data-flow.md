# Payload Integration - Data Flow Scenarios

This document walks through specific scenarios showing how data flows through the system.

## Scenario 1: Creating a New Product

### Timeline

```
┌─────────────────────────────────────────────────────────────────────┐
│  SCENARIO: Admin Creates "Blue Hoodie" Product                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  T+0s: Admin in Medusa Dashboard                                    │
│  ────────────────────────────────────────────────────────────────   │
│  • Navigates to Products                                            │
│  • Clicks "Add Product"                                             │
│  • Fills in:                                                        │
│    - Title: "Blue Hoodie"                                           │
│    - Description: "Warm and cozy hoodie"                            │
│    - Handle: "blue-hoodie"                                          │
│    - Options: Size (S, M, L)                                        │
│    - Variants: 3 variants (Small, Medium, Large)                    │
│    - Prices: $49.99                                                 │
│  • Clicks "Save"                                                    │
│                                                                      │
│  T+0.5s: Medusa Creates Product                                     │
│  ────────────────────────────────────────────────────────────────   │
│  Medusa Database:                                                   │
│  INSERT INTO product (id, title, description, handle, ...)          │
│  VALUES ('prod_01JH123', 'Blue Hoodie', ...)                        │
│                                                                      │
│  INSERT INTO product_option (id, product_id, title)                 │
│  VALUES ('opt_01JH456', 'prod_01JH123', 'Size')                     │
│                                                                      │
│  INSERT INTO product_variant (id, product_id, title, ...)           │
│  VALUES                                                              │
│    ('var_01JH789', 'prod_01JH123', 'Small', ...),                   │
│    ('var_01JH790', 'prod_01JH123', 'Medium', ...),                  │
│    ('var_01JH791', 'prod_01JH123', 'Large', ...)                    │
│                                                                      │
│  T+0.6s: Event Emitted                                              │
│  ────────────────────────────────────────────────────────────────   │
│  Event Bus:                                                         │
│  {                                                                  │
│    name: "product.created",                                         │
│    data: { id: "prod_01JH123" }                                     │
│  }                                                                  │
│                                                                      │
│  T+0.7s: Subscriber Triggered                                       │
│  ────────────────────────────────────────────────────────────────   │
│  File: src/subscribers/product-created.ts                           │
│  Executes: createPayloadProductsWorkflow({                          │
│    product_ids: ["prod_01JH123"]                                    │
│  })                                                                 │
│                                                                      │
│  T+0.8s: Workflow Step 1 - Query Product                            │
│  ────────────────────────────────────────────────────────────────   │
│  Query Medusa Graph:                                                │
│  {                                                                  │
│    entity: "product",                                               │
│    fields: ["id", "title", "subtitle", "description",              │
│             "options.*", "variants.*", "images.*"]                  │
│    filters: { id: ["prod_01JH123"] }                                │
│  }                                                                  │
│                                                                      │
│  Returns:                                                           │
│  {                                                                  │
│    id: "prod_01JH123",                                              │
│    title: "Blue Hoodie",                                            │
│    description: "Warm and cozy hoodie",                             │
│    handle: "blue-hoodie",                                           │
│    options: [                                                       │
│      { id: "opt_01JH456", title: "Size" }                           │
│    ],                                                               │
│    variants: [                                                      │
│      {                                                              │
│        id: "var_01JH789",                                           │
│        title: "Small",                                              │
│        options: [                                                   │
│          {                                                          │
│            id: "optval_01JH111",                                    │
│            option: { id: "opt_01JH456" },                           │
│            value: "S"                                               │
│          }                                                          │
│        ]                                                            │
│      },                                                             │
│      // ... Medium and Large variants                               │
│    ]                                                                │
│  }                                                                  │
│                                                                      │
│  T+0.9s: Workflow Step 2 - Transform Data                           │
│  ────────────────────────────────────────────────────────────────   │
│  Transform to Payload format:                                       │
│  {                                                                  │
│    collection: "products",                                          │
│    items: [                                                         │
│      {                                                              │
│        medusa_id: "prod_01JH123",                                   │
│        title: "Blue Hoodie",                                        │
│        subtitle: null,                                              │
│        description: "Warm and cozy hoodie",                         │
│        options: [                                                   │
│          {                                                          │
│            title: "Size",                                           │
│            medusa_id: "opt_01JH456"                                 │
│          }                                                          │
│        ],                                                           │
│        variants: [                                                  │
│          {                                                          │
│            title: "Small",                                          │
│            medusa_id: "var_01JH789",                                │
│            option_values: [                                         │
│              {                                                      │
│                medusa_id: "optval_01JH111",                         │
│                medusa_option_id: "opt_01JH456",                     │
│                value: "S"                                           │
│              }                                                      │
│            ]                                                        │
│          },                                                         │
│          // ... Medium and Large                                    │
│        ]                                                            │
│      }                                                              │
│    ]                                                                │
│  }                                                                  │
│                                                                      │
│  T+1.0s: Workflow Step 3 - Create in Payload                        │
│  ────────────────────────────────────────────────────────────────   │
│  HTTP Request:                                                      │
│  POST http://localhost:8000/api/products?is_from_medusa=true       │
│  Headers: {                                                         │
│    Authorization: "users API-Key sk_abc123...",                     │
│    Content-Type: "application/json"                                 │
│  }                                                                  │
│  Body: { ... transformed data ... }                                 │
│                                                                      │
│  Payload Processing:                                                │
│  1. Validates API key ✓                                             │
│  2. Checks access control (is_from_medusa=true) ✓                   │
│  3. Runs beforeChange hook:                                         │
│     - Converts "Warm and cozy hoodie" markdown to Lexical           │
│  4. Inserts into Payload database                                   │
│                                                                      │
│  Payload Database:                                                  │
│  INSERT INTO products (id, medusa_id, title, description, ...)      │
│  VALUES                                                              │
│  (                                                                  │
│    '67abc123def',                                                   │
│    'prod_01JH123',                                                  │
│    'Blue Hoodie',                                                   │
│    '{"root": {"type": "root", "children": [...Lexical JSON...]}}', │
│    ...                                                              │
│  )                                                                  │
│                                                                      │
│  Response:                                                          │
│  {                                                                  │
│    doc: {                                                           │
│      id: "67abc123def",                                             │
│      medusa_id: "prod_01JH123",                                     │
│      title: "Blue Hoodie",                                          │
│      description: { /* Lexical data */ },                           │
│      // ...                                                         │
│    }                                                                │
│  }                                                                  │
│                                                                      │
│  T+1.2s: Workflow Step 4 - Update Medusa Metadata                   │
│  ────────────────────────────────────────────────────────────────   │
│  Execute updateProductsWorkflow:                                    │
│  {                                                                  │
│    products: [                                                      │
│      {                                                              │
│        id: "prod_01JH123",                                          │
│        metadata: {                                                  │
│          payload_id: "67abc123def"                                  │
│        }                                                            │
│      }                                                              │
│    ]                                                                │
│  }                                                                  │
│                                                                      │
│  Medusa Database:                                                   │
│  UPDATE product                                                     │
│  SET metadata = '{"payload_id": "67abc123def"}'                     │
│  WHERE id = 'prod_01JH123'                                          │
│                                                                      │
│  T+1.3s: Complete ✓                                                 │
│  ────────────────────────────────────────────────────────────────   │
│  Product now exists in both systems:                                │
│  • Medusa: Commerce data + link to Payload                          │
│  • Payload: Content shell with basic info                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### What Exists Where After Creation

**Medusa Database**:
```json
{
  "id": "prod_01JH123",
  "title": "Blue Hoodie",
  "description": "Warm and cozy hoodie",
  "handle": "blue-hoodie",
  "thumbnail": null,
  "metadata": {
    "payload_id": "67abc123def"
  },
  "variants": [/* 3 variants with prices */],
  "options": [/* Size option */]
}
```

**Payload Database**:
```json
{
  "id": "67abc123def",
  "medusa_id": "prod_01JH123",
  "title": "Blue Hoodie",
  "subtitle": null,
  "description": {
    "root": {
      "type": "root",
      "children": [
        {
          "type": "paragraph",
          "children": [
            { "type": "text", "text": "Warm and cozy hoodie" }
          ]
        }
      ]
    }
  },
  "thumbnail": null,
  "images": [],
  "seo": null,
  "options": [
    { "title": "Size", "medusa_id": "opt_01JH456" }
  ],
  "variants": [
    {
      "title": "Small",
      "medusa_id": "var_01JH789",
      "option_values": [
        {
          "medusa_id": "optval_01JH111",
          "medusa_option_id": "opt_01JH456",
          "value": "S"
        }
      ]
    }
    // ... more variants
  ]
}
```

## Scenario 2: Content Manager Enriches Product in Payload

### Timeline

```
┌─────────────────────────────────────────────────────────────────────┐
│  SCENARIO: Content Manager Adds Rich Content                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  T+0s: Content Manager Opens Payload Admin                          │
│  ────────────────────────────────────────────────────────────────   │
│  • Visits: http://localhost:8000/admin                              │
│  • Navigates to Products collection                                 │
│  • Finds "Blue Hoodie"                                              │
│  • Clicks Edit                                                      │
│                                                                      │
│  T+5m: Content Manager Updates Product                              │
│  ────────────────────────────────────────────────────────────────   │
│  Updates:                                                           │
│  1. Description (Rich Text Editor):                                 │
│     - Adds formatting (bold, italic, headings)                      │
│     - Adds bullet points for features                               │
│     - Adds product care instructions                                │
│                                                                      │
│     "# Blue Hoodie - Premium Quality                                │
│                                                                      │
│      Stay warm and stylish with our **Blue Hoodie**.                │
│                                                                      │
│      ## Features                                                    │
│      - 100% organic cotton                                          │
│      - Double-layered hood                                          │
│      - Kangaroo pocket                                              │
│      - Ribbed cuffs and waistband"                                  │
│                                                                      │
│  2. Thumbnail:                                                      │
│     - Uploads "hoodie-front.jpg"                                    │
│     - Payload auto-generates thumbnails                             │
│                                                                      │
│  3. Image Gallery:                                                  │
│     - Uploads 4 images (front, back, detail, lifestyle)             │
│                                                                      │
│  4. SEO Fields:                                                     │
│     - Meta Title: "Blue Hoodie - Organic Cotton | YourBrand"        │
│     - Meta Description: "Shop our premium blue hoodie..."           │
│     - Meta Keywords: "hoodie, blue hoodie, organic cotton"          │
│                                                                      │
│  • Clicks "Save"                                                    │
│                                                                      │
│  T+5m 30s: Payload Updates Database                                 │
│  ────────────────────────────────────────────────────────────────   │
│  Payload Database:                                                  │
│  UPDATE products                                                    │
│  SET                                                                │
│    description = '{"root": {...rich Lexical JSON...}}',             │
│    thumbnail = 'media_01ABC',                                       │
│    seo = '{                                                         │
│      "meta_title": "Blue Hoodie - Organic Cotton | YourBrand",      │
│      "meta_description": "Shop our premium blue hoodie...",         │
│      "meta_keywords": "hoodie, blue hoodie, organic cotton"         │
│    }'                                                               │
│  WHERE id = '67abc123def'                                           │
│                                                                      │
│  INSERT INTO products_rels (parent_id, media_id, order)             │
│  VALUES                                                              │
│    ('67abc123def', 'media_01ABC', 1),  -- front                     │
│    ('67abc123def', 'media_02DEF', 2),  -- back                      │
│    ('67abc123def', 'media_03GHI', 3),  -- detail                    │
│    ('67abc123def', 'media_04JKL', 4)   -- lifestyle                 │
│                                                                      │
│  T+5m 31s: Complete ✓                                               │
│  ────────────────────────────────────────────────────────────────   │
│  Changes saved in Payload ONLY                                      │
│  NO sync to Medusa (by design)                                      │
│  Medusa still has original simple description                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### What Changed

**Medusa Database** (NO CHANGES):
```json
{
  "id": "prod_01JH123",
  "title": "Blue Hoodie",
  "description": "Warm and cozy hoodie",  // ← Still original
  "thumbnail": null,                       // ← Still null
  // ... no changes
}
```

**Payload Database** (UPDATED):
```json
{
  "id": "67abc123def",
  "medusa_id": "prod_01JH123",
  "title": "Blue Hoodie",
  "description": {
    "root": {
      "type": "root",
      "children": [
        {
          "type": "heading",
          "tag": "h1",
          "children": [
            { "type": "text", "text": "Blue Hoodie - Premium Quality" }
          ]
        },
        // ... rich content structure
      ]
    }
  },
  "thumbnail": {
    "id": "media_01ABC",
    "url": "/media/hoodie-front.jpg",
    "thumbnails": {
      "thumbnail": { "url": "/media/hoodie-front-400x300.jpg" },
      "card": { "url": "/media/hoodie-front-768x1024.jpg" }
    }
  },
  "images": [
    { "id": "img_001", "image": "media_01ABC" },
    { "id": "img_002", "image": "media_02DEF" },
    { "id": "img_003", "image": "media_03GHI" },
    { "id": "img_004", "image": "media_04JKL" }
  ],
  "seo": {
    "meta_title": "Blue Hoodie - Organic Cotton | YourBrand",
    "meta_description": "Shop our premium blue hoodie...",
    "meta_keywords": "hoodie, blue hoodie, organic cotton"
  }
}
```

## Scenario 3: Customer Views Product on Storefront

### Timeline

```
┌─────────────────────────────────────────────────────────────────────┐
│  SCENARIO: Customer Browses to Product Page                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  T+0s: Customer Navigates                                           │
│  ────────────────────────────────────────────────────────────────   │
│  • Opens: http://localhost:8000/us/products/blue-hoodie             │
│  • Next.js server-side renders page                                 │
│                                                                      │
│  T+0.1s: Next.js Fetches Product Data                               │
│  ────────────────────────────────────────────────────────────────   │
│  Function: retrieveProduct(handle: "blue-hoodie")                   │
│  Location: src/lib/data/products.ts                                 │
│                                                                      │
│  HTTP Request to Medusa:                                            │
│  GET http://localhost:9000/store/products/blue-hoodie               │
│  ?fields=*variants.calculated_price,                                │
│           +variants.inventory_quantity,                             │
│           +metadata,                                                │
│           +tags,                                                    │
│           *payload_product                                          │
│    ^^^^^^^^^^^^^^                                                   │
│    This tells Medusa to include linked Payload data                 │
│                                                                      │
│  T+0.15s: Medusa Processes Request                                  │
│  ────────────────────────────────────────────────────────────────   │
│  1. Query Medusa database for product                               │
│  2. Calculate prices for customer's region (US)                     │
│  3. Check inventory levels                                          │
│  4. See "*payload_product" in fields                                │
│  5. Check product.metadata.payload_id = "67abc123def"               │
│  6. Use product-payload link definition                             │
│  7. Call PayloadModuleService.list({                                │
│       product_id: "prod_01JH123"                                    │
│     })                                                              │
│                                                                      │
│  T+0.2s: Medusa Queries Payload                                     │
│  ────────────────────────────────────────────────────────────────   │
│  HTTP Request from Medusa to Payload:                               │
│  GET http://localhost:8000/api/products                             │
│      ?where[medusa_id][in]=prod_01JH123                             │
│      &depth=2                                                       │
│      &is_from_medusa=true                                           │
│  Headers: {                                                         │
│    Authorization: "users API-Key sk_abc123..."                      │
│  }                                                                  │
│                                                                      │
│  Payload returns:                                                   │
│  {                                                                  │
│    "docs": [                                                        │
│      {                                                              │
│        "id": "67abc123def",                                         │
│        "medusa_id": "prod_01JH123",                                 │
│        "title": "Blue Hoodie",                                      │
│        "description": { /* rich text */ },                          │
│        "thumbnail": {                                               │
│          "id": "media_01ABC",                                       │
│          "url": "/media/hoodie-front.jpg",                          │
│          "thumbnails": { ... }                                      │
│        },                                                           │
│        "images": [ /* 4 images */ ],                                │
│        "seo": { /* SEO data */ }                                    │
│      }                                                              │
│    ]                                                                │
│  }                                                                  │
│                                                                      │
│  T+0.25s: Medusa Combines Data                                      │
│  ────────────────────────────────────────────────────────────────   │
│  Merges Medusa data + Payload data:                                 │
│  {                                                                  │
│    // FROM MEDUSA:                                                  │
│    "id": "prod_01JH123",                                            │
│    "handle": "blue-hoodie",                                         │
│    "title": "Blue Hoodie",                                          │
│    "description": "Warm and cozy hoodie",                           │
│    "variants": [                                                    │
│      {                                                              │
│        "id": "var_01JH789",                                         │
│        "title": "Small",                                            │
│        "calculated_price": {                                        │
│          "calculated_amount": 4999,  // $49.99                      │
│          "currency_code": "usd"                                     │
│        },                                                           │
│        "inventory_quantity": 25                                     │
│      },                                                             │
│      // ... more variants                                           │
│    ],                                                               │
│    "metadata": {                                                    │
│      "payload_id": "67abc123def"                                    │
│    },                                                               │
│                                                                      │
│    // FROM PAYLOAD (linked):                                        │
│    "payload_product": {                                             │
│      "id": "67abc123def",                                           │
│      "medusa_id": "prod_01JH123",                                   │
│      "title": "Blue Hoodie",                                        │
│      "description": {                                               │
│        "root": {                                                    │
│          "type": "root",                                            │
│          "children": [/* rich content */]                           │
│        }                                                            │
│      },                                                             │
│      "thumbnail": {                                                 │
│        "url": "/media/hoodie-front.jpg"                             │
│      },                                                             │
│      "images": [/* 4 images */],                                    │
│      "seo": {                                                       │
│        "meta_title": "Blue Hoodie - Organic Cotton | YourBrand"     │
│      }                                                              │
│    }                                                                │
│  }                                                                  │
│                                                                      │
│  Returns combined data to Next.js                                   │
│                                                                      │
│  T+0.3s: Next.js Renders Page                                       │
│  ────────────────────────────────────────────────────────────────   │
│  Component: ProductInfo                                             │
│  Location: src/modules/products/templates/product-info/index.tsx    │
│                                                                      │
│  Rendering Logic:                                                   │
│  ```tsx                                                             │
│  // Use Payload title if available                                  │
│  <h1>{product?.payload_product?.title || product.title}</h1>        │
│  // Result: "Blue Hoodie"                                           │
│                                                                      │
│  // Use Payload rich description if available                       │
│  {product?.payload_product?.description && (                        │
│    <RichText data={product.payload_product.description} />          │
│  )}                                                                 │
│  // Result: Formatted rich content with headings, bullets           │
│                                                                      │
│  // Use Payload thumbnail                                           │
│  <img src={product.payload_product?.thumbnail?.url} />              │
│  // Result: /media/hoodie-front.jpg                                 │
│                                                                      │
│  // Show Medusa price                                               │
│  <Price amount={variant.calculated_price.calculated_amount} />      │
│  // Result: $49.99                                                  │
│                                                                      │
│  // Show Medusa inventory                                           │
│  <span>In Stock: {variant.inventory_quantity}</span>                │
│  // Result: 25 units                                                │
│  ```                                                                │
│                                                                      │
│  T+0.35s: Page Rendered to Customer ✓                               │
│  ────────────────────────────────────────────────────────────────   │
│  Customer sees:                                                     │
│  • Rich formatted description from Payload                          │
│  • High-quality images from Payload                                 │
│  • Real-time pricing from Medusa                                    │
│  • Current inventory from Medusa                                    │
│  • SEO meta tags from Payload                                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Sources on Product Page

| Element | Source | Reason |
|---------|--------|--------|
| Product Title | Payload (fallback: Medusa) | Content team can optimize |
| Description | Payload | Rich formatting, images |
| Images | Payload | Managed media library |
| SEO Meta Tags | Payload | Optimized for search |
| Price | Medusa | Dynamic, region-specific |
| Inventory | Medusa | Real-time stock levels |
| Variants | Both | Structure from Medusa, labels from Payload |
| Add to Cart | Medusa | Commerce operation |

## Scenario 4: Variant Update Sync

### Timeline

```
┌─────────────────────────────────────────────────────────────────────┐
│  SCENARIO: Admin Updates Variant Title in Medusa                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  T+0s: Admin Updates Variant                                        │
│  ────────────────────────────────────────────────────────────────   │
│  In Medusa Admin:                                                   │
│  • Opens "Blue Hoodie" product                                      │
│  • Edits "Small" variant                                            │
│  • Changes title from "Small" to "Small (Limited Edition)"          │
│  • Saves                                                            │
│                                                                      │
│  T+0.1s: Medusa Emits Event                                         │
│  ────────────────────────────────────────────────────────────────   │
│  Event: "product-variant.updated"                                   │
│  Data: { id: "var_01JH789" }                                        │
│                                                                      │
│  T+0.2s: Subscriber Triggered                                       │
│  ────────────────────────────────────────────────────────────────   │
│  File: src/subscribers/variant-updated.ts                           │
│  Executes: updatePayloadProductVariantsWorkflow({                   │
│    variant_ids: ["var_01JH789"]                                     │
│  })                                                                 │
│                                                                      │
│  T+0.3s: Workflow Queries Data                                      │
│  ────────────────────────────────────────────────────────────────   │
│  Queries Medusa for variant with linked Payload product:            │
│  {                                                                  │
│    id: "var_01JH789",                                               │
│    title: "Small (Limited Edition)",                                │
│    options: [/* option values */],                                  │
│    product: {                                                       │
│      payload_product: {                                             │
│        id: "67abc123def",                                           │
│        variants: [                                                  │
│          {                                                          │
│            medusa_id: "var_01JH789",                                │
│            title: "Small",  // ← Old title                          │
│            option_values: [...]                                     │
│          },                                                         │
│          // ... other variants                                      │
│        ]                                                            │
│      }                                                              │
│    }                                                                │
│  }                                                                  │
│                                                                      │
│  T+0.4s: Workflow Updates Payload                                   │
│  ────────────────────────────────────────────────────────────────   │
│  HTTP Request:                                                      │
│  PATCH http://localhost:8000/api/products/67abc123def              │
│        ?is_from_medusa=true                                         │
│  Body: {                                                            │
│    variants: [                                                      │
│      {                                                              │
│        medusa_id: "var_01JH789",                                    │
│        title: "Small (Limited Edition)",  // ← Updated              │
│        option_values: [...]                                         │
│      },                                                             │
│      // ... other variants unchanged                                │
│    ]                                                                │
│  }                                                                  │
│                                                                      │
│  T+0.5s: Payload Updates Database                                   │
│  ────────────────────────────────────────────────────────────────   │
│  UPDATE products                                                    │
│  SET variants = '[                                                  │
│    {                                                                │
│      "medusa_id": "var_01JH789",                                    │
│      "title": "Small (Limited Edition)",                            │
│      "option_values": [...]                                         │
│    },                                                               │
│    ...                                                              │
│  ]'                                                                 │
│  WHERE id = '67abc123def'                                           │
│                                                                      │
│  T+0.6s: Sync Complete ✓                                            │
│  ────────────────────────────────────────────────────────────────   │
│  Variant title now matches in both systems                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Scenario 5: Product Deletion

### Timeline

```
┌─────────────────────────────────────────────────────────────────────┐
│  SCENARIO: Admin Deletes Product                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  T+0s: Admin Deletes Product                                        │
│  ────────────────────────────────────────────────────────────────   │
│  In Medusa Admin:                                                   │
│  • Finds "Blue Hoodie" product                                      │
│  • Clicks "Delete"                                                  │
│  • Confirms deletion                                                │
│                                                                      │
│  T+0.1s: Medusa Deletes Product                                     │
│  ────────────────────────────────────────────────────────────────   │
│  DELETE FROM product_variant WHERE product_id = 'prod_01JH123'      │
│  DELETE FROM product_option WHERE product_id = 'prod_01JH123'       │
│  DELETE FROM product WHERE id = 'prod_01JH123'                      │
│                                                                      │
│  T+0.2s: Event Emitted                                              │
│  ────────────────────────────────────────────────────────────────   │
│  Event: "product.deleted"                                           │
│  Data: { id: "prod_01JH123" }                                       │
│                                                                      │
│  T+0.3s: Subscriber Triggered                                       │
│  ────────────────────────────────────────────────────────────────   │
│  File: src/subscribers/product-deleted.ts                           │
│  Executes: deletePayloadProductsWorkflow({                          │
│    product_ids: ["prod_01JH123"]                                    │
│  })                                                                 │
│                                                                      │
│  T+0.4s: Workflow Deletes from Payload                              │
│  ────────────────────────────────────────────────────────────────   │
│  HTTP Request:                                                      │
│  DELETE http://localhost:8000/api/products                          │
│         ?where[medusa_id][in]=prod_01JH123                          │
│         &is_from_medusa=true                                        │
│                                                                      │
│  Payload Database:                                                  │
│  DELETE FROM products WHERE medusa_id = 'prod_01JH123'              │
│  Result: Deleted 67abc123def                                        │
│                                                                      │
│  T+0.5s: Complete ✓                                                 │
│  ────────────────────────────────────────────────────────────────   │
│  Product removed from both systems                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Scenario 6: Bulk Sync

### Timeline

```
┌─────────────────────────────────────────────────────────────────────┐
│  SCENARIO: Admin Triggers Manual Bulk Sync                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Use Case: You have 500 products in Medusa, Payload is empty        │
│                                                                      │
│  T+0s: Admin Opens Payload Settings                                 │
│  ────────────────────────────────────────────────────────────────   │
│  In Medusa Admin:                                                   │
│  • Navigate to Settings                                             │
│  • Click "Payload" section                                          │
│  • Click "Sync Products to Payload" button                          │
│                                                                      │
│  T+0.5s: API Call Triggered                                         │
│  ────────────────────────────────────────────────────────────────   │
│  POST /admin/payload/sync/products                                  │
│                                                                      │
│  API Route emits event:                                             │
│  Event: "products.sync-payload"                                     │
│                                                                      │
│  T+1s: Subscriber Processes                                         │
│  ────────────────────────────────────────────────────────────────   │
│  File: src/subscribers/products-sync-payload.ts                     │
│                                                                      │
│  Logic:                                                             │
│  1. Query all products from Medusa in batches (1000 at a time)      │
│  2. Filter products without payload_id in metadata                  │
│  3. For each batch, trigger createPayloadProductsWorkflow           │
│                                                                      │
│  Iteration 1:                                                       │
│  • Products 1-500 queried                                           │
│  • 500 products missing payload_id                                  │
│  • Workflow triggered with 500 product IDs                          │
│                                                                      │
│  T+1m: Batch 1 Processing                                           │
│  ────────────────────────────────────────────────────────────────   │
│  createPayloadProductsWorkflow runs 500 times                       │
│  (Could be optimized with bulk API in future)                       │
│                                                                      │
│  For each product:                                                  │
│  1. Query product data from Medusa                                  │
│  2. Transform to Payload format                                     │
│  3. Create in Payload via API                                       │
│  4. Update Medusa metadata with payload_id                          │
│                                                                      │
│  T+5m: Complete ✓                                                   │
│  ────────────────────────────────────────────────────────────────   │
│  All 500 products now exist in Payload                              │
│  Content team can begin enriching products                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Data Flow Summary

### What Syncs Automatically

| Event | Direction | Data | Timing |
|-------|-----------|------|--------|
| Product Created | Medusa → Payload | Product structure, options, variants | Immediate (< 1s) |
| Product Deleted | Medusa → Payload | Delete command | Immediate (< 1s) |
| Variant Created | Medusa → Payload | New variant data | Immediate (< 1s) |
| Variant Updated | Medusa → Payload | Updated variant | Immediate (< 1s) |
| Variant Deleted | Medusa → Payload | Delete command | Immediate (< 1s) |
| Option Created | Medusa → Payload | New option data | Immediate (< 1s) |
| Option Deleted | Medusa → Payload | Delete command | Immediate (< 1s) |

### What DOESN'T Sync

| Change | Where | Impact |
|--------|-------|--------|
| Content updates in Payload | Payload only | Medusa keeps original description |
| Image uploads in Payload | Payload only | Medusa thumbnail unchanged |
| SEO updates in Payload | Payload only | No impact on Medusa |
| Price changes in Medusa | Medusa only | Payload doesn't store prices |
| Inventory changes in Medusa | Medusa only | Payload doesn't store inventory |
| Title changes in Payload | Payload only | Medusa title unchanged |

### Data Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    COMPLETE DATA FLOW                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────┐                                             │
│  │   MEDUSA        │                                             │
│  │   (Commerce)    │                                             │
│  └────────┬────────┘                                             │
│           │                                                       │
│           │ (1) Product Created/Updated/Deleted                  │
│           ├─────► Event Bus                                      │
│           │                                                       │
│           │ (2) Subscriber Catches Event                         │
│           ├─────► Workflow Triggered                             │
│           │                                                       │
│           │ (3) Query Product Data                               │
│           ◄──────┐                                               │
│           │      │                                               │
│           │ (4) Transform & Sync                                 │
│           ├──────┼─────────────────────┐                         │
│           │      │                     │                         │
│           │      │                     ▼                         │
│  ┌────────▼──────▼────┐     ┌─────────────────┐                │
│  │  Payload Module     │────►│  PAYLOAD CMS    │                │
│  │  (HTTP Client)      │     │  (Content)      │                │
│  └─────────────────────┘     └────────┬────────┘                │
│                                        │                         │
│                               (5) Stores Content                 │
│                                        │                         │
│                                        │                         │
│  ┌─────────────────────────────────────┼─────────────┐          │
│  │         STOREFRONT READS            │             │          │
│  └─────────────────────────────────────┼─────────────┘          │
│                                        │                         │
│                               (6) Query with Link                │
│           ┌────────────────────────────┘                         │
│           │                                                       │
│  ┌────────▼────────┐                                             │
│  │  NEXT.JS        │                                             │
│  │  Fetches:       │                                             │
│  │  • Medusa API ──┼──► Prices, Inventory, Cart                 │
│  │  • Payload      │                                             │
│  │    (via link) ──┼──► Content, Images, SEO                    │
│  └─────────────────┘                                             │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

## Next Steps

- [Implementation Guide](./04-implementation.md) - Step-by-step setup instructions
- [Pros & Cons](./05-pros-cons.md) - When to use this integration

