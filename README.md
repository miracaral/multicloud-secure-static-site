# Secure static website on a student budget (Azure + AWS)

I put my portfolio site (a static Next.js export) behind a secure cloud setup, using only student credits. My main site, miracaral.com, stays on Vercel and was not touched. This repo documents what I built, what blocked me, and how I proved each control works.

## What I built

**Azure (stays up, costs almost nothing)**
- Storage account with static website hosting
- HTTPS only, minimum TLS 1.2, public blob access disabled, 7-day soft delete
- GitHub Actions deploy on every push to `main`, logging in with OIDC: no password or key is stored in GitHub
- The deploy identity can write to this one storage account and nothing else

**AWS (built, tested, then deleted)**
- Private S3 bucket (Block Public Access on, versioning on)
- CloudFront in front of it, with Origin Access Control: only this distribution may read the bucket
- AWS WAF on CloudFront: AWS Common Rule Set and a rate limit of 100 requests per 5 minutes per IP

## Why two clouds

The original plan was Azure only: Storage, Front Door and WAF. A student account turned out to block several steps:

| Blocker | Error | What I did instead |
| --- | --- | --- |
| My university's tenant blocks Azure CLI login | `AADSTS50105` | Used Azure Cloud Shell in the portal |
| Students may not create app registrations | `allowedToCreateApps: false` | User-assigned managed identity with a federated credential for GitHub OIDC |
| Only five regions allowed; Static Web Apps is in none of them | `RequestDisallowedByAzure` | Storage static website in France Central |
| Front Door is forbidden on student accounts | `Free Trial and Student account is forbidden for Azure Frontdoor resources` | Built the edge layer (CDN + WAF) on AWS |

Two smaller lessons: a service must be registered on a new subscription before first use (a missing `Microsoft.Storage` registration showed up as a misleading `SubscriptionNotFound`), and GitHub's OIDC token now uses an immutable subject format (`repo:owner@id/repo@id:ref:...`), so the first federated credential I wrote did not match (`AADSTS700213`).

## Proof

Tests against the CloudFront distribution:

| Test | Result | Meaning |
| --- | --- | --- |
| Normal request | `200` | Site served through CloudFront |
| Plain HTTP | `301 -> https://` | HTTPS enforced |
| `?q=<script>alert(1)</script>` | `403` | WAF blocked the XSS attempt |
| Request without User-Agent | `403` | WAF blocked a bot-like request |
| Direct S3 URL | `403` | Bucket is private; only CloudFront can read it |

The WAF dashboard showed 20 requests, 18 allowed and 2 blocked, matching the tests. Screenshots are in [`images/`](images).

## Known limitations

- No private link between edge and Azure Storage: that needs Front Door Premium, which students cannot use. The Azure storage web endpoint is public.
- The AWS part was torn down after testing, so there is no custom domain in front of it.

## Cost

Azure storage: a fraction of a cent per month, from student credit. AWS: a few cents for about one hour, from credit. Nothing was paid out of pocket.
