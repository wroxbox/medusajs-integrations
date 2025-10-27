# Frequently Asked Questions (FAQ)

Quick answers to common questions about the Medusa-Strapi integration.

## General Questions

### Q: What is this integration?

**A:** A connection between Medusa v1 (e-commerce backend) and Strapi v4 (headless CMS) that automatically syncs product structure from Medusa to Strapi, allowing content teams to enrich products with rich descriptions, images, and custom content.

---

### Q: Is this an official Medusa plugin?

**A:** No, this is a community-maintained plugin created by [@SGFGOV](https://github.com/SGFGOV). While it follows Medusa's plugin architecture, it's not officially supported by Medusa core team.

---

### Q: Is this free?

**A:** Yes, both Medusa and Strapi are open source (MIT license). However, you'll need to pay for infrastructure (servers, databases). Typical cost: $100-200/month for production deployment.

---

### Q: Can I use this with Medusa v2?

**A:** No, this plugin was built for Medusa v1 and is not compatible with v2's architecture. For Medusa v2, use the Sanity or Payload integrations instead.

**Migration Path:**
- Stay on Medusa v1 with Strapi (maintenance mode)
- Or migrate to Medusa v2 + Sanity/Payload (significant effort)

---

### Q: How does this compare to Sanity or Payload integrations?

**A:**

| Feature | Strapi | Sanity | Payload |
|---------|--------|--------|---------|
| Medusa Version | v1 | v2 | v2 |
| Open Source | ✅ Yes | ❌ No | ✅ Yes |
| Self-Hosted | ✅ Yes | ❌ No | ✅ Yes |
| Deployment | Separate | External SaaS | Embedded |
| Cost | Infra only | Usage-based | Infra only |
| Maintenance | High | Low | Medium |

**Choose Strapi if:** On v1, need FOSS, want self-hosting.
**Choose Sanity if:** On v2, want ease, prefer managed.
**Choose Payload if:** On v2, want integrated, TypeScript-native.

---

## Technical Questions

### Q: What data syncs between Medusa and Strapi?

**A:**

**Medusa → Strapi (Automatic):**
- Products (title, description, handle, metadata)
- Variants (SKU, title, prices, options)
- Collections (title, handle, products)
- Categories (name, handle, hierarchy)
- Regions (name, currency, providers)

**Strapi → Medusa (Limited):**
- Product title (if manually triggered)
- Product subtitle (if manually triggered)
- All other content stays in Strapi only

---

### Q: Does the storefront read from Medusa or Strapi?

**A:** Both. The storefront needs to:
1. Fetch commerce data (prices, inventory, cart) from Medusa
2. Fetch content data (descriptions, images, custom fields) from Strapi

**Two Options:**
- **Option A:** Make two API calls (one to each system)
- **Option B:** Use Medusa proxy endpoint `/strapi/content/`

---

### Q: What happens if Strapi goes down?

**A:**

**During Sync:**
- Medusa events queue up
- Retry logic attempts to reconnect
- Once Strapi is back, sync resumes
- No data loss (events stored in Redis)

**On Storefront:**
- Commerce functionality works (prices, cart, checkout)
- Content missing (descriptions, images)
- Graceful degradation possible with try/catch

**Best Practice:** Implement fallbacks and cache Strapi content.

---

### Q: Can I customize what fields sync?

**A:** Yes, but requires code changes:

```typescript
// In medusa-plugin-strapi-ts/src/subscribers/strapi.ts
// Modify the subscriber to sync additional fields

// In medusa-plugin-strapi-ts/src/services/update-strapi.ts
// Add your custom field transformations
```

**Note:** You'll need to maintain these customizations across plugin updates.

---

### Q: How do I add custom fields to Strapi?

**A:**

1. **Add Field in Strapi Admin:**
   - Content-Type Builder → Products
   - Add Field (e.g., "Sustainability Score")
   - Save

2. **Update Plugin Transform (if syncing from Medusa):**
   ```typescript
   // In update-strapi.ts
   const productToSend = {
     ...product,
     sustainability_score: product.metadata?.sustainability_score
   };
   ```

3. **Query in Storefront:**
   ```javascript
   const content = await fetch(
     `${STRAPI_URL}/api/products?fields[]=sustainability_score`
   );
   ```

---

### Q: How do I prevent sync loops?

**A:** The plugin uses Redis "ignore flags" with TTL:

```typescript
// Before syncing, check flag
const ignore = await redis.get(`${productId}_ignore_strapi`);
if (ignore) return; // Skip

// After syncing, set flag (expires in 3 seconds)
await redis.set(`${productId}_ignore_strapi`, 1, 'EX', 3);
```

**Requirements:**
- Redis must be running
- Both systems must use Redis flags
- TTL must be longer than sync time

---

### Q: Can I use this without Redis?

**A:** No, Redis is required because:
1. Medusa's event bus depends on Redis
2. Ignore flags prevent sync loops
3. Token caching uses Redis

**Minimum Redis:** v6.0+

---

### Q: How do I bulk sync existing products?

**A:**

**Method 1: Via Config (Recommended)**
```javascript
// medusa-config.js
{
  resolve: "medusa-plugin-strapi-ts",
  options: {
    sync_on_init: true,  // ← Enable
    auto_start: true
  }
}
```

**Method 2: Via API**
```bash
# Call Strapi sync endpoint
curl -X POST http://localhost:1337/strapi-plugin-medusajs/synchronise-medusa-tables \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN"
```

**Note:** Can take 30+ minutes for large catalogs.

---

### Q: Why are products not syncing?

**A:** Check these in order:

1. **Redis running?**
   ```bash
   redis-cli ping  # Should return PONG
   ```

2. **Strapi reachable?**
   ```bash
   curl http://localhost:1337/_health  # Should return 204
   ```

3. **Secrets match?**
   ```bash
   # Medusa .env
   echo $STRAPI_SECRET
   
   # Strapi .env
   echo $JWT_SECRET
   echo $MEDUSA_STRAPI_SECRET
   
   # All three must be identical
   ```

4. **Subscriber loaded?**
   ```bash
   # Check Medusa logs for:
   info: Strapi Subscriber Initialized
   ```

5. **Event bus working?**
   ```bash
   # Try creating product, watch logs
   # Should see event: "product.created"
   ```

---

## Performance Questions

### Q: How fast is the sync?

**A:**

**Single Product:**
- Event emission: ~5ms
- Authentication: ~100ms (cached)
- HTTP request: ~150ms
- Strapi processing: ~80ms
- **Total:** ~335ms

**Bulk Sync:**
- ~50-100 products/minute
- 500 products: ~5-10 minutes
- 2000 products: ~20-40 minutes

**Bottlenecks:**
- Network latency (Medusa ↔ Strapi)
- Strapi API rate limits
- PostgreSQL write speed

---

### Q: Does this slow down my storefront?

**A:**

**Option A: Dual Fetch**
- Medusa API: ~150ms
- Strapi API: ~200ms
- **Total:** ~350ms (vs 150ms Medusa-only)
- **Impact:** +200ms latency

**Option B: Medusa Proxy**
- Medusa → Strapi → Response: ~250ms
- **Impact:** +100ms latency

**Mitigations:**
- Cache Strapi content (Redis/CDN)
- Implement request parallelization
- Use edge caching (Cloudflare, Vercel)
- Lazy load non-critical content

---

### Q: Can this scale to millions of products?

**A:** Yes, but with considerations:

**Challenges:**
- Initial bulk sync will take hours
- Database size grows in both systems
- API rate limits may throttle sync

**Solutions:**
- Use managed PostgreSQL (RDS, Cloud SQL)
- Scale Strapi horizontally (load balancer + multiple instances)
- Implement incremental sync (only changed products)
- Use database connection pooling
- Consider async job queues for bulk operations

**Real-World:** Successfully used with 10,000+ product catalogs.

---

## Cost Questions

### Q: What are the infrastructure costs?

**A:**

**Development:**
- $0 (run locally)

**Production (Self-Hosted):**

| Component | Monthly Cost |
|-----------|--------------|
| Medusa server (2GB RAM) | $40 |
| Medusa PostgreSQL | $15 |
| Medusa Redis | $10 |
| Strapi server (2GB RAM) | $40 |
| Strapi PostgreSQL | $15 |
| **Total** | **$120/month** |

**Production (Managed Services):**

| Component | Monthly Cost |
|-----------|--------------|
| Railway/Heroku (Medusa) | $25-50 |
| Managed PostgreSQL | $25 |
| Managed Redis | $15 |
| Strapi Cloud (1GB) | $15-99 |
| **Total** | **$80-189/month** |

**vs. Alternatives:**
- Medusa only: $65/month
- Medusa + Sanity: $65 + $0-199/month (usage-based)
- Medusa + Payload: $65/month (embedded)

---

### Q: Is there a free tier?

**A:**

**Software:**
- ✅ Both Medusa and Strapi are free (open source)

**Infrastructure:**
- ⚠️ Need servers and databases (costs apply)
- Can use free tiers for testing:
  - Railway: $5 credit/month
  - Heroku: Hobby tier (limited)
  - Render: Free tier (slow)

**Recommended:** Budget $100-200/month for production.

---

## Troubleshooting

### Q: "Unable to connect to Strapi" error

**A:**

**Check 1: Strapi running?**
```bash
curl http://localhost:1337/_health
# Should return: 204 No Content
```

**Check 2: Correct hostname?**
```bash
# In Medusa .env
STRAPI_SERVER_HOSTNAME=localhost  # For local
STRAPI_SERVER_HOSTNAME=strapi.yourdomain.com  # For production
```

**Check 3: Firewall blocking?**
```bash
# Test from Medusa server
telnet localhost 1337
# Should connect
```

---

### Q: "Authentication failed" error

**A:**

**Check 1: Secrets match?**
```bash
# Medusa
grep STRAPI_SECRET .env

# Strapi
grep JWT_SECRET .env
grep MEDUSA_STRAPI_SECRET .env

# Must all be identical
```

**Check 2: Service account exists?**
- Open Strapi Admin → Users
- Look for user with email `STRAPI_MEDUSA_EMAIL`
- If missing, restart Medusa (auto-creates)

**Check 3: Test login manually:**
```bash
curl -X POST http://localhost:1337/api/auth/local \
  -H "Content-Type: application/json" \
  -d '{
    "identifier": "medusa@service.local",
    "password": "your_password"
  }'

# Should return: {"jwt": "...", "user": {...}}
```

---

### Q: Products showing in Medusa but not Strapi

**A:**

**Check 1: Initial sync completed?**
```bash
# Check Strapi database
psql -U strapi_user -d strapi_medusa
SELECT COUNT(*) FROM products;
```

**Check 2: Check Medusa logs**
```bash
# Should see:
info: creating product in strapi
info: POST http://localhost:1337/api/products
info: Successfully created product
```

**Check 3: Manual trigger**
```bash
# Restart Medusa with sync enabled
# In medusa-config.js: sync_on_init: true
```

---

### Q: Sync is very slow (minutes per product)

**A:**

**Cause 1: Network latency**
```bash
# Test ping
ping strapi.yourdomain.com

# If >100ms, consider:
# - Moving servers closer
# - Using same datacenter
# - Checking network issues
```

**Cause 2: Database slow**
```bash
# Check PostgreSQL performance
# Add indexes:
CREATE INDEX idx_products_medusa_id ON products(medusa_id);
CREATE INDEX idx_variants_medusa_id ON product_variants(medusa_id);
```

**Cause 3: Rate limiting**
```bash
# Check for 429 errors in logs
# Increase rate limits in Strapi config
```

---

## Security Questions

### Q: Is this secure?

**A:**

**Security Measures:**
- ✅ JWT authentication between systems
- ✅ Shared secret validation
- ✅ Service account with limited permissions
- ✅ HTTPS recommended for production
- ✅ PostgreSQL authentication
- ✅ Redis password protection (configure)

**Vulnerabilities:**
- ⚠️ Secrets in .env files (use secret managers)
- ⚠️ No built-in rate limiting (add middleware)
- ⚠️ Public Strapi API (configure access control)

**Best Practices:**
1. Use strong, unique secrets
2. Enable HTTPS/SSL in production
3. Restrict Strapi API access
4. Implement rate limiting
5. Regular security updates
6. Monitor access logs

---

### Q: Can I use this in production?

**A:** Yes, but:

**Proven:** Used by real e-commerce stores.

**Considerations:**
- Community plugin (not official)
- Limited ongoing development
- You're responsible for maintenance
- Legacy architecture (Medusa v1)

**Recommendations:**
1. Test thoroughly in staging
2. Have rollback plan
3. Monitor sync reliability
4. Budget for maintenance
5. Plan migration path to v2

---

### Q: What about GDPR compliance?

**A:**

**Data Storage:**
- Both Medusa and Strapi store product data
- Customer data stays in Medusa
- You control all data (self-hosted)

**Compliance Steps:**
1. Document data flows
2. Implement data retention policies
3. Add data deletion workflows
4. Configure backup policies
5. Ensure secure transmission (HTTPS)
6. Add privacy notices

**Note:** Self-hosting gives you full control for compliance.

---

## Migration Questions

### Q: Can I migrate from Medusa v1 to v2 with this?

**A:** Not easily. Medusa v2 has breaking changes:

**Challenges:**
- Event system changed
- Service architecture redesigned
- Plugin structure different
- No direct migration path

**Options:**
1. Stay on v1 (maintenance mode)
2. Port plugin to v2 (significant development)
3. Migrate to Sanity/Payload for v2

**Recommendation:** For new projects, start with Medusa v2 + Sanity/Payload.

---

### Q: How do I migrate away from Strapi?

**A:**

**To Medusa Only:**
1. Export content from Strapi via API
2. Store in Medusa metadata/custom fields
3. Remove Strapi plugin from config
4. Update storefront to read from Medusa only
5. Decommission Strapi server

**To Sanity (v2):**
1. Migrate Medusa v1 → v2
2. Export Strapi content
3. Transform to Sanity schema
4. Import to Sanity
5. Update storefront
6. Decommission Strapi

**To Payload (v2):**
1. Migrate Medusa v1 → v2
2. Install Payload in Next.js
3. Export and import content
4. Update queries
5. Decommission Strapi

---

## Support Questions

### Q: Where can I get help?

**A:**

**Community:**
- [Medusa Discord](https://discord.gg/medusajs) - #plugins channel
- [Strapi Discord](https://discord.strapi.io)
- [GitHub Issues](https://github.com/SGFGOV/medusa-strapi-repo/issues)

**Documentation:**
- [Medusa Docs](https://docs.medusajs.com/v1/)
- [Strapi Docs](https://docs.strapi.io/)
- This documentation

**Plugin Maintainer:**
- GitHub: [@SGFGOV](https://github.com/SGFGOV)
- Discord: @govdiw

**Commercial Support:**
- Consider [sponsoring maintainer](https://github.com/sponsors/SGFGOV)
- Hire Medusa consultants
- Hire Strapi agencies

---

### Q: Can I contribute to the plugin?

**A:** Yes! The plugin is open source.

**Ways to Contribute:**
1. Report bugs (GitHub Issues)
2. Submit pull requests
3. Improve documentation
4. Share use cases
5. Help others in Discord
6. Sponsor development

**Repository:** https://github.com/SGFGOV/medusa-strapi-repo

---

### Q: Is there commercial support available?

**A:**

**Official:** No official commercial support (community plugin).

**Alternatives:**
1. Sponsor maintainer for priority support
2. Hire Medusa consultants
3. Contract Strapi agencies
4. Internal team training

---

## Still Have Questions?

**Didn't find your answer?**

1. **Search this documentation** - Use browser search (Ctrl/Cmd + F)
2. **Check GitHub Issues** - Someone may have asked before
3. **Ask in Discord** - Medusa #plugins channel
4. **Open GitHub Issue** - For bugs or feature requests

**Found an issue with this documentation?**
- Submit a PR with corrections
- Open an issue with details

---

## What's Next?

- **[Implementation Guide](./04-implementation.md)** - Set it up step-by-step
- **[Pros & Cons](./05-pros-cons.md)** - Make the decision
- **[Architecture](./02-architecture.md)** - Technical deep dive
- **[Data Flow](./03-data-flow.md)** - Understand sync scenarios
- **[Summary](./SUMMARY.md)** - Quick reference guide

