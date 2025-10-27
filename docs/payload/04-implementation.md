# Payload Integration - Implementation Guide

This guide walks you through setting up the Payload integration from scratch.

## Prerequisites

- Node.js v20+
- PostgreSQL database server
- Git CLI
- Basic understanding of Medusa and Next.js

## Architecture Overview

Before starting, understand what you'll be setting up:

```
┌─────────────────────────────────────────────────────────┐
│  YOUR INFRASTRUCTURE                                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌───────────────┐           ┌───────────────┐         │
│  │ PostgreSQL    │           │ PostgreSQL    │         │
│  │ (Medusa DB)   │           │ (Payload DB)  │         │
│  │ Port: 5432    │           │ Port: 5432    │         │
│  └───────┬───────┘           └───────┬───────┘         │
│          │                           │                  │
│  ┌───────▼────────────┐    ┌────────▼──────────┐      │
│  │ MEDUSA BACKEND     │    │ NEXT.JS +         │      │
│  │ http://localhost:  │    │ PAYLOAD CMS       │      │
│  │       9000         │◄──►│ http://localhost: │      │
│  │                    │    │       8000        │      │
│  │ • Admin Dashboard  │    │ • Storefront      │      │
│  │ • Store API        │    │ • Payload Admin   │      │
│  │ • Payload Module   │    │ • Collections     │      │
│  └────────────────────┘    └───────────────────┘      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

## Part 1: Set Up Medusa Backend

### Step 1.1: Create Medusa Project

If you already have a Medusa project, skip to Step 1.2.

```bash
# Create new Medusa project
npx create-medusa-app@latest

# Follow prompts:
# - Project name: my-store
# - Starter: medusa-starter-default
# - Database: PostgreSQL
```

### Step 1.2: Create Payload Module

Create the custom Payload module in your Medusa project:

```bash
cd my-store
mkdir -p src/modules/payload
```

Create the module files:

**`src/modules/payload/types.ts`**:
```typescript
export interface PayloadModuleOptions {
  serverUrl: string
  apiKey: string
  userCollection?: string
}

export interface PayloadCollectionItem {
  id: string
  [key: string]: any
}

export interface PayloadUpsertData {
  [key: string]: any
}

export interface PayloadQueryOptions {
  where?: {
    [key: string]: any
  }
  depth?: number
  limit?: number
  page?: number
  [key: string]: any
}

export interface PayloadItemResult<T = PayloadCollectionItem> {
  doc: T
}

export interface PayloadBulkResult<T = PayloadCollectionItem> {
  docs: T[]
  totalDocs: number
  limit: number
  page: number
  totalPages: number
}

export interface PayloadApiResponse {
  message?: string
  errors?: Array<{ message: string }>
}
```

**`src/modules/payload/service.ts`**:
```typescript
import { MedusaError } from "@medusajs/framework/utils"
import {
  PayloadModuleOptions,
  PayloadCollectionItem,
  PayloadUpsertData,
  PayloadQueryOptions,
  PayloadBulkResult,
  PayloadApiResponse,
  PayloadItemResult,
} from "./types"
import qs from "qs"

type InjectedDependencies = {}

export default class PayloadModuleService {
  private baseUrl: string
  private headers: Record<string, string>
  private defaultOptions: Record<string, any> = {
    is_from_medusa: true,
  }

  constructor(
    container: InjectedDependencies,
    options: PayloadModuleOptions
  ) {
    this.validateOptions(options)
    this.baseUrl = `${options.serverUrl}/api`
    
    this.headers = {
      "Content-Type": "application/json",
      "Authorization": `${options.userCollection || "users"} API-Key ${options.apiKey}`,
    }
  }

  validateOptions(options: Record<any, any>): void | never {
    if (!options.serverUrl) {
      throw new MedusaError(
        MedusaError.Types.INVALID_ARGUMENT,
        "Payload server URL is required"
      )
    }
    
    if (!options.apiKey) {
      throw new MedusaError(
        MedusaError.Types.INVALID_ARGUMENT,
        "Payload API key is required"
      )
    }
  }

  private async makeRequest<T = any>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    const url = `${this.baseUrl}${endpoint}`
    
    try {
      const response = await fetch(url, {
        ...options,
        headers: {
          ...this.headers,
          ...options.headers,
        },
      })

      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}))
        throw new MedusaError(
          MedusaError.Types.UNEXPECTED_STATE,
          `Payload API error: ${response.status} ${response.statusText}. ${
            errorData.message || ""
          }`
        )
      }

      return await response.json()
    } catch (error) {
      throw new MedusaError(
        MedusaError.Types.UNEXPECTED_STATE,
        `Failed to communicate with Payload CMS: ${JSON.stringify(error)}`
      )
    }
  }

  async create<T extends PayloadCollectionItem = PayloadCollectionItem>(
    collection: string,
    data: PayloadUpsertData,
    options: PayloadQueryOptions = {}
  ): Promise<PayloadItemResult<T>> {
    const stringifiedQuery = qs.stringify({
      ...options,
      ...this.defaultOptions
    }, {
      addQueryPrefix: true
    })
    const endpoint = `/${collection}/${stringifiedQuery}`

    const result = await this.makeRequest<PayloadItemResult<T>>(endpoint, {
      method: "POST",
      body: JSON.stringify(data),
    })
    return result
  }

  async update<T extends PayloadCollectionItem = PayloadCollectionItem>(
    collection: string,
    data: PayloadUpsertData,
    options: PayloadQueryOptions = {}
  ): Promise<PayloadItemResult<T>> {
    const stringifiedQuery = qs.stringify({
      ...options,
      ...this.defaultOptions
    }, {
      addQueryPrefix: true
    })
    const endpoint = `/${collection}/${stringifiedQuery}`

    const result = await this.makeRequest<PayloadItemResult<T>>(endpoint, {
      method: "PATCH",
      body: JSON.stringify(data),
    })

    return result
  }

  async delete(
    collection: string,
    options: PayloadQueryOptions = {}
  ): Promise<PayloadApiResponse> {
    const stringifiedQuery = qs.stringify({
      ...options,
      ...this.defaultOptions
    }, {
      addQueryPrefix: true
    })
    const endpoint = `/${collection}/${stringifiedQuery}`

    const result = await this.makeRequest<PayloadApiResponse>(endpoint, {
      method: "DELETE",
    })

    return result
  }

  async find(
    collection: string,
    options: PayloadQueryOptions = {}
  ): Promise<PayloadBulkResult<PayloadCollectionItem>> {
    const stringifiedQuery = qs.stringify({
      ...options,
      ...this.defaultOptions
    }, {
      addQueryPrefix: true
    })
    const endpoint = `/${collection}${stringifiedQuery}`

    const result = await this.makeRequest<PayloadBulkResult<PayloadCollectionItem>>(endpoint)

    return result
  }

  async list(
    filter: {
      product_id: string | string[]
    }
  ) {
    const collection = filter.product_id ? "products" : "unknown"
    const ids = Array.isArray(filter.product_id) ? filter.product_id : [filter.product_id]
    const result = await this.find(
      collection,
      {
        where: {
          medusa_id: {
            in: ids.join(","),
          }
        },
        depth: 2,
      }
    )

    return result.docs.map((doc) => ({
      ...doc,
      product_id: doc.medusa_id,
    }))
  }
}
```

**`src/modules/payload/index.ts`**:
```typescript
import { Module } from "@medusajs/framework/utils"
import PayloadModuleService from "./service"

export const PAYLOAD_MODULE = "payload"

export default Module(PAYLOAD_MODULE, {
  service: PayloadModuleService,
})
```

### Step 1.3: Install Dependencies

```bash
npm install qs
npm install --save-dev @types/qs
```

### Step 1.4: Create Link Definition

Create `src/links/product-payload.ts`:

```typescript
import { defineLink } from "@medusajs/framework/utils"
import ProductModule from "@medusajs/medusa/product"
import { PAYLOAD_MODULE } from "../modules/payload"

export default defineLink(
  {
    linkable: ProductModule.linkable.product,
    field: "id",
  },
  {
    linkable: {
      serviceName: PAYLOAD_MODULE,
      alias: "payload_product",
      primaryKey: "product_id",
    },
  },
  {
    readOnly: true,
  }
)
```

### Step 1.5: Create Workflow Steps

Create `src/workflows/steps/create-payload-items.ts`:

```typescript
import { createStep, StepResponse } from "@medusajs/framework/workflows-sdk"
import { PayloadUpsertData } from "../../modules/payload/types"
import { PAYLOAD_MODULE } from "../../modules/payload"

type StepInput = {
  collection: string
  items: PayloadUpsertData[]
}

export const createPayloadItemsStep = createStep(
  "create-payload-items",
  async ({ items, collection }: StepInput, { container }) => {
    const payloadModuleService = container.resolve(PAYLOAD_MODULE)
    
    const createdItems = await Promise.all(
      items.map(async item => await payloadModuleService.create(
        collection,
        item
      ))
    )

    return new StepResponse({
      items: createdItems.map(item => item.doc)
    }, {
      ids: createdItems.map(item => item.doc.id),
      collection,
    })
  },
  async (data, { container }) => {
    if (!data) {
      return
    }
    const { ids, collection } = data

    const payloadModuleService = container.resolve(PAYLOAD_MODULE)

    await payloadModuleService.delete(
      collection,
      {
        where: {
          id: {
            in: ids.join(",")
          }
        }
      }
    )
  }
)
```

Create similar files for `update-payload-items.ts`, `delete-payload-items.ts`, and `retrieve-payload-items.ts` following the pattern above.

### Step 1.6: Create Workflows

Create `src/workflows/create-payload-products.ts`:

```typescript
import { createWorkflow, transform, WorkflowResponse } from "@medusajs/framework/workflows-sdk"
import { createPayloadItemsStep } from "./steps/create-payload-items"
import { updateProductsWorkflow, useQueryGraphStep } from "@medusajs/medusa/core-flows"

type WorkflowInput = {
  product_ids: string[]
}

export const createPayloadProductsWorkflow = createWorkflow(
  "create-payload-products",
  (input: WorkflowInput) => {
    const { data: products } = useQueryGraphStep({
      entity: "product",
      fields: [
        "id",
        "title",
        "handle",
        "subtitle",
        "description",
        "created_at",
        "updated_at",
        "options.*",
        "variants.*",
        "variants.options.*",
        "thumbnail",
        "images.*"
      ],
      filters: {
        id: input.product_ids,
      },
      options: {
        throwIfKeyNotFound: true,
      }
    })

    const createData = transform({
      products
    }, (data) => {
      return {
        collection: "products",
        items: data.products.map(product => ({
          medusa_id: product.id,
          createdAt: product.created_at as string,
          updatedAt: product.updated_at as string,
          title: product.title,
          subtitle: product.subtitle,
          description: product.description || "",
          options: product.options.map((option) => ({
            title: option.title,
            medusa_id: option.id,
          })),
          variants: product.variants.map((variant) => ({
            title: variant.title,
            medusa_id: variant.id,
            option_values: variant.options.map((option) => ({
              medusa_id: option.id,
              medusa_option_id: option.option?.id,
              value: option.value,
            })),
          })),
        }))
      }
    })

    const { items } = createPayloadItemsStep(
      createData
    )

    const updateData = transform({
      items
    }, (data) => {
      return data.items.map(item => ({
        id: item.medusa_id,
        metadata: {
          payload_id: item.id
        }
      }))
    })

    updateProductsWorkflow.runAsStep({
      input: {
        products: updateData
      }
    })

    return new WorkflowResponse({
      items
    })
  }
)
```

Create similar workflows for updates and deletes.

### Step 1.7: Create Subscribers

Create `src/subscribers/product-created.ts`:

```typescript
import { SubscriberArgs, type SubscriberConfig } from "@medusajs/framework"
import { createPayloadProductsWorkflow } from "../workflows/create-payload-products"

export default async function productCreatedHandler({
  event: { data },
  container,
}: SubscriberArgs<{
  id: string
}>) {
  await createPayloadProductsWorkflow(container)
    .run({
      input: {
        product_ids: [data.id],
      }
    })
}

export const config: SubscriberConfig = {
  event: "product.created",
}
```

Create similar subscribers for other events (see example code).

### Step 1.8: Create Admin API Route

Create `src/api/admin/payload/sync/[collection]/route.ts`:

```typescript
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http"

export const POST = async (
  req: MedusaRequest,
  res: MedusaResponse
) => {
  const { collection } = req.params
  const eventModuleService = req.scope.resolve("event_bus")

  await eventModuleService.emit({
    name: `${collection}.sync-payload`,
    data: {}
  })

  return res.status(200).json({
    message: `Syncing ${collection} with Payload`
  })
}
```

### Step 1.9: Configure Medusa

Update `medusa-config.ts`:

```typescript
import { loadEnv, defineConfig } from '@medusajs/framework/utils'

loadEnv(process.env.NODE_ENV || 'development', process.cwd())

module.exports = defineConfig({
  projectConfig: {
    databaseUrl: process.env.DATABASE_URL,
    http: {
      storeCors: process.env.STORE_CORS!,
      adminCors: process.env.ADMIN_CORS!,
      authCors: process.env.AUTH_CORS!,
      jwtSecret: process.env.JWT_SECRET || "supersecret",
      cookieSecret: process.env.COOKIE_SECRET || "supersecret",
    }
  },
  modules: [
    {
      resolve: "./src/modules/payload",
      options: {
        serverUrl: process.env.PAYLOAD_SERVER_URL || "http://localhost:8000",
        apiKey: process.env.PAYLOAD_API_KEY,
        userCollection: process.env.PAYLOAD_USER_COLLECTION || "users",
      },
    },
  ]
})
```

### Step 1.10: Set Environment Variables

Create `.env`:

```bash
# Medusa Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/medusa

# Payload Integration
PAYLOAD_SERVER_URL=http://localhost:8000
PAYLOAD_API_KEY=                    # Will generate in Part 2
PAYLOAD_USER_COLLECTION=users

# Medusa Configuration
STORE_CORS=http://localhost:8000,http://localhost:7001
ADMIN_CORS=http://localhost:7001,http://localhost:9000
AUTH_CORS=http://localhost:8000,http://localhost:7001
JWT_SECRET=supersecret
COOKIE_SECRET=supersecret
```

### Step 1.11: Install and Start Medusa

```bash
# Install dependencies
npm install

# Run database migrations
npx medusa db:setup

# Seed data (optional)
npm run seed

# Start Medusa
npm run dev
```

Medusa should now be running at `http://localhost:9000`.

## Part 2: Set Up Next.js + Payload

### Step 2.1: Create Next.js Project

If using the example storefront, skip to Step 2.2.

```bash
# In a separate directory
npx create-next-app@latest storefront
cd storefront
```

### Step 2.2: Install Payload

```bash
npm install payload @payloadcms/db-postgres @payloadcms/richtext-lexical sharp
npm install --save-dev @payloadcms/next
```

### Step 2.3: Create Payload Collections

Create `src/collections/Products.ts`:

```typescript
import { CollectionConfig } from 'payload'
import { convertMarkdownToLexical, editorConfigFactory } from '@payloadcms/richtext-lexical'

export const Products: CollectionConfig = {
  slug: 'products',
  admin: {
    useAsTitle: 'title',
  },
  fields: [
    {
      name: 'medusa_id',
      type: 'text',
      label: 'Medusa Product ID',
      required: true,
      unique: true,
      admin: {
        description: 'The unique identifier from Medusa',
        hidden: true,
      },
      access: {
        update: ({ req }) => !!req.query.is_from_medusa,
      }
    },
    {
      name: 'title',
      type: 'text',
      label: 'Title',
      required: true,
    },
    {
      name: 'subtitle',
      type: 'text',
      label: 'Subtitle',
    },
    {
      name: 'description',
      type: 'richText',
      label: 'Description',
    },
    {
      name: 'thumbnail',
      type: 'upload',
      relationTo: 'media',
      label: 'Thumbnail',
    },
    {
      name: 'images',
      type: 'array',
      label: 'Product Images',
      fields: [
        {
          name: 'image',
          type: 'upload',
          relationTo: 'media',
          required: true,
        },
      ],
    },
    {
      name: 'seo',
      type: 'group',
      label: 'SEO',
      fields: [
        {
          name: 'meta_title',
          type: 'text',
          label: 'Meta Title',
        },
        {
          name: 'meta_description',
          type: 'textarea',
          label: 'Meta Description',
        },
        {
          name: 'meta_keywords',
          type: 'text',
          label: 'Meta Keywords',
        },
      ],
    },
    // Add options and variants fields (see example code)
  ],
  access: {
    create: ({ req }) => !!req.query.is_from_medusa,
    delete: ({ req }) => !!req.query.is_from_medusa,
  },
  hooks: {
    beforeChange: [
      async ({ data, req }) => {
        if (typeof data.description === "string") {
          data.description = convertMarkdownToLexical({
            editorConfig: await editorConfigFactory.default({
              config: req.payload.config
            }),
            markdown: data.description,
          })
        }
        return data
      }
    ]
  }
}
```

Create `src/collections/Media.ts` and `src/collections/Users.ts` (see example code).

### Step 2.4: Create Payload Config

Create `src/payload.config.ts`:

```typescript
import sharp from 'sharp'
import { lexicalEditor } from '@payloadcms/richtext-lexical'
import { postgresAdapter } from '@payloadcms/db-postgres'
import { buildConfig } from 'payload'
import { Users } from './collections/Users'
import { Products } from './collections/Products'
import { Media } from './collections/Media'

export default buildConfig({
  editor: lexicalEditor(),
  collections: [Users, Products, Media],
  secret: process.env.PAYLOAD_SECRET || '',
  db: postgresAdapter({
    pool: {
      connectionString: process.env.PAYLOAD_DATABASE_URL || '',
    },
  }),
  sharp,
})
```

### Step 2.5: Set Up Next.js App Router

Create the Payload admin routes in `src/app/(payload)/admin/[[...segments]]/page.tsx`:

```typescript
import { RenderAdmin } from '@payloadcms/next/views'
import config from '@payload-config'

export default function AdminPage() {
  return <RenderAdmin config={config} />
}
```

Create API routes in `src/app/(payload)/api/[...slug]/route.ts`:

```typescript
import config from '@payload-config'
import '@payloadcms/next/css'
import {
  REST_DELETE,
  REST_GET,
  REST_OPTIONS,
  REST_PATCH,
  REST_POST,
  REST_PUT,
} from '@payloadcms/next/routes'

export const GET = REST_GET(config)
export const POST = REST_POST(config)
export const DELETE = REST_DELETE(config)
export const PATCH = REST_PATCH(config)
export const PUT = REST_PUT(config)
export const OPTIONS = REST_OPTIONS(config)
```

### Step 2.6: Configure Environment Variables

Create `.env.local`:

```bash
# Payload Configuration
PAYLOAD_DATABASE_URL=postgresql://postgres:postgres@localhost:5432/payload
PAYLOAD_SECRET=your-super-secret-key-here

# Medusa Configuration
NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
MEDUSA_PAYLOAD_API_KEY=               # Will generate next
```

### Step 2.7: Update next.config.js

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    reactCompiler: false
  },
  images: {
    remotePatterns: [
      {
        protocol: 'http',
        hostname: 'localhost',
      },
      {
        protocol: 'https',
        hostname: '**.amazonaws.com',
      },
    ],
  },
}

module.exports = nextConfig
```

### Step 2.8: Start Next.js + Payload

```bash
# Install dependencies
npm install

# Start development server
npm run dev
```

Next.js + Payload should now be running at `http://localhost:8000`.

## Part 3: Connect the Systems

### Step 3.1: Create Payload User and API Key

1. Open Payload Admin: `http://localhost:8000/admin`
2. Create your first user (admin account)
3. After logging in, go to your user profile
4. Click "Enable API Key"
5. Copy the generated API key

### Step 3.2: Configure Medusa with API Key

Update Medusa's `.env`:

```bash
PAYLOAD_API_KEY=your-copied-api-key-here
```

Restart Medusa:

```bash
npm run dev
```

### Step 3.3: Test the Integration

1. **Create a product in Medusa**:
   - Open Medusa Admin: `http://localhost:9000/app`
   - Navigate to Products
   - Click "Add Product"
   - Fill in: Title, Description, Options, Variants
   - Click "Save"

2. **Verify sync to Payload**:
   - Open Payload Admin: `http://localhost:8000/admin`
   - Navigate to Products collection
   - You should see the product created from Medusa
   - Check that `medusa_id` field is populated (hidden in UI)

3. **Enrich content in Payload**:
   - Open the product in Payload
   - Update description with rich text formatting
   - Upload product images
   - Add SEO metadata
   - Save

4. **View on storefront**:
   - Navigate to product page: `http://localhost:8000/products/{handle}`
   - Verify you see:
     - Rich description from Payload
     - Images from Payload
     - Prices from Medusa
     - Working add to cart (Medusa commerce)

## Part 4: Integrate Storefront

### Step 4.1: Update Product Fetching

In `src/lib/data/products.ts`, ensure you're fetching with Payload data:

```typescript
sdk.client.fetch(`/store/products`, {
  method: "GET",
  query: {
    fields: "*variants.calculated_price,+variants.inventory_quantity,+metadata,*payload_product"
    //                                                                          ^^^^^^^^^^^^^^^^
  }
})
```

### Step 4.2: Update Product Components

Update your product display components to use Payload data:

```tsx
// src/modules/products/templates/product-info/index.tsx
export default function ProductInfo({ product }: { product: StoreProductWithPayload }) {
  return (
    <div>
      {/* Use Payload title if available */}
      <h1>{product?.payload_product?.title || product.title}</h1>
      
      {/* Use Payload rich description */}
      {product?.payload_product?.description && (
        <RichText data={product.payload_product.description} />
      )}
      
      {/* Fallback to Medusa description */}
      {!product?.payload_product?.description && (
        <p>{product.description}</p>
      )}
    </div>
  )
}
```

### Step 4.3: Add Type Definitions

Create `src/types/global.ts`:

```typescript
import { StoreProduct } from "@medusajs/types"

export type StoreProductWithPayload = StoreProduct & {
  payload_product?: {
    medusa_id: string
    title: string
    description: any // Lexical rich text
    thumbnail?: {
      url: string
    }
    images?: Array<{
      id: string
      image: {
        url: string
      }
    }>
    seo?: {
      meta_title?: string
      meta_description?: string
      meta_keywords?: string
    }
  }
}
```

## Troubleshooting

### Issue: Product not syncing to Payload

**Symptoms**: Product created in Medusa but doesn't appear in Payload

**Debugging steps**:

1. Check Medusa logs for errors:
   ```bash
   # Look for workflow errors or HTTP errors
   tail -f /path/to/medusa/logs
   ```

2. Verify Payload API key:
   ```bash
   # Test API call manually
   curl -X GET http://localhost:8000/api/products \
     -H "Authorization: users API-Key YOUR_API_KEY"
   ```

3. Check event subscriber:
   ```typescript
   // Add logging to src/subscribers/product-created.ts
   console.log("Product created event received:", data.id)
   ```

4. Verify Payload is running:
   ```bash
   curl http://localhost:8000/api/users
   ```

**Common fixes**:
- Wrong API key → Regenerate in Payload
- Wrong URL → Check `PAYLOAD_SERVER_URL` in `.env`
- Network issue → Ensure both servers can communicate

### Issue: Payload data not appearing in storefront

**Symptoms**: Product shows in storefront but no rich description or images

**Debugging steps**:

1. Verify link is working:
   ```bash
   curl "http://localhost:9000/store/products/your-handle?fields=*payload_product"
   ```

2. Check product metadata:
   ```sql
   SELECT metadata FROM product WHERE handle = 'your-handle';
   -- Should contain: {"payload_id": "..."}
   ```

3. Verify Payload product has medusa_id:
   ```bash
   curl http://localhost:8000/api/products?where[medusa_id][equals]=prod_123
   ```

**Common fixes**:
- Missing `*payload_product` in fields query
- Product metadata doesn't have `payload_id`
- Link definition not loaded (restart Medusa)

### Issue: Images not loading

**Symptoms**: Payload admin shows images but storefront doesn't display them

**Debugging steps**:

1. Check image URLs:
   ```typescript
   console.log(product.payload_product?.thumbnail?.url)
   // Should be: /media/filename.jpg or full URL
   ```

2. Verify Next.js image configuration:
   ```javascript
   // next.config.js should allow localhost
   images: {
     remotePatterns: [
       { protocol: 'http', hostname: 'localhost' }
     ]
   }
   ```

3. Check if images are accessible:
   ```bash
   curl http://localhost:8000/media/your-image.jpg
   ```

**Common fixes**:
- Add `http://localhost` to allowed image domains
- Use absolute URLs: `http://localhost:8000${image.url}`
- Check file permissions in `public/media/`

### Issue: "Access Denied" when Medusa creates product in Payload

**Symptoms**: Error: "Access denied" or 403 response

**Debugging steps**:

1. Verify `is_from_medusa` query parameter:
   ```typescript
   // Should be added by PayloadModuleService
   defaultOptions: { is_from_medusa: true }
   ```

2. Check Payload collection access control:
   ```typescript
   // Products.ts
   access: {
     create: ({ req }) => !!req.query.is_from_medusa
   }
   ```

3. Verify API key permissions:
   - API keys must belong to admin users
   - Check user role has create permissions

**Common fixes**:
- Regenerate API key with admin user
- Add `is_from_medusa=true` to all requests
- Check Payload access control logic

## Production Deployment

### Environment Variables

**Medusa (.env)**:
```bash
DATABASE_URL=postgresql://user:pass@prod-db-host:5432/medusa
PAYLOAD_SERVER_URL=https://your-storefront.com
PAYLOAD_API_KEY=production-api-key
STORE_CORS=https://your-storefront.com
```

**Next.js + Payload (.env.local)**:
```bash
PAYLOAD_DATABASE_URL=postgresql://user:pass@prod-db-host:5432/payload
PAYLOAD_SECRET=production-secret-key
NEXT_PUBLIC_MEDUSA_BACKEND_URL=https://api.your-store.com
```

### Deployment Checklist

- [ ] Two separate PostgreSQL databases created
- [ ] Environment variables configured
- [ ] CORS configured for production domains
- [ ] API keys regenerated for production
- [ ] SSL certificates configured
- [ ] Database backups configured
- [ ] Monitoring and logging set up
- [ ] Test product creation → sync → storefront display
- [ ] Test bulk sync with existing products
- [ ] Load test both systems

### Recommended Architecture

```
┌────────────────────────────────────────────────────────┐
│  PRODUCTION ARCHITECTURE                               │
├────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────┐       ┌──────────┐      ┌──────────┐   │
│  │ Client   │──────►│ CDN/     │◄─────│ Next.js  │   │
│  │ Browser  │       │ Load     │      │ +Payload │   │
│  └──────────┘       │ Balancer │      │ (Vercel) │   │
│                     └────┬─────┘      └────┬─────┘   │
│                          │                  │          │
│                          │                  │          │
│                     ┌────▼────┐        ┌───▼──────┐   │
│                     │ Medusa  │◄──────►│ Payload  │   │
│                     │ Backend │        │ Postgres │   │
│                     │ (Cloud) │        └──────────┘   │
│                     └────┬────┘                        │
│                          │                             │
│                     ┌────▼────┐                        │
│                     │ Medusa  │                        │
│                     │ Postgres│                        │
│                     └─────────┘                        │
│                                                         │
└────────────────────────────────────────────────────────┘
```

## Next Steps

- Read [Data Flow](./03-data-flow.md) to understand how data moves
- Review [Pros & Cons](./05-pros-cons.md) for trade-offs
- Customize Payload collections for your needs
- Add custom fields and workflows
- Set up monitoring and alerts

## Support

- [Medusa Documentation](https://docs.medusajs.com)
- [Payload Documentation](https://payloadcms.com/docs)
- [Medusa Discord](https://discord.gg/medusajs)
- [Payload Discord](https://discord.gg/payload)

