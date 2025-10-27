# Implementation Guide

This guide walks you through setting up the Medusa-Strapi integration from scratch.

## 🚨 Critical: Read This First

**IMPORTANT**: Before starting, understand which packages can be installed from npm vs which must be cloned:

| Component | Installation Method | Repository |
|-----------|---------------------|------------|
| **Medusa Plugin** (`medusa-plugin-strapi-ts`) | ✅ **npm install** | `yarn add medusa-plugin-strapi-ts` |
| **Strapi Application** (`medusa-strapi`) | ❌ **Must clone repo** | Clone [SGFGOV](https://github.com/SGFGOV/medusa-strapi-repo) or [Fork](https://github.com/dannycdannydo/medusa-strapi-repo) |
| **Strapi Plugins** | ✅ npm or included in monorepo | Already in cloned repo |

**Why?**
- The **Medusa plugin** is published to npm and identical in both repositories
- The **Strapi application** is NOT published to npm (marked as `"private": true`)
- The fork has 45 additional commits with custom content types and features

**See**: [Complete Package Installation Guide](./PACKAGE-VERSIONS.md) for detailed breakdown.

**Choose Your Path**:
- **Original Repository** → Simple setup, core features only
- **Fork Repository** → Production-ready with 14 additional content types

---

## Prerequisites

Before you begin, ensure you have:

### Required Software

- **Node.js** v16.17.2+ (v18 or v20 recommended)
- **npm** or **yarn** or **pnpm**
- **PostgreSQL** v12+ (two databases needed)
- **Redis** v6+ (for Medusa event bus)
- **Git** (to clone repository)

### Existing Medusa Installation

- Medusa v1.8+ backend
- Admin dashboard accessible
- Database configured and migrated

### Knowledge Requirements

- Basic command line usage
- Environment variable configuration
- Basic PostgreSQL administration
- REST API concepts

### Recommended

- Docker (for easier PostgreSQL/Redis setup)
- Database GUI tool (TablePlus, pgAdmin, DBeaver)
- API testing tool (Postman, Insomnia)

---

## Part 1: Set Up Strapi Server

### Step 1.1: Clone Strapi Template

The official template includes pre-configured content types and plugins.

```bash
# Clone the repository
git clone https://github.com/SGFGOV/medusa-strapi-repo.git

# Navigate to Strapi package
cd medusa-strapi-repo/packages/medusa-strapi

# Check files
ls -la
# Should see: config/, src/, package.json, .env.test
```

### Step 1.2: Create Strapi Database

```bash
# Connect to PostgreSQL
psql -U postgres

# Create database
CREATE DATABASE strapi_medusa;

# Create user (optional, for isolation)
CREATE USER strapi_user WITH PASSWORD 'your_secure_password';

# Grant privileges
GRANT ALL PRIVILEGES ON DATABASE strapi_medusa TO strapi_user;

# Exit
\q
```

**Verify:**
```bash
psql -U strapi_user -d strapi_medusa -c "SELECT 1;"
# Should return: 1
```

### Step 1.3: Configure Environment Variables

```bash
# Copy example environment file
cp .env.test .env

# Edit with your preferred editor
nano .env
# or
vim .env
# or
code .env
```

**Required Variables:**

```bash
# ============================================
# IMPORTANT: Security Keys (CHANGE THESE!)
# ============================================
# Generate random strings for production
# Use: openssl rand -base64 32

APP_KEYS=key1,key2,key3,key4  # 4 random strings
API_TOKEN_SALT=your_random_salt  # Random string
ADMIN_JWT_SECRET=your_admin_jwt_secret  # Random string
JWT_SECRET=your_jwt_secret  # Random string (CRITICAL)

# ============================================
# Medusa Integration (CRITICAL)
# ============================================
MEDUSA_STRAPI_SECRET=your_jwt_secret  # MUST MATCH Medusa JWT_SECRET
MEDUSA_BACKEND_URL=http://localhost:9000
MEDUSA_BACKEND_ADMIN=http://localhost:7001

# ============================================
# Strapi Super Admin (First-Time Setup)
# ============================================
SUPERUSER_EMAIL=admin@yourdomain.com
SUPERUSER_USERNAME=SuperUser
SUPERUSER_PASSWORD=YourSecurePassword123!
SUPERUSER_FIRSTNAME=Admin
SUPERUSER_LASTNAME=User

# ============================================
# Database Configuration
# ============================================
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=strapi_medusa
DATABASE_USERNAME=strapi_user
DATABASE_PASSWORD=your_secure_password
DATABASE_SSL=false
DATABASE_SCHEMA=public

# ============================================
# Server Configuration
# ============================================
HOST=0.0.0.0
PORT=1337
NODE_ENV=development
```

**Generate Secure Keys:**
```bash
# On Linux/Mac
openssl rand -base64 32

# Or using Node.js
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"

# Generate 4 keys and set as APP_KEYS=key1,key2,key3,key4
```

**⚠️ CRITICAL:** The `MEDUSA_STRAPI_SECRET` MUST be the same as your Medusa `JWT_SECRET`. This allows both systems to verify tokens.

### Step 1.4: Install Dependencies

```bash
# In medusa-strapi-repo root
yarn install

# Or if you prefer npm
npm install
```

**Expected Output:**
```
[1/4] Resolving packages...
[2/4] Fetching packages...
[3/4] Linking dependencies...
[4/4] Building fresh packages...
success Saved lockfile.
Done in 45.3s
```

### Step 1.5: Build Packages

```bash
# Build all packages in monorepo
yarn build

# Or
npm run build
```

This builds:
- `medusa-plugin-strapi-ts`
- `strapi-plugin-medusajs`
- `strapi-plugin-sso-medusa`
- `strapi-plugin-multi-country-select`

**Expected Output:**
```
Building medusa-plugin-strapi-ts...
Building strapi-plugin-medusajs...
Building strapi-plugin-sso-medusa...
Building strapi-plugin-multi-country-select...
Build complete ✓
```

### Step 1.6: Start Strapi (Development Mode)

```bash
# Navigate to Strapi package
cd packages/medusa-strapi

# Start in development mode
yarn develop

# Or
npm run develop
```

**First-Time Startup Process:**

```
Building admin panel...
⠹ Creating admin...
✔ Building build context
✔ Creating admin panel
[2024-01-20 10:30:00.000] info: Database connected
[2024-01-20 10:30:01.000] info: Running migrations...
[2024-01-20 10:30:05.000] info: Migrations completed
[2024-01-20 10:30:06.000] info: Strapi plugin medusajs initialized
[2024-01-20 10:30:07.000] info: Super admin user created
[2024-01-20 10:30:08.000] info: Server listening on http://localhost:1337
```

**Verify Strapi is Running:**

```bash
# Health check
curl http://localhost:1337/_health
# Returns: 204 No Content (success)

# Admin panel
open http://localhost:1337/admin
# Opens browser to Strapi admin login
```

**Login to Strapi Admin:**
- URL: `http://localhost:1337/admin`
- Email: Value of `SUPERUSER_EMAIL` from `.env`
- Password: Value of `SUPERUSER_PASSWORD` from `.env`

### Step 1.7: Verify Content Types

After logging in:

1. **Navigate to:** Content-Type Builder (left sidebar)
2. **Check Collection Types exist:**
   - Products
   - Product Variants
   - Product Collections
   - Product Categories
   - Regions
   - ... (20+ types)

3. **Check Product Schema:**
   - Click "Product" in Content-Type Builder
   - Verify fields:
     - `medusa_id` (Text, Required, Unique)
     - `title` (Text)
     - `subtitle` (Text)
     - `description` (Text)
     - `handle` (Text, Unique)
     - Relations to variants, collection, categories, etc.

**If content types are missing:**
```bash
# Stop Strapi (Ctrl+C)
# Check database
psql -U strapi_user -d strapi_medusa -c "\dt"
# Should see tables like: products, product_variants, etc.

# If missing, rebuild admin
yarn build --clean
yarn develop
```

---

## Part 2: Configure Medusa Backend

### Step 2.1: Install Plugin

```bash
# Navigate to your Medusa backend directory
cd /path/to/your/medusa-backend

# Install plugin
yarn add medusa-plugin-strapi-ts

# Or
npm install medusa-plugin-strapi-ts
```

**Verify Installation:**
```bash
ls node_modules/ | grep medusa-plugin-strapi-ts
# Returns: medusa-plugin-strapi-ts
```

### Step 2.2: Configure Plugin

Edit `medusa-config.js`:

```javascript
// medusa-config.js

// ... existing config ...

const plugins = [
  // ... existing plugins ...
  
  {
    resolve: "medusa-plugin-strapi-ts",
    options: {
      // Strapi server details
      strapi_protocol: process.env.STRAPI_PROTOCOL || "http",
      strapi_host: process.env.STRAPI_SERVER_HOSTNAME || "localhost",
      strapi_port: process.env.STRAPI_PORT || 1337,
      
      // CRITICAL: Must match Strapi MEDUSA_STRAPI_SECRET
      strapi_secret: process.env.STRAPI_SECRET,
      
      // Optional: Public key for JWT verification
      strapi_public_key: process.env.STRAPI_PUBLIC_KEY,
      
      // Service account (will be created in Strapi)
      strapi_default_user: {
        username: process.env.STRAPI_MEDUSA_USER || "medusa",
        password: process.env.STRAPI_MEDUSA_PASSWORD,
        email: process.env.STRAPI_MEDUSA_EMAIL || "medusa@service.local",
        confirmed: true,
        blocked: false,
        provider: "local"
      },
      
      // Strapi super admin (for initial setup)
      strapi_admin: {
        username: process.env.STRAPI_SUPER_USERNAME || "SuperUser",
        password: process.env.STRAPI_SUPER_PASSWORD,
        email: process.env.STRAPI_SUPER_USER_EMAIL || "admin@yourdomain.com"
      },
      
      // Behavior options
      auto_start: true,  // Automatically connect on Medusa startup
      sync_on_init: false,  // Set true for initial bulk sync (slow!)
      
      // Advanced options
      strapi_ignore_threshold: 3,  // Seconds to prevent sync loops
      encryption_algorithm: "aes-256-cbc",
      strapi_healthcheck_timeout: 120000,  // 2 minutes
      max_page_size: 100
    }
  }
];

module.exports = {
  projectConfig: {
    // ... existing config ...
    redis_url: process.env.REDIS_URL || "redis://localhost:6379",  // REQUIRED!
  },
  plugins
};
```

### Step 2.3: Configure Environment Variables

Add to `.env` in Medusa backend:

```bash
# ============================================
# Strapi Integration
# ============================================
STRAPI_PROTOCOL=http
STRAPI_SERVER_HOSTNAME=localhost
STRAPI_PORT=1337

# CRITICAL: Must match Strapi JWT_SECRET
STRAPI_SECRET=your_jwt_secret  # Same as Strapi JWT_SECRET/MEDUSA_STRAPI_SECRET

# Service Account (created automatically in Strapi)
STRAPI_MEDUSA_USER=medusa
STRAPI_MEDUSA_PASSWORD=SecurePassword123!
STRAPI_MEDUSA_EMAIL=medusa@service.local

# Super Admin Credentials (same as Strapi)
STRAPI_SUPER_USERNAME=SuperUser
STRAPI_SUPER_PASSWORD=YourSecurePassword123!
STRAPI_SUPER_USER_EMAIL=admin@yourdomain.com

# ============================================
# Redis (REQUIRED for event bus)
# ============================================
REDIS_URL=redis://localhost:6379

# ============================================
# Existing Medusa Config
# ============================================
# Your existing variables...
```

**⚠️ CRITICAL CHECKS:**

1. **Redis must be running:**
   ```bash
   redis-cli ping
   # Returns: PONG
   ```

2. **Secrets must match:**
   ```bash
   # Medusa
   STRAPI_SECRET=your_jwt_secret
   
   # Strapi
   JWT_SECRET=your_jwt_secret
   MEDUSA_STRAPI_SECRET=your_jwt_secret
   ```

3. **Super admin credentials must match Strapi:**
   ```bash
   # Both systems need same credentials for initial setup
   ```

### Step 2.4: Start Medusa

```bash
# Development mode
npx medusa develop

# Or production mode
npx medusa start
```

**Expected Startup Logs:**

```
info: Initializing Medusa...
info: Connected to PostgreSQL
info: Connected to Redis
info: Loading plugins...
info: Plugin medusa-plugin-strapi-ts loaded
info: Checking Strapi Health
debug: check-url: http://localhost:1337/_health
info: Strapi is healthy
info: Strapi Subscriber Initialized
info: Attempting to register default Medusa user in Strapi...
info: Successfully registered default user
info: Medusa server listening on port 9000
```

**⚠️ If you see errors:**

```
error: Unable to connect to Strapi
```
→ Check Strapi is running: `curl http://localhost:1337/_health`

```
error: Authentication failed
```
→ Check secrets match in both .env files

```
error: Redis connection failed
```
→ Check Redis is running: `redis-cli ping`

---

## Part 3: Test the Integration

### Test 3.1: Create a Product in Medusa

1. **Open Medusa Admin:**
   ```
   http://localhost:7001
   ```

2. **Navigate to:** Products → New Product

3. **Fill Form:**
   ```
   Title: "Test Integration Product"
   Subtitle: "Testing Strapi sync"
   Description: "This product tests the integration"
   Handle: "test-integration-product"
   ```

4. **Add Variant:**
   ```
   Title: "Default"
   SKU: "TEST-001"
   Price: $10.00
   ```

5. **Click:** "Publish"

### Test 3.2: Verify in Medusa Logs

Watch Medusa console for sync logs:

```bash
# You should see:
info: Strapi Subscriber Initialized
info: creating product in strapi - {"medusa_id":"prod_01...","title":"Test Integration Product",...}
info: User Endpoint: http://localhost:1337/api/products
info: POST http://localhost:1337/api/products
debug: Strapi Ok : method: POST, id:, type:products, :status:201
info: Successfully created product in Strapi
```

### Test 3.3: Verify in Strapi Admin

1. **Open Strapi Admin:**
   ```
   http://localhost:1337/admin
   ```

2. **Navigate to:** Content Manager → Products

3. **Verify:**
   - "Test Integration Product" appears in list
   - Click to open
   - Check fields:
     - `medusa_id`: `prod_01...` (from Medusa)
     - `title`: "Test Integration Product"
     - `subtitle`: "Testing Strapi sync"
     - `description`: "This product tests the integration"
     - `handle`: "test-integration-product"
   - Check relations:
     - `product_variants`: 1 variant linked

**✅ Success!** Product synced from Medusa to Strapi.

### Test 3.4: Enrich Content in Strapi

1. **In Strapi Admin**, edit "Test Integration Product"

2. **Add Custom Content:**
   - Rich Description: "This is a **test product** with _rich formatting_."
   - Upload an image
   - Add custom field values

3. **Click:** "Save"

4. **Verify in Medusa:**
   - Open product in Medusa Admin
   - Title/subtitle remain unchanged (expected)
   - Rich content stays in Strapi only (expected)

**✅ Success!** Content enrichment working.

### Test 3.5: Update Product in Medusa

1. **In Medusa Admin**, edit "Test Integration Product"

2. **Change:**
   - Title: "Test Integration Product (Updated)"

3. **Click:** "Save"

4. **Check Strapi:**
   - Title updates to "Test Integration Product (Updated)"
   - Rich description preserved
   - Custom fields preserved

**✅ Success!** Sync updating correctly.

### Test 3.6: Delete Product

⚠️ **Warning:** This permanently deletes from both systems.

1. **In Medusa Admin**, delete "Test Integration Product"

2. **Verify in Strapi:**
   - Product removed from list

**✅ Success!** Deletion propagated.

---

## Part 4: Bulk Sync (Optional)

If you have existing products in Medusa, sync them all at once.

### Step 4.1: Enable Sync on Init

```javascript
// medusa-config.js
{
  resolve: "medusa-plugin-strapi-ts",
  options: {
    ...
    sync_on_init: true,  // ← Enable this
    auto_start: true
  }
}
```

### Step 4.2: Restart Medusa

```bash
npx medusa develop
```

**Watch Logs:**
```
info: Medusa server starting...
info: Connected to Strapi
info: Initiating bulk synchronization...
info: Syncing products... (0/100)
info: Syncing products... (50/100)
info: Syncing products... (100/100)
info: Syncing collections...
info: Syncing categories...
info: Syncing regions...
info: Sync complete ✓
info: Medusa server listening on port 9000
```

**⚠️ Notes:**
- This can take 30+ minutes for large catalogs
- Set `sync_on_init: false` after initial sync
- Run during low-traffic periods

### Step 4.3: Verify Sync

```bash
# Check Strapi database
psql -U strapi_user -d strapi_medusa

# Count products
SELECT COUNT(*) FROM products;

# Check sample product
SELECT medusa_id, title FROM products LIMIT 5;

# Exit
\q
```

**Or via API:**
```bash
curl "http://localhost:1337/api/products?pagination[pageSize]=1"
# Check meta.pagination.total
```

---

## Part 5: Integrate Storefront

### Option A: Dual Fetch (Direct to Both APIs)

**Install Clients:**
```bash
cd your-storefront
yarn add @medusajs/medusa-js axios
```

**Fetch Function:**
```typescript
// lib/data.ts
import Medusa from "@medusajs/medusa-js";
import axios from "axios";

const medusa = new Medusa({
  baseUrl: process.env.NEXT_PUBLIC_MEDUSA_URL || "http://localhost:9000",
  maxRetries: 3
});

const STRAPI_URL = process.env.NEXT_PUBLIC_STRAPI_URL || "http://localhost:1337";

export async function getProductWithContent(handle: string) {
  // Fetch commerce data from Medusa
  const { product } = await medusa.products.list({
    handle
  });
  
  if (!product) return null;
  
  // Fetch content from Strapi
  try {
    const strapiRes = await axios.get(
      `${STRAPI_URL}/api/products`,
      {
        params: {
          filters: {
            medusa_id: {
              $eq: product.id
            }
          },
          populate: '*'
        }
      }
    );
    
    const content = strapiRes.data.data[0]?.attributes;
    
    return {
      ...product,
      richDescription: content?.rich_description,
      images: content?.images,
      careInstructions: content?.care_instructions,
      // ... other Strapi fields
    };
  } catch (error) {
    console.error("Failed to fetch Strapi content:", error);
    return product;  // Return Medusa data only
  }
}
```

**Component:**
```tsx
// app/products/[handle]/page.tsx
import { getProductWithContent } from "@/lib/data";

export default async function ProductPage({ params }) {
  const product = await getProductWithContent(params.handle);
  
  return (
    <div>
      <h1>{product.title}</h1>
      <p>{product.subtitle}</p>
      
      {/* From Strapi */}
      {product.richDescription && (
        <div dangerouslySetInnerHTML={{ __html: product.richDescription }} />
      )}
      
      {/* From Medusa */}
      <p>Price: ${product.variants[0].prices[0].amount / 100}</p>
      <button>Add to Cart</button>
    </div>
  );
}
```

### Option B: Via Medusa Proxy

**Fetch Function:**
```typescript
// lib/data.ts
export async function getProductWithContent(handle: string) {
  const res = await fetch(
    `${MEDUSA_URL}/strapi/content/products/${productId}`
  );
  
  return await res.json();
}
```

**Simpler, but requires Medusa proxy endpoint implementation.**

---

## Troubleshooting

### Issue: Products not syncing to Strapi

**Symptoms:**
- No logs in Medusa console
- Product doesn't appear in Strapi

**Checks:**

1. **Redis running?**
   ```bash
   redis-cli ping
   # Should return: PONG
   ```

2. **Event bus configured?**
   ```javascript
   // medusa-config.js
   projectConfig: {
     redis_url: process.env.REDIS_URL
   }
   ```

3. **Subscriber loaded?**
   ```bash
   # Check Medusa logs for:
   info: Strapi Subscriber Initialized
   ```

4. **Strapi reachable?**
   ```bash
   curl http://localhost:1337/_health
   # Should return: 204 No Content
   ```

5. **Secrets match?**
   ```bash
   # In Medusa .env
   echo $STRAPI_SECRET
   
   # In Strapi .env
   echo $JWT_SECRET
   echo $MEDUSA_STRAPI_SECRET
   
   # All three should be identical
   ```

**Fix:**
```bash
# Restart both services
# In Strapi terminal: Ctrl+C, then yarn develop
# In Medusa terminal: Ctrl+C, then npx medusa develop
```

### Issue: Authentication errors

**Symptoms:**
```
error: Unable to login to Strapi
error: 401 Unauthorized
```

**Fix:**

1. **Check service account exists in Strapi:**
   - Open Strapi Admin
   - Settings → Users & Permissions Plugin → Users
   - Look for user with email matching `STRAPI_MEDUSA_EMAIL`
   - If missing, restart Medusa (will auto-create)

2. **Check credentials:**
   ```bash
   # Medusa .env
   STRAPI_MEDUSA_EMAIL=medusa@service.local
   STRAPI_MEDUSA_PASSWORD=YourPassword

   # Test login manually
   curl -X POST http://localhost:1337/api/auth/local \
     -H "Content-Type: application/json" \
     -d '{
       "identifier": "medusa@service.local",
       "password": "YourPassword"
     }'
   
   # Should return: {"jwt": "...", "user": {...}}
   ```

3. **Check super admin:**
   ```bash
   # If service account creation fails, check super admin exists
   # Strapi Admin → Settings → Administration Panel → Users
   # Should see user matching SUPERUSER_EMAIL
   ```

### Issue: Sync loop (endless updates)

**Symptoms:**
- Product keeps updating
- Logs show rapid sync events

**Cause:** Ignore flags not working (Redis issue)

**Fix:**

1. **Check Redis connection:**
   ```bash
   redis-cli
   > KEYS *_ignore_*
   # Should show ignore keys with TTL
   > TTL prod_123_ignore_strapi
   # Should return: 2 or 1 (seconds remaining)
   ```

2. **Increase threshold:**
   ```javascript
   // medusa-config.js
   strapi_ignore_threshold: 10,  // Increase from 3 to 10 seconds
   ```

### Issue: Strapi admin panel not loading

**Symptoms:**
- `/admin` returns 404 or blank page

**Fix:**

1. **Rebuild admin panel:**
   ```bash
   cd packages/medusa-strapi
   yarn build --clean
   yarn develop
   ```

2. **Check NODE_ENV:**
   ```bash
   # .env
   NODE_ENV=development  # Not 'production' for local dev
   ```

3. **Clear cache:**
   ```bash
   rm -rf .cache
   rm -rf build
   yarn develop
   ```

### Issue: CORS errors on storefront

**Symptoms:**
```
Access to fetch at 'http://localhost:1337/api/products' from origin
'http://localhost:3000' has been blocked by CORS policy
```

**Fix:**

Edit `config/middlewares.js` in Strapi:

```javascript
module.exports = [
  'strapi::errors',
  {
    name: 'strapi::security',
    config: {
      contentSecurityPolicy: {
        useDefaults: true,
        directives: {
          'connect-src': ["'self'", 'https:'],
          'img-src': ["'self'", 'data:', 'blob:', 'https:'],
          'media-src': ["'self'", 'data:', 'blob:', 'https:'],
          upgradeInsecureRequests: null,
        },
      },
    },
  },
  {
    name: 'strapi::cors',
    config: {
      origin: [
        'http://localhost:3000',  // Your storefront
        'http://localhost:9000',  // Medusa
        'http://localhost:7001',  // Medusa Admin
      ],
      methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'HEAD', 'OPTIONS'],
      headers: ['Content-Type', 'Authorization', 'Origin', 'Accept'],
      keepHeaderOnError: true,
    },
  },
  'strapi::poweredBy',
  'strapi::logger',
  'strapi::query',
  'strapi::body',
  'strapi::session',
  'strapi::favicon',
  'strapi::public',
];
```

Restart Strapi after changing.

---

## Production Deployment

### Environment Variables

**Never commit to version control:**
- API keys and secrets
- Database passwords
- JWT secrets

**Use environment-specific configs:**

```bash
# Development
STRAPI_SERVER_HOSTNAME=localhost

# Production
STRAPI_SERVER_HOSTNAME=strapi.yourdomain.com
STRAPI_PROTOCOL=https
```

### Database

**Use managed PostgreSQL:**
- AWS RDS
- Google Cloud SQL
- DigitalOcean Managed Database
- Supabase

**Enable SSL:**
```bash
DATABASE_SSL=true
```

### Redis

**Use managed Redis:**
- AWS ElastiCache
- Redis Cloud
- Upstash

**Connection string:**
```bash
REDIS_URL=rediss://user:password@host:port
```

### Strapi Deployment

**Options:**
- Strapi Cloud (easiest)
- AWS EC2 + RDS
- DigitalOcean Droplet
- Heroku
- Vercel (with serverless Strapi)

**Build command:**
```bash
yarn build
NODE_ENV=production yarn start
```

### Medusa Deployment

Follow Medusa deployment guide:
- Railway
- Heroku
- DigitalOcean
- AWS

**Ensure Redis URL is set correctly.**

### Health Checks

**Strapi:**
```bash
curl https://strapi.yourdomain.com/_health
```

**Medusa:**
```bash
curl https://api.yourdomain.com/health
```

### Monitoring

**Set up logging:**
- Strapi → Console logs or file
- Medusa → Console logs or file
- Use log aggregation (Logtail, Papertrail)

**Set up alerts:**
- Database connection failures
- Redis connection failures
- Sync failures
- High error rates

---

## Part 7: Optional Enhancements (Production Fork)

The `medusa-with-strapi-dannycdannydo` fork includes production-ready enhancements and additional features. This section documents how to implement these optional features.

### 7.1: Cloudinary Image Hosting

Alternative to AWS S3 for image uploads with powerful transformations.

**Benefits:**
- Automatic image optimization
- On-the-fly transformations
- CDN delivery included
- Generous free tier

**Installation:**

```bash
cd packages/medusa-strapi
yarn add @strapi/provider-upload-cloudinary
```

**Configuration:**

Create or edit `config/plugins.js`:

```javascript
module.exports = ({ env }) => ({
  upload: {
    config: {
      provider: 'cloudinary',
      providerOptions: {
        cloud_name: env('CLOUDINARY_NAME'),
        api_key: env('CLOUDINARY_KEY'),
        api_secret: env('CLOUDINARY_SECRET'),
      },
      actionOptions: {
        upload: {},
        uploadStream: {},
        delete: {},
      },
    },
  },
});
```

**Environment Variables:**

```bash
CLOUDINARY_NAME=your_cloud_name
CLOUDINARY_KEY=your_api_key
CLOUDINARY_SECRET=your_api_secret
```

**Get Credentials:**
1. Sign up at [cloudinary.com](https://cloudinary.com)
2. Dashboard → Account Details
3. Copy Cloud Name, API Key, API Secret

### 7.2: Mailchimp Integration

Marketing automation and email campaigns for Medusa.

**Installation:**

```bash
cd /path/to/medusa-backend
yarn add medusa-plugin-mailchimp
```

**Medusa Configuration:**

Add to `medusa-config.js`:

```javascript
const plugins = [
  // ... other plugins
  {
    resolve: 'medusa-plugin-mailchimp',
    options: {
      api_key: process.env.MAILCHIMP_API_KEY,
      newsletter_list_id: process.env.MAILCHIMP_NEWSLETTER_LIST_ID,
    },
  },
];
```

**Environment Variables:**

```bash
MAILCHIMP_API_KEY=your_mailchimp_api_key
MAILCHIMP_NEWSLETTER_LIST_ID=your_list_id
```

**Features:**
- Subscribe customers to newsletter
- Sync customer data
- Trigger campaigns on events
- Abandoned cart emails

### 7.3: SendGrid Email Integration

Transactional emails for Medusa events.

**Installation:**

```bash
cd packages/medusa-strapi
yarn add medusa-plugin-sendgrid
```

**Strapi Configuration:**

Add to Medusa `medusa-config.js`:

```javascript
const plugins = [
  {
    resolve: 'medusa-plugin-sendgrid',
    options: {
      api_key: process.env.SENDGRID_API_KEY,
      from: process.env.SENDGRID_FROM,
      order_placed_template: 'order-placed-template-id',
    },
  },
];
```

**Environment Variables:**

```bash
SENDGRID_API_KEY=your_sendgrid_api_key
SENDGRID_FROM=noreply@yourdomain.com
```

### 7.4: Custom Homepage Content Types

The production fork includes pre-built content types for homepage management.

**Available Content Types:**
- `homepage-banner` - Hero banners with images and CTAs
- `homepage-feature` - Feature showcases
- `homepage-featured-section-one` - Promotional section #1
- `homepage-featured-section-two` - Promotional section #2
- `homepage-three-offer` - Three-column promotional offers
- `homepage-triple-point` - USP/benefits bullets

**Implementation:**

1. **Copy content types from fork:**

```bash
# From the dannycdannydo fork location
cp -r packages/medusa-strapi/src/api/homepage-* \
  /your-strapi-project/src/api/
```

2. **Restart Strapi to register content types:**

```bash
yarn develop
```

3. **Access in Strapi Admin:**
   - Navigate to Content Manager
   - Find new content types under "Collection Types"
   - Create entries

4. **Fetch in storefront:**

```typescript
// Fetch homepage banners
const response = await fetch('http://localhost:1337/api/homepage-banners?populate=*');
const { data } = await response.json();

// Use in your components
data.forEach(banner => {
  console.log(banner.attributes.title);
  console.log(banner.attributes.image.data.attributes.url);
});
```

### 7.5: Policy & Legal Pages

Manage terms of service, privacy policy, and legal content.

**Content Type Structure:**

```typescript
// api/policy/content-types/policy/schema.json
{
  "kind": "collectionType",
  "collectionName": "policies",
  "info": {
    "singularName": "policy",
    "pluralName": "policies",
    "displayName": "Policy"
  },
  "options": {
    "draftAndPublish": true
  },
  "attributes": {
    "title": { "type": "string", "required": true },
    "slug": { "type": "uid", "targetField": "title" },
    "content": { "type": "richtext" },
    "type": {
      "type": "enumeration",
      "enum": ["privacy", "terms", "refund", "shipping", "cookie"]
    }
  }
}
```

**Copy from fork:**

```bash
cp -r packages/medusa-strapi/src/api/policy \
  /your-strapi-project/src/api/
```

**Fetch in storefront:**

```typescript
// Get privacy policy
const response = await fetch('http://localhost:1337/api/policies?filters[slug][$eq]=privacy-policy');
const { data } = await response.json();
```

### 7.6: Website Maintenance Mode

Toggle site-wide maintenance mode from Strapi Admin.

**Implementation:**

1. **Copy content type:**

```bash
cp -r packages/medusa-strapi/src/api/website-maintenance-mode \
  /your-strapi-project/src/api/
```

2. **Create single entry in Strapi Admin:**
   - Maintenance enabled: Yes/No
   - Maintenance message: "We're upgrading..."
   - Estimated return time

3. **Check in storefront middleware:**

```typescript
// Next.js middleware example
import { NextResponse } from 'next/server';

export async function middleware(request) {
  const maintenanceResponse = await fetch(
    'http://localhost:1337/api/website-maintenance-mode'
  );
  const { data } = await maintenanceResponse.json();
  
  if (data.attributes.enabled) {
    // Redirect to maintenance page
    return NextResponse.rewrite(new URL('/maintenance', request.url));
  }
  
  return NextResponse.next();
}
```

### 7.7: TypeScript Environment Configurations

Production-grade environment-specific database configs.

**Structure:**

```
config/
├── env/
│   ├── development/
│   │   └── database.ts
│   ├── production/
│   │   └── database.ts
│   └── test/
│       └── database.js
```

**Example (production):**

```typescript
// config/env/production/database.ts
export default ({ env }) => ({
  connection: {
    client: 'postgres',
    connection: {
      host: env('DATABASE_HOST'),
      port: env.int('DATABASE_PORT', 5432),
      database: env('DATABASE_NAME'),
      user: env('DATABASE_USERNAME'),
      password: env('DATABASE_PASSWORD'),
      ssl: env.bool('DATABASE_SSL', true), // Force SSL in production
      pool: {
        min: env.int('DATABASE_POOL_MIN', 2),
        max: env.int('DATABASE_POOL_MAX', 10),
      },
    },
    debug: false,
  },
});
```

**Example (development):**

```typescript
// config/env/development/database.ts
export default ({ env }) => ({
  connection: {
    client: 'postgres',
    connection: {
      host: env('DATABASE_HOST', 'localhost'),
      port: env.int('DATABASE_PORT', 5432),
      database: env('DATABASE_NAME', 'strapi_dev'),
      user: env('DATABASE_USERNAME', 'postgres'),
      password: env('DATABASE_PASSWORD', 'postgres'),
      ssl: env.bool('DATABASE_SSL', false),
    },
    debug: true, // Enable query logging in dev
  },
});
```

**Benefits:**
- Type safety with TypeScript
- Environment-specific optimizations
- Connection pool tuning
- SSL enforcement in production

### 7.8: Team Member Profiles

Manage team/staff profiles with photos and bios.

**Copy from fork:**

```bash
cp -r packages/medusa-strapi/src/api/team-member \
  /your-strapi-project/src/api/
```

**Use case:**
- About Us page
- Customer support team
- Store staff directory

### 7.9: Contact Form Submissions

Store contact form submissions in Strapi.

**Copy from fork:**

```bash
cp -r packages/medusa-strapi/src/api/contact-us \
  /your-strapi-project/src/api/
```

**Storefront implementation:**

```typescript
// Submit contact form
const response = await fetch('http://localhost:1337/api/contact-us', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    data: {
      name: 'John Doe',
      email: 'john@example.com',
      message: 'I have a question...',
      phone: '+1234567890',
    },
  }),
});
```

### 7.10: Enhanced Image Management

Specialized image content types for different site sections.

**Available Types:**
- `general-image` - Site-wide reusable images
- `bundle-main-image` - Product bundle hero images
- `all-products-image` - Catalog/collection page images

**Copy from fork:**

```bash
cp -r packages/medusa-strapi/src/api/general-image \
  /your-strapi-project/src/api/
cp -r packages/medusa-strapi/src/api/bundle-main-image \
  /your-strapi-project/src/api/
cp -r packages/medusa-strapi/src/api/all-products-image \
  /your-strapi-project/src/api/
```

---

## Summary: Base vs Enhanced Setup

### Base Setup (Original Repository)
- ✅ Core Medusa-Strapi sync
- ✅ Product, variant, collection sync
- ✅ Minimal content types (27 core entities)
- ✅ S3 upload only
- ✅ Basic configuration

**Best for:** Learning, minimal setup, simple e-commerce

### Enhanced Setup (Production Fork)
- ✅ Everything in base
- ✅ 14 additional custom content types
- ✅ Multiple upload providers (S3 + Cloudinary)
- ✅ Email marketing (Mailchimp)
- ✅ Transactional emails (SendGrid)
- ✅ Homepage management
- ✅ Policy pages
- ✅ Team profiles
- ✅ Maintenance mode
- ✅ TypeScript configs

**Best for:** Production sites, content-heavy stores, marketing-focused e-commerce

---

## What's Next?

- **[Pros & Cons](./05-pros-cons.md)** - Evaluate trade-offs
- **[FAQ](./FAQ.md)** - Common questions
- **[Data Flow](./03-data-flow.md)** - Understand sync scenarios
- **[Architecture](./02-architecture.md)** - Deep dive into technical details
- **[Production Examples Analysis](./ANALYSIS.md)** - Detailed comparison of base vs enhanced

