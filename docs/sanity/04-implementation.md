# Sanity CMS Integration - Implementation Guide

Complete step-by-step guide to implement Sanity CMS integration with Medusa v2.

## Prerequisites

Before starting, ensure you have:

### Required

- ✅ **Node.js v20+** installed
- ✅ **PostgreSQL** running locally or accessible remotely
- ✅ **Git** installed
- ✅ **Sanity account** (free tier available at [sanity.io](https://www.sanity.io))
- ✅ Basic knowledge of TypeScript and React
- ✅ Familiarity with Medusa and Next.js

### Tools

```bash
node --version  # Should be v20 or higher
npm --version   # Should be 9+
psql --version  # Any recent version
git --version
```

## Part 1: Create and Configure Sanity Project

### Step 1.1: Create Sanity Project

```bash
# Install Sanity CLI globally (optional)
npm install -g sanity

# Or use npx (recommended)
npx sanity@latest init
```

Follow the prompts:

```
? Select project to use: Create new project
? Your project name: My Medusa Store
? Use the default dataset configuration? Yes
? Project output path: /path/to/your/sanity-studio (skip - not needed)
? Select project template: Clean project with no predefined schemas
```

**Important**: Note your **Project ID** - you'll need this later!

Example: `abc123xyz`

### Step 1.2: Generate API Token

1. Go to [manage.sanity.io](https://manage.sanity.io)
2. Select your project
3. Navigate to **API** → **Tokens**
4. Click **Add API token**
5. Settings:
   - **Name**: "Medusa Backend Integration"
   - **Permissions**: **Editor** (required for create/update/delete)
6. Click **Add token**
7. **COPY THE TOKEN IMMEDIATELY** - you won't see it again!

Example token: `skAbc123xyz...`

### Step 1.3: Configure CORS

Still in Sanity project settings:

1. Navigate to **API** → **CORS Origins**
2. Click **Add CORS origin**
3. Add these origins (adjust ports if needed):
   ```
   http://localhost:9000   (Medusa backend)
   http://localhost:8000   (Next.js storefront)
   ```
4. For each origin:
   - ✅ Allow credentials
   - ✅ All HTTP methods (GET, POST, PUT, PATCH, DELETE)

**For Production**: Add your production URLs later

## Part 2: Set Up Medusa Backend

### Step 2.1: Clone or Create Medusa Project

If starting fresh:

```bash
npx create-medusa-app@latest my-medusa-store
cd my-medusa-store/backend
```

If using existing project, navigate to your Medusa directory.

### Step 2.2: Install Dependencies

```bash
# Install Sanity client
npm install @sanity/client

# Or with yarn
yarn add @sanity/client
```

### Step 2.3: Configure Environment Variables

Create or edit `.env`:

```bash
# Existing Medusa variables
DATABASE_URL=postgresql://user:password@localhost:5432/medusa-db
STORE_CORS=http://localhost:8000
ADMIN_CORS=http://localhost:7001,http://localhost:7000
AUTH_CORS=http://localhost:7001,http://localhost:7000
JWT_SECRET=supersecret
COOKIE_SECRET=supersecret

# Add Sanity configuration
SANITY_API_TOKEN=skAbc123xyz...                    # Your token from Step 1.2
SANITY_PROJECT_ID=abc123xyz                        # Your project ID
SANITY_STUDIO_URL=http://localhost:8000/studio    # Studio location
```

⚠️ **Security**: Never commit `.env` to Git! Ensure it's in `.gitignore`.

### Step 2.4: Create Sanity Module

Create directory structure:

```bash
mkdir -p src/modules/sanity
touch src/modules/sanity/index.ts
touch src/modules/sanity/service.ts
```

**File**: `src/modules/sanity/index.ts`

```typescript
import { Module } from "@medusajs/framework/utils";
import SanityModuleService from "./service";

export const SANITY_MODULE = "sanity";

export default Module(SANITY_MODULE, {
  service: SanityModuleService,
});
```

**File**: `src/modules/sanity/service.ts`

```typescript
import { ProductDTO, Logger } from "@medusajs/framework/types";
import {
  createClient,
  FirstDocumentMutationOptions,
  SanityClient,
} from "@sanity/client";

const SyncDocumentTypes = {
  PRODUCT: "product",
} as const;

type SyncDocumentTypes =
  (typeof SyncDocumentTypes)[keyof typeof SyncDocumentTypes];

type SyncDocumentInputs<T> = T extends "product" ? ProductDTO : never;

type ModuleOptions = {
  api_token: string;
  project_id: string;
  api_version: string;
  dataset: "production" | "development";
  type_map?: Record<SyncDocumentTypes, string>;
  studio_url?: string;
};

type TransformationMap<T> = Record<
  SyncDocumentTypes,
  (data: SyncDocumentInputs<T>) => any
>;

type InjectedDependencies = {
  logger: Logger;
};

export default class SanityModuleService {
  private client: SanityClient;
  private studioUrl?: string;
  private logger: Logger;
  private typeMap: Record<SyncDocumentTypes, string>;
  private createTransformationMap: TransformationMap<SyncDocumentTypes>;
  private updateTransformationMap: TransformationMap<SyncDocumentTypes>;

  constructor({ logger }: InjectedDependencies, options: ModuleOptions) {
    this.client = createClient({
      projectId: options.project_id,
      apiVersion: options.api_version,
      dataset: options.dataset,
      token: options.api_token,
    });
    this.logger = logger;

    this.logger.info("Connected to Sanity");

    this.studioUrl = options.studio_url;
    this.typeMap = Object.assign(
      {},
      {
        [SyncDocumentTypes.PRODUCT]: "product",
      },
      options.type_map || {}
    );

    this.createTransformationMap = {
      [SyncDocumentTypes.PRODUCT]: this.transformProductForCreate,
    };

    this.updateTransformationMap = {
      [SyncDocumentTypes.PRODUCT]: this.transformProductForUpdate,
    };
  }

  async upsertSyncDocument<T extends SyncDocumentTypes>(
    type: T,
    data: SyncDocumentInputs<T>
  ) {
    const existing = await this.client.getDocument(data.id);
    if (existing) {
      return await this.updateSyncDocument(type, data);
    }
    return await this.createSyncDocument(type, data);
  }

  async createSyncDocument<T extends SyncDocumentTypes>(
    type: T,
    data: SyncDocumentInputs<T>,
    options?: FirstDocumentMutationOptions
  ) {
    const doc = this.createTransformationMap[type](data);
    return await this.client.create(doc, options);
  }

  async updateSyncDocument<T extends SyncDocumentTypes>(
    type: T,
    data: SyncDocumentInputs<T>
  ) {
    const operations = this.updateTransformationMap[type](data);
    return await this.client.patch(data.id, operations).commit();
  }

  async getStudioLink(
    type: string,
    id: string,
    config: { explicit_type?: boolean } = {}
  ) {
    const resolvedType = config.explicit_type ? type : this.typeMap[type];
    if (!this.studioUrl) {
      throw new Error("No studio URL provided");
    }
    return `${this.studioUrl}/structure/${resolvedType};${id}`;
  }

  async list(filter: { id: string | string[] }) {
    const data = await this.client.getDocuments(
      Array.isArray(filter.id) ? filter.id : [filter.id]
    );

    return data.map((doc) => ({
      id: doc?._id,
      ...doc,
    }));
  }

  async retrieve(id: string) {
    return this.client.getDocument(id);
  }

  async delete(id: string) {
    return this.client.delete(id);
  }

  async update(id: string, data: any) {
    return await this.client.patch(id, { set: data }).commit();
  }

  private transformProductForUpdate = (product: ProductDTO) => {
    return {
      set: {
        title: product.title,
      },
    };
  };

  private transformProductForCreate = (product: ProductDTO) => {
    return {
      _type: this.typeMap[SyncDocumentTypes.PRODUCT],
      _id: product.id,
      title: product.title,
      specs: [
        {
          _key: product.id,
          _type: "spec",
          title: product.title,
          lang: "en",
        },
      ],
    };
  };
}
```

### Step 2.5: Register Module in Config

**File**: `medusa-config.ts`

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
    // ... your existing modules
    {
      resolve: "./src/modules/sanity",
      options: {
        api_token: process.env.SANITY_API_TOKEN,
        project_id: process.env.SANITY_PROJECT_ID,
        api_version: new Date().toISOString().split("T")[0], // YYYY-MM-DD
        dataset: "production",
        studio_url: process.env.SANITY_STUDIO_URL || 
          "http://localhost:8000/studio",
        type_map: {
          product: "product",
        },
      },
    },
  ]
})
```

### Step 2.6: Create Link Definition

```bash
mkdir -p src/links
touch src/links/product-sanity.ts
```

**File**: `src/links/product-sanity.ts`

```typescript
import { defineLink } from "@medusajs/framework/utils";
import ProductModule from "@medusajs/medusa/product";
import { SANITY_MODULE } from "../modules/sanity";

defineLink(
  {
    linkable: ProductModule.linkable.product.id,
    field: "id",
  },
  {
    linkable: {
      serviceName: SANITY_MODULE,
      alias: "sanity_product",
      primaryKey: "id",
    },
  },
  {
    readOnly: true,
  }
);
```

### Step 2.7: Create Workflow

```bash
mkdir -p src/workflows/sanity-sync-products/steps
touch src/workflows/sanity-sync-products/index.ts
touch src/workflows/sanity-sync-products/steps/sync.ts
```

**File**: `src/workflows/sanity-sync-products/index.ts`

```typescript
import {
  createWorkflow,
  WorkflowResponse,
} from "@medusajs/framework/workflows-sdk";
import { syncStep } from "./steps/sync";

export type SanitySyncProductsWorkflowInput = {
  product_ids?: string[];
};

export const sanitySyncProductsWorkflow = createWorkflow(
  { name: "sanity-sync-products", retentionTime: 10000 },
  function (input: SanitySyncProductsWorkflowInput) {
    const result = syncStep(input);
    return new WorkflowResponse(result);
  }
);
```

**File**: `src/workflows/sanity-sync-products/steps/sync.ts`

```typescript
import { createStep, StepResponse } from "@medusajs/framework/workflows-sdk";
import { ProductDTO } from "@medusajs/framework/types";
import { ContainerRegistrationKeys, promiseAll } from "@medusajs/framework/utils";
import SanityModuleService from "../../../modules/sanity/service";
import { SANITY_MODULE } from "../../../modules/sanity";

export type SyncStepInput = {
  product_ids?: string[];
};

export const syncStep = createStep(
  { name: "sync-step", async: true },
  async (input: SyncStepInput, { container }) => {
    const sanityModule: SanityModuleService = container.resolve(SANITY_MODULE);
    const query = container.resolve(ContainerRegistrationKeys.QUERY);

    let total = 0;
    const upsertMap: {
      before: any;
      after: any;
    }[] = [];

    const batchSize = 200;
    let hasMore = true;
    let offset = 0;
    let filters = input.product_ids ? { id: input.product_ids } : {};

    while (hasMore) {
      const {
        data: products,
        metadata: { count },
      } = await query.graph({
        entity: "product",
        fields: ["id", "title", "sanity_product.*"],
        filters,
        pagination: {
          skip: offset,
          take: batchSize,
          order: { id: "ASC" },
        },
      });

      try {
        await promiseAll(
          products.map(async (prod) => {
            const after = await sanityModule.upsertSyncDocument(
              "product",
              prod as ProductDTO
            );

            upsertMap.push({
              // @ts-ignore
              before: prod.sanity_product,
              after,
            });

            return after;
          })
        );
      } catch (e) {
        return StepResponse.permanentFailure(
          `An error occurred while syncing documents: ${e}`,
          upsertMap
        );
      }

      offset += batchSize;
      hasMore = offset < count;
      total += products.length;
    }

    return new StepResponse({ total }, upsertMap);
  },
  async (upsertMap, { container }) => {
    if (!upsertMap) {
      return;
    }

    const sanityModule: SanityModuleService = container.resolve(SANITY_MODULE);

    await promiseAll(
      upsertMap.map(({ before, after }) => {
        if (!before) {
          return sanityModule.delete(after._id);
        }

        const { _id: id, ...oldData } = before;
        return sanityModule.update(id, oldData);
      })
    );
  }
);
```

### Step 2.8: Create Subscriber

```bash
mkdir -p src/subscribers
touch src/subscribers/sanity-product-sync.ts
```

**File**: `src/subscribers/sanity-product-sync.ts`

```typescript
import type { SubscriberArgs, SubscriberConfig } from "@medusajs/medusa";
import { sanitySyncProductsWorkflow } from "../workflows/sanity-sync-products";

export default async function upsertSanityProduct({
  event: { data },
  container,
}: SubscriberArgs<{ id: string }>) {
  await sanitySyncProductsWorkflow(container).run({
    input: {
      product_ids: [data.id],
    },
  });
}

export const config: SubscriberConfig = {
  event: ["product.created", "product.updated"],
};
```

### Step 2.9: Create Admin API Routes

```bash
mkdir -p src/api/admin/sanity/syncs
mkdir -p src/api/admin/sanity/documents/\[id\]/sync
touch src/api/admin/sanity/syncs/route.ts
touch src/api/admin/sanity/documents/\[id\]/route.ts
touch src/api/admin/sanity/documents/\[id\]/sync/route.ts
```

**File**: `src/api/admin/sanity/syncs/route.ts`

```typescript
import { MedusaRequest, MedusaResponse } from "@medusajs/framework";
import { Modules } from "@medusajs/framework/utils";
import { sanitySyncProductsWorkflow } from "../../../../workflows/sanity-sync-products";

export const POST = async (req: MedusaRequest, res: MedusaResponse) => {
  const { transaction } = await sanitySyncProductsWorkflow(req.scope).run({
    input: {},
  });

  res.json({ transaction_id: transaction.transactionId });
};

export const GET = async (req: MedusaRequest, res: MedusaResponse) => {
  const workflowEngine = req.scope.resolve(Modules.WORKFLOW_ENGINE);

  const [executions, count] =
    await workflowEngine.listAndCountWorkflowExecutions(
      {
        workflow_id: sanitySyncProductsWorkflow.getName(),
      },
      { order: { created_at: "DESC" } }
    );

  res.json({ workflow_executions: executions, count });
};
```

**File**: `src/api/admin/sanity/documents/[id]/route.ts`

```typescript
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http";
import SanityModuleService from "src/modules/sanity/service";
import { SANITY_MODULE } from "../../../../../modules/sanity";

export const GET = async (req: MedusaRequest, res: MedusaResponse) => {
  const { id } = req.params;

  const sanityModule: SanityModuleService = req.scope.resolve(SANITY_MODULE);
  const sanityDocument = await sanityModule.retrieve(id);

  const url = sanityDocument
    ? await sanityModule.getStudioLink(sanityDocument._type, sanityDocument._id, {
        explicit_type: true,
      })
    : "";

  res.json({ sanity_document: sanityDocument, studio_url: url });
};
```

**File**: `src/api/admin/sanity/documents/[id]/sync/route.ts`

```typescript
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http";
import { sanitySyncProductsWorkflow } from "../../../../../../workflows/sanity-sync-products";

export const POST = async (req: MedusaRequest, res: MedusaResponse) => {
  const { transaction } = await sanitySyncProductsWorkflow(req.scope).run({
    input: { product_ids: [req.params.id] },
  });

  res.json({ transaction_id: transaction.transactionId });
};
```

### Step 2.10: Setup Database and Start

```bash
# Create/migrate database
npx medusa db:setup

# Start Medusa
npm run dev
```

You should see in logs:
```
Connected to Sanity
```

## Part 3: Set Up Next.js Storefront with Sanity Studio

### Step 3.1: Install Dependencies

```bash
cd storefront  # or your Next.js directory
npm install sanity next-sanity @sanity/vision
```

### Step 3.2: Configure Environment Variables

Create or edit `.env.local`:

```bash
# Existing Medusa variables
NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY=pk_...

# Add Sanity configuration (PUBLIC - safe for frontend)
NEXT_PUBLIC_SANITY_PROJECT_ID=abc123xyz
NEXT_PUBLIC_SANITY_DATASET=production
```

### Step 3.3: Create Sanity Configuration

```bash
touch sanity.config.ts
mkdir -p src/sanity/lib
mkdir -p src/sanity/schemaTypes/documents
touch src/sanity/env.ts
touch src/sanity/lib/client.ts
touch src/sanity/schemaTypes/index.ts
touch src/sanity/schemaTypes/documents/product.ts
```

**File**: `src/sanity/env.ts`

```typescript
export const apiVersion =
  process.env.NEXT_PUBLIC_SANITY_API_VERSION || '2024-01-01'

export const dataset = process.env.NEXT_PUBLIC_SANITY_DATASET || 'production'

export const projectId = process.env.NEXT_PUBLIC_SANITY_PROJECT_ID!

if (!projectId) {
  throw new Error('Missing NEXT_PUBLIC_SANITY_PROJECT_ID')
}
```

**File**: `src/sanity/lib/client.ts`

```typescript
import { createClient } from 'next-sanity'
import { apiVersion, dataset, projectId } from '../env'

export const client = createClient({
  projectId,
  dataset,
  apiVersion,
  useCdn: true,
})
```

**File**: `src/sanity/schemaTypes/documents/product.ts`

```typescript
import { ComposeIcon } from "@sanity/icons";
import { DocumentDefinition } from "sanity";

const productSchema: DocumentDefinition = {
  fields: [
    {
      name: "title",
      type: "string",
    },
    {
      group: "content",
      name: "specs",
      of: [
        {
          fields: [
            { name: "lang", title: "Language", type: "string" },
            { name: "title", title: "Title", type: "string" },
            {
              name: "content",
              rows: 3,
              title: "Content",
              type: "text",
            },
          ],
          name: "spec",
          type: "object",
        },
      ],
      type: "array",
    },
    {
      fields: [
        { name: "title", title: "Title", type: "string" },
        {
          name: "products",
          of: [{ to: [{ type: "product" }], type: "reference" }],
          title: "Addons",
          type: "array",
          validation: (Rule) => Rule.max(3),
        },
      ],
      name: "addons",
      type: "object",
    },
  ],
  name: "product",
  preview: {
    select: {
      title: "title",
    },
  },
  title: "Product Page",
  type: "document",
  groups: [
    {
      default: true,
      // @ts-ignore
      icon: ComposeIcon,
      name: "content",
      title: "Content",
    },
  ],
};

export default productSchema;
```

**File**: `src/sanity/schemaTypes/index.ts`

```typescript
import productSchema from './documents/product'

export const schema = {
  types: [productSchema],
}
```

**File**: `sanity.config.ts` (root)

```typescript
'use client'

import { visionTool } from '@sanity/vision'
import { defineConfig } from 'sanity'
import { structureTool } from 'sanity/structure'
import { apiVersion, dataset, projectId } from './src/sanity/env'
import { schema } from './src/sanity/schemaTypes'

export default defineConfig({
  basePath: '/studio',
  projectId,
  dataset,
  schema,
  plugins: [
    structureTool(),
    visionTool({ defaultApiVersion: apiVersion }),
  ],
})
```

### Step 3.4: Create Studio Route

```bash
mkdir -p src/app/studio/\[\[...tool\]\]
touch src/app/studio/\[\[...tool\]\]/page.tsx
```

**File**: `src/app/studio/[[...tool]]/page.tsx`

```typescript
'use client'

import { NextStudio } from 'next-sanity/studio'
import config from '../../../../sanity.config'

export default function StudioPage() {
  return <NextStudio config={config} />
}
```

### Step 3.5: Update Product Page to Fetch Sanity

**File**: `src/app/[countryCode]/(main)/products/[handle]/page.tsx`

```typescript
import { Metadata } from "next"
import { notFound } from "next/navigation"
import { listProducts } from "@lib/data/products"
import { getRegion } from "@lib/data/regions"
import ProductTemplate from "@modules/products/templates"
import { client } from "../../../../../sanity/lib/client"

type Props = {
  params: Promise<{ countryCode: string; handle: string }>
}

export default async function ProductPage(props: Props) {
  const params = await props.params
  const region = await getRegion(params.countryCode)

  if (!region) {
    notFound()
  }

  const pricedProduct = await listProducts({
    countryCode: params.countryCode,
    queryParams: { handle: params.handle },
  }).then(({ response }) => response.products[0])

  if (!pricedProduct) {
    notFound()
  }

  // Fetch content from Sanity
  const sanity = (await client.getDocument(pricedProduct.id))?.specs?.[0]

  return (
    <ProductTemplate
      product={pricedProduct}
      region={region}
      countryCode={params.countryCode}
      sanity={sanity}
    />
  )
}
```

### Step 3.6: Update Product Template

Update your product template to use Sanity content:

```typescript
// In your ProductInfo or ProductTemplate component
const ProductInfo = ({ product, sanity }) => {
  return (
    <div>
      <h1>{product.title}</h1>
      <p>{sanity?.content || product.description}</p>
      {/* Sanity content takes priority, fallback to Medusa description */}
    </div>
  )
}
```

### Step 3.7: Start Storefront

```bash
npm run dev
```

Visit:
- **Storefront**: http://localhost:8000
- **Sanity Studio**: http://localhost:8000/studio

## Part 4: Test the Integration

### Test 1: Create Product in Medusa

1. Open Medusa Admin (http://localhost:7001)
2. Navigate to Products
3. Create new product:
   - Title: "Test Product"
   - Description: "Initial description"
   - Add variants, set price
4. Click **Save**

### Test 2: Verify Sync to Sanity

1. Check Medusa logs - should see "Product created, triggering sync"
2. Open Sanity Studio (http://localhost:8000/studio)
3. You should see "Test Product" in products list
4. Click on it - should show title and empty specs array

### Test 3: Enrich Content in Sanity

1. In Sanity Studio, edit "Test Product"
2. Fill in the English content field:
   ```
   This is rich content from Sanity!
   ```
3. Add Spanish spec:
   - Lang: "es"
   - Title: "Producto de Prueba"
   - Content: "¡Contenido rico de Sanity!"
4. Click **Publish**

### Test 4: View on Storefront

1. Visit storefront product page
2. Should display:
   - Title from Medusa
   - **Content from Sanity** (not Medusa description)
   - Price, variants from Medusa

**Success!** ✅ Integration working!

## Troubleshooting

### Issue: "Connected to Sanity" not in logs

**Causes**:
- Module not loaded
- Invalid configuration

**Solutions**:
```bash
# Verify module is in medusa-config.ts
# Check environment variables
echo $SANITY_PROJECT_ID
echo $SANITY_API_TOKEN

# Restart Medusa
npm run dev
```

### Issue: Product not syncing to Sanity

**Check**:
1. Medusa logs for errors
2. Sanity API token has Editor permissions
3. CORS configured correctly
4. Test Sanity connection:

```bash
curl https://abc123.api.sanity.io/v2024-01-01/data/query/production \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d 'query=*[_type=="product"][0..10]'
```

### Issue: Sanity Studio not loading

**Solutions**:
```bash
# Clear Next.js cache
rm -rf .next
npm run dev

# Check environment variables
echo $NEXT_PUBLIC_SANITY_PROJECT_ID

# Verify sanity.config.ts exists in root
```

### Issue: Sanity content not showing on storefront

**Check**:
1. `client.getDocument(productId)` is being called
2. Product ID matches between Medusa and Sanity
3. Browser network tab shows Sanity API call
4. Console for any errors

## Production Deployment

### Medusa Backend

1. Set environment variables in production:
   ```
   SANITY_API_TOKEN=your-production-token
   SANITY_PROJECT_ID=abc123xyz
   SANITY_STUDIO_URL=https://yourstore.com/studio
   ```

2. Update Sanity CORS to include production URL

3. Deploy to Railway, Render, AWS, etc.

### Next.js Storefront

1. Set environment variables:
   ```
   NEXT_PUBLIC_SANITY_PROJECT_ID=abc123xyz
   NEXT_PUBLIC_SANITY_DATASET=production
   ```

2. Deploy to Vercel, Netlify, etc.

3. Studio will be available at `/studio` route

### Sanity Project

No deployment needed - Sanity is SaaS!

Just update:
- CORS origins with production URLs
- API tokens if needed

## Next Steps

- **[Data Flow](./03-data-flow.md)** - Understand how data moves
- **[Pros & Cons](./05-pros-cons.md)** - Evaluate trade-offs
- **[FAQ](./FAQ.md)** - Common questions

