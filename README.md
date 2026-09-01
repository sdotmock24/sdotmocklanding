# sdotmock.com — Personal Site Infrastructure

Source and infrastructure for [sdotmock.com](https://sdotmock.com). The site is a
static landing page; what makes this repo worth reading is how it's served and
deployed — a private-origin CDN architecture with credential-free CI/CD, all on AWS.

**Live:** https://sdotmock.com · **Region:** us-west-2 (content) / us-east-1 (certs)

---

## Architecture

```
                         Route 53 / DNS
                              │
             ┌────────────────┴────────────────┐
             ▼                                  ▼
   www.sdotmock.com                       sdotmock.com
   CloudFront (E19ZH97TU1T92I)            CloudFront (EO35ARSZQHAK3)
   │  viewer-request function:            │  viewer-request function:
   │  301 → https://sdotmock.com          │  append index.html to dir requests
   │  (origin never contacted)            │
   │                                      ▼
   │                            ACM cert (us-east-1, TLS 1.2+)
   │                                      │
   └──────────────┐            Origin Access Control (SigV4)
                  ▼                       │
        (dummy S3 origin)                 ▼
                              S3: sdotmock.com  (PRIVATE — no public access)
                                        ▲
                                        │  s3:GetObject allowed ONLY for this
                                        │  distribution, via OAC + SourceArn condition
```

Two distributions, one per hostname. The apex serves content from a fully private
S3 bucket; `www` is a redirect-only distribution that 301s to the apex at the edge.

---

## Design decisions & tradeoffs

| Decision | Alternative considered | Why this way |
|---|---|---|
| **CloudFront + private S3 (OAC)** | S3 static website hosting | Website hosting requires a public bucket, has no TLS on a custom domain, and no edge cache. OAC keeps the bucket fully private — CloudFront is the only reader. |
| **Origin Access Control (OAC)** | Origin Access Identity (OAI) | OAC is the current mechanism; supports SigV4, SSE-KMS, and all regions. OAI is legacy. |
| **`www` redirect via CloudFront Function** | S3 website RedirectAllRequestsTo | A private (OAC) origin has no website-hosting features, so the redirect can't live in S3. A viewer-request function returns the 301 before the origin is ever contacted. |
| **CloudFront Function for `index.html` rewrite** | Rely on default behavior | The REST endpoint (unlike the website endpoint) doesn't auto-serve `index.html` for sub-paths. `default_root_object` covers the root; the function covers directories. |
| **GitHub Actions + OIDC federation** | Access keys stored in GitHub secrets | No long-lived credentials anywhere. Actions assumes an IAM role via short-lived OIDC tokens, scoped by a trust condition to this GitHub account's repos. |
| **Terraform, imported not recreated** | Rebuild from scratch | The distributions were already live. Importing and migrating in place avoided downtime and preserved the existing certs and distribution IDs. |
| **Two-phase, zero-downtime cutover** | Single apply | Public read was kept alive while the new OAC origin propagated, then removed in a second apply. The site never went unreachable during migration. |

---

## Security posture

The bucket is genuinely private — this is verifiable, not asserted:

```bash
# Through CloudFront: works
curl -sI https://sdotmock.com | head -n 1
# HTTP/2 200

# Direct to the S3 origin: denied
curl -sI http://sdotmock.com.s3-website-us-west-2.amazonaws.com | head -n 1
# HTTP/1.1 403 Forbidden
```

- S3 bucket: all four public-access-block settings **on**.
- Bucket policy: `s3:GetObject` allowed only for the CloudFront service principal,
  further scoped with an `AWS:SourceArn` condition to this one distribution.
- CloudFront: HTTP redirects to HTTPS; TLS 1.2 minimum.
- CI/CD: zero stored AWS credentials — OIDC only.

---

## CI/CD

On every push to `master`, GitHub Actions:

1. Assumes `GitHubActionsDeployRole` via OIDC (no secrets).
2. Syncs the site to the S3 bucket (`aws s3 sync --delete`, excluding repo metadata).
3. Invalidates the CloudFront cache.

Workflow: [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml).

### Debugging note: OIDC trust conditions

The first deploy failed with `Not authorized to perform sts:AssumeRoleWithWebIdentity`.
The role's trust policy had three problems worth recording, because each is a common
OIDC pitfall:

- The `sub` claim used **`StringEquals`** with a wildcard — wildcards only match under
  **`StringLike`**; under `StringEquals` the `*` is a literal and never matches.
- The `sub` value contained numeric org/repo IDs (`sdotmock24@…/repo@…`) — the real
  token uses plain `owner/repo` names.
- It was scoped to a different repo and to `main`, while this repo deploys from `master`.

Fixed by scoping the trust condition to `repo:<owner>/*:*` under `StringLike`. In a
team setting this would be tightened to per-repo and per-environment; the wildcard is
a deliberate convenience for a single-owner portfolio account.

---

## Cost

Effectively a rounding error for a low-traffic personal site:

- CloudFront: pay-per-request; pennies at portfolio traffic.
- S3: a few MB of static assets — cents.
- Route 53 hosted zone: ~$0.50/mo if applicable.
- ACM certificates: free.

The larger cost lesson lived in cleanup: an audit of the account found ~135 MB of
orphaned CodePipeline artifacts and unused staging buckets from earlier experiments,
since removed.

---

## Repo layout

```
.
├── index.html
├── assests/            # css, icons, images, js  (note: legacy spelling)
└── .github/workflows/
    └── deploy.yml      # OIDC deploy pipeline
```

Terraform for the infrastructure (OAC, bucket policy, CloudFront, functions) lives
in the infrastructure repo, applied against a remote S3 backend with DynamoDB locking.

---

## What I'd do differently at scale

- **Scope the OIDC trust condition per-repo and per-branch/environment** rather than a
  wildcard — smaller blast radius.
- **Tighten the deploy role** from broad managed permissions to a custom least-privilege
  policy covering only the specific bucket and distribution.
- **Version asset filenames** (content hashing) instead of blanket `/*` cache
  invalidations, which cost money past the monthly free tier and are wasteful.
- **Add a response-headers policy** (HSTS, CSP, `X-Content-Type-Options`) at CloudFront.
- **Fix the `assests/` → `assets/` spelling** (requires updating HTML/CSS references).
```
