# Implementation Guide

This guide provides step-by-step instructions for setting up the Contentful integration with Medusa.

## Prerequisites

### System Requirements

**Software:**
- Node.js v16 or higher
- npm or yarn package manager
- PostgreSQL v12 or higher
- Redis v6 or higher
- Git (for version control)

**Medusa Setup:**
- Medusa v1.8 or higher installed and running
- Medusa Admin dashboard accessible
- PostgreSQL database configured
- Redis configured for event bus

**Contentful Account:**
- Free or paid Contentful account
- Space created (one is created by default)
- Admin access to space

### Knowledge Requirements

- Basic understanding of Medusa architecture
- Familiarity with Contentful concepts (spaces, content types, entries)
- Command line proficiency
- Basic JavaScript/TypeScript knowledge

---

## Part 1: Set Up Contentful Space

### Step 1.1: Create Contentful Account (if needed)

**1. Sign up for Contentful**
```
Visit: https://www.contentful.com/sign-up/
Fill form with email and password
Verify email address
```

**2. Create Space**
```
After login, Contentful automatically creates a default space
Or click "Add space" → "Create an empty space"
Name: "Medusa E-commerce" (or your preference)
```

**3. Note Your Space ID**
```
Navigate to: Settings → General settings
Find: Space ID (e.g., "abc123xyz")
Copy this - you'll need it later
```

### Step 1.2: Create API Tokens

**1. Create Management API Token (for write operations)**

```
Settings → API keys → Content management tokens
Click "Generate personal access token"
Name: "Medusa Integration"
Click "Generate"
Copy token immediately (shown only once!)
Save as: CONTENTFUL_ACCESS_TOKEN
```

**Example token:**
```
CFPAT-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**2. Create Delivery API Token (for storefront reads)**

```
Settings → API keys → Content delivery / preview tokens
Click "Add API key"
Name: "Storefront Production"
Copy "Content Delivery API - access token"
Save as: CONTENTFUL_DELIVERY_TOKEN
```

**3. Create Preview API Token (optional, for draft content)**

```
Same location as delivery token
Copy "Content Preview API - access token"
Save as: CONTENTFUL_PREVIEW_TOKEN
```

### Step 1.3: Configure Locales

**1. View Current Locales**
```
Settings → Locales
Default: en-US (English - United States)
```

**2. Add Additional Locales (if needed)**
```
Click "Add locale"
Examples:
  - es-ES (Spanish - Spain)
  - de-DE (German - Germany)
  - fr-FR (French - France)
  - ja-JP (Japanese - Japan)

For each locale:
  - Name: Spanish (Spain)
  - Code: es-ES
  - Fallback locale: en-US (if translation missing)
  - Optional: Yes/No (can entries exist without this locale?)
```

**3. Save Configuration**

---

## Part 2: Set Up Medusa Backend

### Step 2.1: Install Plugin

**1. Navigate to Medusa backend directory**
```bash
cd /path/to/medusa-backend
```

**2. Install the plugin**
```bash
npm install medusa-plugin-contentful

# Or with yarn
yarn add medusa-plugin-contentful
```

**3. Verify installation**
```bash
npm list medusa-plugin-contentful
# Should show: medusa-plugin-contentful@x.x.x
```

### Step 2.2: Configure Environment Variables

**1. Open `.env` file**
```bash
nano .env
# Or use your preferred editor
```

**2. Add Contentful configuration**
```bash
# Contentful Configuration
CONTENTFUL_SPACE_ID=your_space_id_here
CONTENTFUL_ACCESS_TOKEN=CFPAT-your_management_token_here
CONTENTFUL_ENV=master

# Optional: Delivery token for storefront
CONTENTFUL_DELIVERY_TOKEN=your_delivery_token_here

# Optional: Preview token
CONTENTFUL_PREVIEW_TOKEN=your_preview_token_here
```

**3. Save and close**

**Security Note:** Never commit `.env` to version control! Ensure `.env` is in `.gitignore`.

### Step 2.3: Configure Plugin in medusa-config.js

**1. Open `medusa-config.js`**
```bash
nano medusa-config.js
```

**2. Add plugin configuration**
```javascript
module.exports = {
  projectConfig: {
    // ... existing config
    redis_url: REDIS_URL,  // Required for event bus
  },
  plugins: [
    // ... existing plugins
    {
      resolve: `medusa-plugin-contentful`,
      options: {
        space_id: process.env.CONTENTFUL_SPACE_ID,
        access_token: process.env.CONTENTFUL_ACCESS_TOKEN,
        environment: process.env.CONTENTFUL_ENV || 'master',
        
        // Optional: Customize field mappings
        // custom_product_fields: {
        //   title: "name",  // If you renamed 'title' field in Contentful
        // },
        
        // Optional: Ignore threshold (seconds to wait before re-syncing)
        // ignore_threshold: 2,
        
        // Optional: Auto-publish entries after creation
        // auto_publish: true,
      },
    },
  ],
};
```

**3. Save and close**

### Step 2.4: Verify Redis is Running

The integration requires Redis for the event bus.

**Check if Redis is running:**
```bash
redis-cli ping
# Should return: PONG
```

**If not running, start Redis:**
```bash
# macOS (Homebrew)
brew services start redis

# Linux (systemd)
sudo systemctl start redis

# Docker
docker run -d -p 6379:6379 redis:latest
```

---

## Part 3: Create Content Models in Contentful

Content models define the structure of your content. You need to create these in Contentful to match Medusa's data structure.

### Step 3.1: Create Product Content Type

**Option A: Via Contentful UI (Recommended for learning)**

```
1. Contentful Dashboard → Content model
2. Click "Add content type"
3. Name: "Product"
4. API Identifier: product (auto-generated)
5. Click "Create"
```

**Add fields:**

| Field Name | Field ID | Type | Localized | Required | Unique | Notes |
|------------|----------|------|-----------|----------|--------|-------|
| Medusa ID | medusaId | Short text | No | Yes | Yes | Reference to Medusa |
| Title | title | Short text | Yes | Yes | No | Product name |
| Subtitle | subtitle | Short text | Yes | No | No | Product tagline |
| Description | description | Long text | Yes | No | No | Base description |
| Rich Description | richDescription | Rich text | Yes | No | No | Editorial content |
| Handle | handle | Short text | No | No | Yes | URL slug |
| Thumbnail | thumbnail | Short text | No | No | No | Image URL |
| Images | images | Media (multiple) | No | No | No | Product images |
| Weight | weight | Number (decimal) | No | No | No | Product weight |
| Length | length | Number (decimal) | No | No | No | Dimensions |
| Height | height | Number (decimal) | No | No | No | Dimensions |
| Width | width | Number (decimal) | No | No | No | Dimensions |
| Material | material | Short text | Yes | No | No | Material type |
| SEO Title | seoTitle | Short text | Yes | No | No | Meta title |
| SEO Description | seoDescription | Long text | Yes | No | No | Meta description |

**Option B: Via Migration Script (Recommended for automation)**

Create file: `scripts/contentful-migrations/product.js`

```javascript
module.exports = function(migration) {
  const product = migration.createContentType('product')
    .name('Product')
    .description('E-commerce product with localized content')
    .displayField('title');
  
  // Medusa reference (not localized, required, unique)
  product.createField('medusaId')
    .name('Medusa ID')
    .type('Symbol')
    .required(true)
    .localized(false)
    .validations([{ unique: true }]);
  
  // Title (localized)
  product.createField('title')
    .name('Title')
    .type('Symbol')
    .required(true)
    .localized(true);
  
  // Subtitle (localized)
  product.createField('subtitle')
    .name('Subtitle')
    .type('Symbol')
    .localized(true);
  
  // Description (localized, long text)
  product.createField('description')
    .name('Description')
    .type('Text')
    .localized(true);
  
  // Rich Description (localized, rich text)
  product.createField('richDescription')
    .name('Rich Description')
    .type('RichText')
    .localized(true);
  
  // Handle (not localized, unique)
  product.createField('handle')
    .name('Handle')
    .type('Symbol')
    .localized(false)
    .validations([{ unique: true }]);
  
  // Thumbnail URL
  product.createField('thumbnail')
    .name('Thumbnail')
    .type('Symbol')
    .localized(false);
  
  // Images (array of assets)
  product.createField('images')
    .name('Images')
    .type('Array')
    .items({
      type: 'Link',
      linkType: 'Asset'
    });
  
  // Dimensions
  product.createField('weight')
    .name('Weight')
    .type('Number');
  
  product.createField('length')
    .name('Length')
    .type('Number');
  
  product.createField('height')
    .name('Height')
    .type('Number');
  
  product.createField('width')
    .name('Width')
    .type('Number');
  
  // Material (localized)
  product.createField('material')
    .name('Material')
    .type('Symbol')
    .localized(true);
  
  // SEO fields (localized)
  product.createField('seoTitle')
    .name('SEO Title')
    .type('Symbol')
    .localized(true);
  
  product.createField('seoDescription')
    .name('SEO Description')
    .type('Text')
    .localized(true);
};
```

**Run migration:**
```bash
npm install -g contentful-cli

contentful login

contentful space migration \
  --space-id $CONTENTFUL_SPACE_ID \
  --environment-id master \
  scripts/contentful-migrations/product.js
```

### Step 3.2: Create Additional Content Types

Create these content types similarly:

**Product Variant:**
- medusaId (Symbol, required, unique)
- title (Symbol, localized)
- sku (Symbol)
- product (Reference to Product)
- Options, prices, etc.

**Product Collection:**
- medusaId (Symbol, required, unique)
- title (Symbol, localized)
- handle (Symbol, unique)
- description (Text, localized)

**Product Category:**
- medusaId (Symbol, required, unique)
- name (Symbol, localized)
- handle (Symbol, unique)
- description (Text, localized)
- parent (Reference to Product Category)

**Region:**
- medusaId (Symbol, required, unique)
- name (Symbol, localized)
- currencyCode (Symbol)
- countries (Array of symbols)

**See full migration scripts in the medusa-plugin-contentful repository.**

---

## Part 4: Set Up Webhooks

Webhooks enable Contentful → Medusa sync.

### Step 4.1: Ensure Backend is Publicly Accessible

**For Production:**
```
Your backend must be deployed and have a public URL
Example: https://api.yourstore.com
```

**For Development (using ngrok):**
```bash
# Install ngrok
brew install ngrok  # macOS
# Or download from https://ngrok.com

# Start Medusa
npx medusa develop

# In another terminal, tunnel port 9000
ngrok http 9000

# Note the forwarding URL:
# Forwarding: https://abc123.ngrok.io → http://localhost:9000
```

### Step 4.2: Configure Webhook in Contentful

**1. Open Webhook Settings**
```
Contentful Dashboard → Settings → Webhooks
Click "Add Webhook"
```

**2. Configure Webhook**
```
Name: Medusa Integration

URL: https://your-backend.com/hooks/contentful
(or https://abc123.ngrok.io/hooks/contentful for dev)

Method: POST

Custom headers (optional, for security):
  Key: X-Contentful-Webhook-Secret
  Value: your_secret_here_123

Triggers (select these):
  ☑ Entry: publish
  ☑ Entry: unpublish
  ☑ Entry: delete
  ☑ Entry: archive
  ☑ Entry: unarchive

Filters (optional):
  Content type: product (to only trigger for products)

Content type: application/json
```

**3. Save Webhook**

**4. Test Webhook**
```
In Contentful:
  1. Create or update a product entry
  2. Click "Publish"
  3. Go to Settings → Webhooks → [Your webhook]
  4. Check "Activity log"
  5. Should see successful POST (200 OK)
```

### Step 4.3: Secure Webhook Endpoint (Recommended)

**Add webhook secret to Medusa `.env`:**
```bash
CONTENTFUL_WEBHOOK_SECRET=your_secret_here_123
```

**Validate in webhook handler:**
```typescript
// In medusa-backend/src/api/webhooks/contentful.ts
export async function POST(req, res) {
  const secret = req.headers['x-contentful-webhook-secret'];
  
  if (secret !== process.env.CONTENTFUL_WEBHOOK_SECRET) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  // Process webhook...
}
```

---

## Part 5: Test the Integration

### Step 5.1: Start Medusa Backend

```bash
cd /path/to/medusa-backend
npx medusa develop

# Watch for initialization logs:
# info: Contentful plugin initialized
# info: Space ID: abc123, Environment: master
# info: Contentful subscriber registered
```

### Step 5.2: Test Medusa → Contentful Sync

**1. Create Product in Medusa Admin**
```
Open: http://localhost:7001
Login with admin credentials
Navigate to: Products → Add Product

Fill form:
  Title: "Test Product"
  Description: "This is a test"
  Handle: "test-product"
  
Click "Publish"
```

**2. Check Medusa Logs**
```bash
# Should see:
info: Product created: prod_xxxxx
info: Syncing product prod_xxxxx to Contentful
info: Successfully created product in Contentful (entry: xxxxx)
```

**3. Verify in Contentful**
```
Open: https://app.contentful.com
Navigate to: Content → Product
Should see: "Test Product" entry
Fields populated:
  - medusaId: prod_xxxxx ✓
  - title: "Test Product" ✓
  - description: "This is a test" ✓
```

**✅ Medusa → Contentful sync working!**

### Step 5.3: Test Contentful → Medusa Sync (Webhook)

**1. Update Product in Contentful**
```
Open "Test Product" entry
Change title: "Test Product" → "Updated Test Product"
Click "Save"
Click "Publish changes"
```

**2. Check Webhook Delivery**
```
Contentful → Settings → Webhooks → [Your webhook]
Activity log:
  - Latest POST to /hooks/contentful
  - Status: 200 OK ✓
```

**3. Check Medusa Logs**
```bash
# Should see:
info: Webhook received from Contentful
info: Updating product prod_xxxxx
info: Product updated successfully
```

**4. Verify in Medusa Admin**
```
Refresh product page in Medusa Admin
Title should be: "Updated Test Product" ✓
```

**✅ Contentful → Medusa sync working!**

### Step 5.4: Test Localization

**1. Add Spanish Translation in Contentful**
```
Open "Test Product" entry
Switch locale: en-US → es-ES
Fill fields:
  Title: "Producto de Prueba"
  Description: "Esta es una prueba"
Click "Save" and "Publish"
```

**2. Test Contentful Delivery API**
```bash
# Fetch English
curl "https://cdn.contentful.com/spaces/$SPACE_ID/entries?content_type=product&locale=en-US" \
  -H "Authorization: Bearer $DELIVERY_TOKEN"

# Should return: "Test Product"

# Fetch Spanish
curl "https://cdn.contentful.com/spaces/$SPACE_ID/entries?content_type=product&locale=es-ES" \
  -H "Authorization: Bearer $DELIVERY_TOKEN"

# Should return: "Producto de Prueba"
```

**✅ Localization working!**

---

## Part 6: Storefront Integration

### Option 1: Direct Dual Fetch

**Install Contentful SDK:**
```bash
cd /path/to/storefront
npm install contentful
```

**Create Contentful client:**
```typescript
// lib/contentful.ts
import { createClient } from 'contentful';

export const contentfulClient = createClient({
  space: process.env.NEXT_PUBLIC_CONTENTFUL_SPACE_ID!,
  accessToken: process.env.NEXT_PUBLIC_CONTENTFUL_DELIVERY_TOKEN!,
  environment: process.env.NEXT_PUBLIC_CONTENTFUL_ENV || 'master',
});
```

**Fetch product with content:**
```typescript
// lib/data.ts
import { medusaClient } from './medusa';
import { contentfulClient } from './contentful';

export async function getProduct(handle: string, locale: string = 'en-US') {
  // Fetch from Medusa
  const { products } = await medusaClient.products.list({ handle });
  const product = products[0];
  
  // Fetch from Contentful
  const contentfulResponse = await contentfulClient.getEntries({
    content_type: 'product',
    'fields.medusaId': product.id,
    locale: locale,
  });
  
  const content = contentfulResponse.items[0]?.fields;
  
  return {
    ...product,
    richDescription: content?.richDescription,
    images: content?.images,
    seoTitle: content?.seoTitle,
    seoDescription: content?.seoDescription,
  };
}
```

**Use in component:**
```typescript
// app/products/[handle]/page.tsx
import { getProduct } from '@/lib/data';

export default async function ProductPage({ 
  params 
}: { 
  params: { handle: string } 
}) {
  const product = await getProduct(params.handle, 'en-US');
  
  return (
    <div>
      <h1>{product.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: product.richDescription }} />
      {/* ... */}
    </div>
  );
}
```

### Option 2: Server-Side Merge (Next.js)

```typescript
// app/products/[handle]/page.tsx
import { medusaClient } from '@/lib/medusa';
import { contentfulClient } from '@/lib/contentful';

export async function generateMetadata({ params }: { params: { handle: string } }) {
  const [product, content] = await Promise.all([
    medusaClient.products.list({ handle: params.handle }),
    contentfulClient.getEntries({
      content_type: 'product',
      'fields.handle': params.handle,
      locale: 'en-US',
    }),
  ]);
  
  const contentFields = content.items[0]?.fields;
  
  return {
    title: contentFields?.seoTitle || product.products[0]?.title,
    description: contentFields?.seoDescription,
  };
}

export default async function ProductPage({ params }: { params: { handle: string } }) {
  const [productRes, contentRes] = await Promise.all([
    medusaClient.products.list({ handle: params.handle }),
    contentfulClient.getEntries({
      content_type: 'product',
      'fields.handle': params.handle,
      locale: 'en-US',
    }),
  ]);
  
  const product = productRes.products[0];
  const content = contentRes.items[0]?.fields;
  
  return (
    <ProductTemplate 
      product={product} 
      content={content} 
    />
  );
}
```

---

## Troubleshooting

### Issue 1: Plugin Not Loading

**Symptoms:**
```
No log: "Contentful plugin initialized"
Products not syncing
```

**Solutions:**

1. **Check plugin is in `medusa-config.js`:**
```javascript
plugins: [
  {
    resolve: `medusa-plugin-contentful`,
    options: { /* ... */ }
  }
]
```

2. **Verify environment variables:**
```bash
echo $CONTENTFUL_SPACE_ID
echo $CONTENTFUL_ACCESS_TOKEN
echo $CONTENTFUL_ENV
```

3. **Check for installation errors:**
```bash
npm list medusa-plugin-contentful
```

4. **Restart Medusa:**
```bash
npx medusa develop
```

### Issue 2: Products Not Syncing to Contentful

**Symptoms:**
```
Product created in Medusa
No entry appears in Contentful
```

**Solutions:**

1. **Check Redis is running:**
```bash
redis-cli ping
# Should return: PONG
```

2. **Check Medusa logs for errors:**
```bash
# Look for:
error: Failed to sync to Contentful: [error message]
```

3. **Verify API token has write permissions:**
```bash
# Test with curl
curl -X POST "https://api.contentful.com/spaces/$SPACE_ID/environments/master/entries" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/vnd.contentful.management.v1+json" \
  -H "X-Contentful-Content-Type: product" \
  -d '{"fields":{"medusaId":{"en-US":"test"}}}'
```

4. **Check content type exists:**
```
Contentful → Content model
Verify "Product" content type exists
Verify "medusaId" field exists
```

### Issue 3: Webhook Not Working

**Symptoms:**
```
Update in Contentful
No change in Medusa
```

**Solutions:**

1. **Check webhook activity log:**
```
Contentful → Settings → Webhooks → [Your webhook]
Activity log → Check latest delivery
Status should be 200 OK
```

2. **Verify URL is publicly accessible:**
```bash
curl -X POST https://your-backend.com/hooks/contentful \
  -H "Content-Type: application/json" \
  -d '{"test": "payload"}'
  
# Should return 200 OK (or validation error, not connection refused)
```

3. **Check Medusa logs:**
```bash
# Look for:
info: Webhook received from Contentful
# Or errors
```

4. **For local development, use ngrok:**
```bash
ngrok http 9000
# Update webhook URL to ngrok URL
```

### Issue 4: Localization Not Working

**Symptoms:**
```
Translations added in Contentful
Storefront shows English only
```

**Solutions:**

1. **Verify locale is configured:**
```
Contentful → Settings → Locales
Check locale exists (e.g., es-ES)
```

2. **Check API request includes locale:**
```typescript
const content = await contentfulClient.getEntries({
  content_type: 'product',
  locale: 'es-ES'  // ← Must specify locale
});
```

3. **Verify fields are marked as localized:**
```
Contentful → Content model → Product → [Field]
Settings → Localization → "Enable localization for this field"
```

### Issue 5: Rate Limit Exceeded

**Symptoms:**
```
error: Contentful API error: Rate limit exceeded
Sync stops after X products
```

**Solutions:**

1. **Check current rate limit:**
```
Free tier: 10 requests/second
Team tier: 98 requests/second
```

2. **Add delay between requests:**
```typescript
// In sync code
await syncProduct(product);
await sleep(100);  // 100ms delay = max 10 req/s
```

3. **Upgrade Contentful plan:**
```
Settings → Plan & billing → Upgrade
```

---

## Production Deployment

### Checklist

**Contentful:**
- [ ] Production space created
- [ ] Content types migrated
- [ ] Locales configured
- [ ] API tokens generated (management + delivery)
- [ ] Webhooks configured with production URL
- [ ] Rate limits reviewed (upgrade plan if needed)

**Medusa:**
- [ ] Plugin installed and configured
- [ ] Environment variables set (secrets stored securely)
- [ ] Redis configured and accessible
- [ ] Backend deployed and publicly accessible
- [ ] Webhook endpoint secured (secret validation)
- [ ] Logging configured (for monitoring syncs)

**Storefront:**
- [ ] Contentful SDK integrated
- [ ] Delivery API token configured (public-safe)
- [ ] Locale detection implemented
- [ ] Fallback locales configured
- [ ] Caching strategy implemented (Redis/CDN)
- [ ] Error handling for missing content

### Environment Variables

**Production `.env` (Medusa):**
```bash
# Contentful
CONTENTFUL_SPACE_ID=prod_space_id
CONTENTFUL_ACCESS_TOKEN=CFPAT-prod_management_token
CONTENTFUL_ENV=master
CONTENTFUL_WEBHOOK_SECRET=prod_webhook_secret_xyz

# Redis (required)
REDIS_URL=redis://prod-redis.example.com:6379
```

**Production `.env` (Storefront):**
```bash
# Contentful (public-safe delivery token)
NEXT_PUBLIC_CONTENTFUL_SPACE_ID=prod_space_id
NEXT_PUBLIC_CONTENTFUL_DELIVERY_TOKEN=delivery_token_xyz
NEXT_PUBLIC_CONTENTFUL_ENV=master
```

### Monitoring

**Set up monitoring for:**
- Sync failures (Medusa logs)
- Webhook delivery failures (Contentful activity log)
- API rate limit warnings
- Redis connectivity
- Performance (sync latency)

---

## What's Next?

- **[Data Flow](./03-data-flow.md)** - See sync examples in action
- **[Pros & Cons](./05-pros-cons.md)** - Evaluate the integration
- **[FAQ](./FAQ.md)** - Common questions
- **[SUMMARY](./SUMMARY.md)** - Quick reference guide

