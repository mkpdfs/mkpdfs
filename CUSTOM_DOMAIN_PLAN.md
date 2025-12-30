# Custom Domain Setup Plan for Templify API

## Domain: mkpdfs.com
**AWS Profile:** `rocketeast`
**AWS Account:** `197837191835`

## Target URLs
| Environment | Domain |
|-------------|--------|
| Production | `https://apis.mkpdfs.com` |
| Development | `https://dev.apis.mkpdfs.com` |

---

## Current State (Completed)

### 1. Domain Registration ✅
- Domain `mkpdfs.com` registered in AWS Route53 Domains
- Auto-renew enabled, expires 2026-12-27

### 2. Route53 Hosted Zone ✅
- Hosted Zone ID: `Z0217803KO361QOLBIHN`
- Nameservers already configured:
  - ns-270.awsdns-33.com
  - ns-1505.awsdns-60.org
  - ns-639.awsdns-15.net
  - ns-1861.awsdns-40.co.uk

### 3. ACM Certificates ✅ (Pending Validation)
- **Prod:** `arn:aws:acm:us-east-1:197837191835:certificate/cbc979b6-0d23-4997-bb6e-0ee72ac3557a`
- **Dev:** `arn:aws:acm:us-east-1:197837191835:certificate/1a16de41-d72e-4c71-8cff-f678dc9ea6b3`
- DNS validation records added to Route53

### 4. Serverless Plugin ✅
- `serverless-domain-manager` already installed

---

## Remaining Steps

### Step 1: Update serverless.ts
Edit the custom domain configuration:

```typescript
// In serverless.ts, update the customDomain section:
customDomain: {
  domainName: '${self:custom.domainNames.${self:provider.stage}}',
  basePath: '',
  certificateName: '${self:custom.domainNames.${self:provider.stage}}',
  createRoute53Record: true,
  createRoute53IPv6Record: true,
  endpointType: 'edge',
  securityPolicy: 'tls_1_2',
  hostedZoneId: 'Z0217803KO361QOLBIHN'  // <-- UPDATE THIS
},
domainNames: {
  dev: 'dev.apis.mkpdfs.com',           // <-- UPDATE THIS
  stage: 'stage.apis.mkpdfs.com',       // <-- UPDATE THIS
  prod: 'apis.mkpdfs.com'               // <-- UPDATE THIS
}
```

### Step 2: Verify Certificates are Issued
```bash
# Check prod certificate status
AWS_PROFILE=rocketeast aws acm describe-certificate \
  --certificate-arn "arn:aws:acm:us-east-1:197837191835:certificate/cbc979b6-0d23-4997-bb6e-0ee72ac3557a" \
  --region us-east-1 \
  --query 'Certificate.Status' --output text

# Check dev certificate status
AWS_PROFILE=rocketeast aws acm describe-certificate \
  --certificate-arn "arn:aws:acm:us-east-1:197837191835:certificate/1a16de41-d72e-4c71-8cff-f678dc9ea6b3" \
  --region us-east-1 \
  --query 'Certificate.Status' --output text
```

Both should return `ISSUED` (may take a few minutes).

### Step 3: Create Custom Domains in API Gateway
```bash
# Create prod domain
AWS_PROFILE=rocketeast npx serverless create_domain --stage prod

# Create dev domain (optional)
AWS_PROFILE=rocketeast npx serverless create_domain --stage dev
```

### Step 4: Deploy
```bash
# Deploy prod
AWS_PROFILE=rocketeast npm run deploy:prod

# Deploy dev (optional)
AWS_PROFILE=rocketeast npm run deploy:dev
```

### Step 5: Verify
```bash
# Check DNS resolution
dig apis.mkpdfs.com

# Test endpoint (should return 401 Unauthorized - that's expected without auth)
curl -I https://apis.mkpdfs.com/user/profile
```

---

## Rollback (if needed)
```bash
# Delete custom domain
AWS_PROFILE=rocketeast npx serverless delete_domain --stage prod

# Revert serverless.ts changes
git checkout serverless.ts
```

---

## Notes
- The old domain `prod.apis.templifying.com` will stop working after this change
- DNS propagation typically takes 1-5 minutes for Route53
- ACM certificate validation usually completes within 5-30 minutes
