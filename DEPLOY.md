# Qintara Corp — deploy bundle for qintaracorp.com

Static site. No build step, no server, no dependencies. Everything is self-contained in `index.html` (fonts, logo, and icons are inlined). Just upload these files to any static host.

## Files
| File | Purpose |
|---|---|
| `index.html` | The full site. Renders completely without JavaScript (content is baked in); JS only adds the animated hero, ROI calculator, filters, and hover effects. |
| `favicon.svg` / `favicon.png` / `apple-touch-icon.png` | Browser-tab and mobile icons (gold Q mark). |
| `og-image.png` | 1200×630 social share card (shows when the link is posted to LinkedIn, X, iMessage, etc.). |
| `robots.txt` / `sitemap.xml` | SEO basics. |

Every URL inside the files uses `https://qintaracorp.com/...`. If you host on a different domain, find-and-replace that string.

---

## Option A — AWS S3 + CloudFront (matches your qintarahealth.com setup)

Mirrors how qintarahealth.com is hosted (S3 bucket + CloudFront + ACM cert). Use the same AWS account and the `axon-admin` profile. Replace names as needed.

**1. Create the bucket and upload**
```bash
aws s3 mb s3://qintaracorp-marketing-603366204939 --profile axon-admin --region us-east-1
aws s3 sync ./qintaracorp-deploy s3://qintaracorp-marketing-603366204939 --profile axon-admin \
  --exclude "DEPLOY.md" --cache-control "public,max-age=300"
```
S3 sets Content-Type automatically from file extensions. If the SVG favicon serves as the wrong type, set it explicitly:
```bash
aws s3 cp ./qintaracorp-deploy/favicon.svg s3://qintaracorp-marketing-603366204939/favicon.svg \
  --profile axon-admin --content-type "image/svg+xml"
```

**2. TLS certificate (must be in us-east-1 for CloudFront)**
Request an ACM cert in `us-east-1` for `qintaracorp.com` and `www.qintaracorp.com`, validate via DNS (add the CNAME records ACM gives you at your registrar).

**3. CloudFront distribution**
- Origin: the S3 bucket, locked down with **Origin Access Control (OAC)** (private bucket, CloudFront-only).
- Default root object: `index.html`
- Alternate domain names (CNAMEs): `qintaracorp.com`, `www.qintaracorp.com`
- Attach the ACM cert from step 2.
- Redirect HTTP → HTTPS.
- ⚠️ **Do NOT add a custom error response mapping 403/404 → /index.html.** This is a single-page marketing site, not a SPA router, and (per our infra notes) distribution-wide error rewrites can break other things later. Leave errors as default.

**4. DNS**
Point `qintaracorp.com` (A/AAAA alias) and `www` at the CloudFront distribution.

**5. On future updates**
Re-sync, then invalidate:
```bash
aws s3 sync ./qintaracorp-deploy s3://qintaracorp-marketing-603366204939 --profile axon-admin --exclude "DEPLOY.md"
aws cloudfront create-invalidation --distribution-id <DIST_ID> --paths "/*" --profile axon-admin
```

---

## Option B — fastest possible (no AWS)
Drag the `qintaracorp-deploy` folder into **Cloudflare Pages**, **Netlify**, or **Vercel**, then point `qintaracorp.com` at it in their dashboard. Live in a couple of minutes. You can always move to S3/CloudFront later.

---

## Verify after deploy
- Visit `https://qintaracorp.com` — full page loads (hero, catalog, ROI calc, industries, all sections).
- View source or disable JavaScript — content still shows (baked in).
- Paste the URL into LinkedIn/iMessage — the gold share card appears.
- Check the tab shows the gold Q favicon.
