# Package Versions & Installation Guide

**IMPORTANT**: This document clarifies which packages can be installed from npm vs which must be used locally.

---

## Repository Overview

### Original Repository
- **URL**: [https://github.com/SGFGOV/medusa-strapi-repo](https://github.com/SGFGOV/medusa-strapi-repo)
- **Commits**: 719
- **Status**: Base version, published to npm
- **Published Packages**: Yes (npm)

### Production Fork
- **URL**: [https://github.com/dannycdannydo/medusa-strapi-repo](https://github.com/dannycdannydo/medusa-strapi-repo)
- **Commits**: 764 (+45 commits after original)
- **Status**: Enhanced fork with additional features
- **Published Packages**: No (use local installation)

---

## Package Installation Guide

### Package 1: `medusa-plugin-strapi-ts` (Medusa Backend Plugin)

**Can use npm published package** ✅

| Aspect | Original Repo | Fork Repo | Recommendation |
|--------|---------------|-----------|----------------|
| **Version** | 5.0.21 | 5.0.21 | Same version |
| **Core Logic** | ✅ Identical | ✅ Identical | Use npm package |
| **Published to npm** | ✅ Yes | ❌ No | Use original |

**Installation**:

```bash
# In your Medusa backend directory
yarn add medusa-plugin-strapi-ts

# Or specify version
yarn add medusa-plugin-strapi-ts@5.0.21
```

**Configuration** (in `medusa-config.js`):

```javascript
const plugins = [
  {
    resolve: 'medusa-plugin-strapi-ts',
    options: {
      strapi_protocol: process.env.STRAPI_PROTOCOL || 'http',
      strapi_host: process.env.STRAPI_SERVER_HOSTNAME || 'localhost',
      strapi_port: process.env.STRAPI_PORT || 1337,
      strapi_secret: process.env.STRAPI_SECRET || '',
      strapi_public_key: process.env.STRAPI_PUBLIC_KEY || '',
      auto_start: true,
    },
  },
];
```

**Source**: Both repositories have identical plugin code, so the npm package works for both setups.

---

### Package 2: `medusa-strapi` (Strapi Application)

**Must use LOCAL version** ⚠️

| Aspect | Original Repo | Fork Repo | Difference |
|--------|---------------|-----------|------------|
| **Version** | 1.0.0 (private) | 1.0.0 (private) | Same |
| **Core Content Types** | 27 entities | 27 entities | Same |
| **Custom Content Types** | 0 | 14 additional | ⚠️ DIFFERENT |
| **Scripts** | `yarn develop` | `yarn dev` | Minor change |
| **Env Configs** | JS only | JS + TS (dev/prod) | ⚠️ DIFFERENT |
| **Dependencies** | Base | + Cloudinary, Mailchimp, SendGrid | ⚠️ DIFFERENT |
| **Published to npm** | ❌ No | ❌ No | Must clone repo |

**Why you CANNOT use npm package**:
- ❌ Package is marked as `"private": true` (not published to npm)
- ❌ This is the full Strapi application, not a library
- ❌ Fork has significant modifications and additions

**Installation (Original Version)**:

```bash
# Clone the original repository
git clone https://github.com/SGFGOV/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi

# Install dependencies
yarn install

# Configure environment
cp .env.test .env
# Edit .env with your settings

# Build and start
yarn build
yarn develop
```

**Installation (Fork Version - Enhanced)**:

```bash
# Clone the fork repository
git clone https://github.com/dannycdannydo/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi

# Install dependencies
yarn install

# Configure environment
cp .env.test .env
# Edit .env with your settings

# Build and start
yarn build
yarn dev  # Note: different script name
```

**Key Differences in Fork**:

```diff
+ 14 additional custom content types:
  - homepage-banner, homepage-feature
  - homepage-three-offer, homepage-triple-point
  - policy, team-member
  - about-us-page, contact-us-page
  - website-maintenance-mode
  - general-image, bundle-main-image, all-products-image
  - groomin-guide, contact-us

+ Additional dependencies:
  "@strapi/provider-upload-cloudinary": "^4.25.9"
  "medusa-plugin-mailchimp": "^1.1.51"
  "medusa-plugin-sendgrid": "^1.3.13"
  "pg": "^8.12.0" (updated from 8.6.0)

+ TypeScript environment configs:
  config/env/development/database.ts
  config/env/production/database.ts
```

---

### Package 3: `strapi-plugin-medusajs` (Strapi Plugin)

**Can use npm published package (with caution)** ⚠️

| Aspect | Original Repo | Fork Repo | Recommendation |
|--------|---------------|-----------|----------------|
| **Version** | 5.0.0 | 5.0.0 | Same version |
| **Core Logic** | ✅ Identical | ✅ Identical | Can use npm |
| **Published to npm** | ✅ Yes | ❌ No | Use original |

**Installation**:

```bash
# In your Strapi application (packages/medusa-strapi)
yarn add strapi-plugin-medusajs
```

**However**, if you're using the fork's `medusa-strapi` application, the plugin is **already included** in the monorepo, so no separate installation needed.

**Configuration** (in Strapi `config/plugins.js`):

```javascript
module.exports = ({ env }) => ({
  'medusajs': {
    enabled: true,
    resolve: './src/plugins/strapi-plugin-medusajs'
  },
});
```

---

### Package 4: `strapi-plugin-sso-medusa` (SSO Plugin)

**Can use npm published package** ✅

| Aspect | Original Repo | Fork Repo | Recommendation |
|--------|---------------|-----------|----------------|
| **Version** | 1.0.0 | 1.0.0 | Same version |
| **Core Logic** | ✅ Identical | ✅ Identical | Use npm package |
| **Published to npm** | ✅ Yes | ❌ No | Use original |

**Installation**:

```bash
yarn add strapi-plugin-sso-medusa
```

---

### Package 5: `strapi-plugin-multi-country-select` (Country Selector)

**Can use npm published package** ✅

**Installation**:

```bash
yarn add @sgftech/strapi-plugin-multi-country-select
```

---

## Decision Matrix: Which Repository to Use?

### Use Original Repository (`SGFGOV/medusa-strapi-repo`) If:

✅ You want to use npm packages  
✅ You're learning the integration  
✅ You don't need custom content types  
✅ You want the simplest setup  
✅ You prefer official/published packages  

**Installation Approach**:
```bash
# Clone for Strapi app
git clone https://github.com/SGFGOV/medusa-strapi-repo.git

# Install plugins via npm
yarn add medusa-plugin-strapi-ts
yarn add strapi-plugin-medusajs
yarn add strapi-plugin-sso-medusa
```

### Use Fork Repository (`dannycdannydo/medusa-strapi-repo`) If:

✅ You need the additional 14 custom content types  
✅ You want Cloudinary integration  
✅ You want Mailchimp/SendGrid plugins  
✅ You need TypeScript environment configs  
✅ You're building a production site  
✅ You want homepage/marketing management features  

**Installation Approach**:
```bash
# Clone the fork
git clone https://github.com/dannycdannydo/medusa-strapi-repo.git

# Use the monorepo structure (everything is included)
cd medusa-strapi-repo/packages/medusa-strapi
yarn install
yarn build
yarn dev

# The Medusa plugin can still be from npm
cd /your-medusa-backend
yarn add medusa-plugin-strapi-ts
```

---

## Hybrid Approach (Recommended for Production)

You can **mix and match** based on your needs:

### For Medusa Backend:
```bash
# Use npm package (works with both setups)
yarn add medusa-plugin-strapi-ts@5.0.21
```

### For Strapi Application:

**Option A: Original (Simple)**
```bash
git clone https://github.com/SGFGOV/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi
```

**Option B: Fork (Enhanced)**
```bash
git clone https://github.com/dannycdannydo/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi
```

**Option C: Cherry-pick (Advanced)**
```bash
# Start with original
git clone https://github.com/SGFGOV/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi

# Manually copy custom content types from fork
# Copy specific API directories you need from fork's src/api/
```

---

## Publishing Fork Packages to Your Own Namespace

If you want to publish the fork's enhanced version under your own npm namespace:

### Step 1: Update package.json

```json
{
  "name": "@your-namespace/medusa-strapi",
  "version": "1.0.0-fork.1",
  "description": "Enhanced Medusa-Strapi with custom content types"
}
```

### Step 2: Publish to npm

```bash
# Login to npm
npm login

# Publish as scoped package
npm publish --access public
```

### Step 3: Update documentation

Document your published packages:

```bash
# Install your published version
yarn add @your-namespace/medusa-strapi
```

**Note**: The `medusa-strapi` package is an **application**, not a library, so publishing it to npm doesn't make much sense. Instead, provide it as a template repository or starter.

---

## Summary Table

| Package | Original (npm) | Fork (local) | Recommendation |
|---------|----------------|--------------|----------------|
| **medusa-plugin-strapi-ts** | ✅ v5.0.21 | ✅ Same | **Use npm** |
| **medusa-strapi** | ⚠️ Not published | ⚠️ Not published | **Clone repo** (choose version) |
| **strapi-plugin-medusajs** | ✅ v5.0.0 | ✅ Same | **Use npm** or included in monorepo |
| **strapi-plugin-sso-medusa** | ✅ v1.0.0 | ✅ Same | **Use npm** or included in monorepo |
| **strapi-plugin-multi-country-select** | ✅ Published | ✅ Same | **Use npm** or included in monorepo |

---

## Documentation References

- [Original Repository](https://github.com/SGFGOV/medusa-strapi-repo)
- [Production Fork](https://github.com/dannycdannydo/medusa-strapi-repo)
- [npm: medusa-plugin-strapi-ts](https://www.npmjs.com/package/medusa-plugin-strapi-ts)
- [npm: strapi-plugin-medusajs](https://www.npmjs.com/package/strapi-plugin-medusajs)
- [Implementation Guide](./04-implementation.md)
- [Analysis Document](./ANALYSIS.md)

---

## Quick Start Clarification

### Original Setup (Using npm packages where possible)

```bash
# 1. Clone Strapi application
git clone https://github.com/SGFGOV/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi
yarn install
yarn build

# 2. Install Medusa plugin via npm
cd /your-medusa-backend
yarn add medusa-plugin-strapi-ts
```

### Fork Setup (Using enhanced local version)

```bash
# 1. Clone fork for Strapi application
git clone https://github.com/dannycdannydo/medusa-strapi-repo.git
cd medusa-strapi-repo/packages/medusa-strapi
yarn install
yarn build

# 2. Install Medusa plugin via npm (still works)
cd /your-medusa-backend
yarn add medusa-plugin-strapi-ts
```

**Key Difference**: Only the **Strapi application** changes between versions. The **Medusa plugin** is the same and can be installed from npm in both cases.

---

**Last Updated**: October 27, 2025  
**Validated Against**:
- Original repo: [SGFGOV/medusa-strapi-repo](https://github.com/SGFGOV/medusa-strapi-repo) (719 commits)
- Fork repo: [dannycdannydo/medusa-strapi-repo](https://github.com/dannycdannydo/medusa-strapi-repo) (764 commits, +45)

