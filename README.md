# Serverless Three-Tier Web Application on AWS

**Oladayo Fasokun** — Data & Cloud Engineer
Built March–July 2026 · Self-directed hands-on builds

---

## What this is

A complete three-tier web application on AWS, built serverless end to end, plus a separate analytics layer over data in S3.

Each tier was built and documented as its own stage so that the responsibilities stay separated — which is the entire point of a tiered architecture. The write-ups below cover what was built, the design decisions behind it, and the failures encountered along the way.

---

## Architecture

```
                        ┌──────────────────────────┐
   End user  ──────────▶│  PRESENTATION TIER       │
                        │  CloudFront distribution │
                        │  Origin: S3 (private)    │
                        │  Access via OAC          │
                        └───────────┬──────────────┘
                                    │  HTTPS / GET
                                    ▼
                        ┌──────────────────────────┐
                        │  LOGIC TIER              │
                        │  API Gateway (REST)      │
                        │  Lambda proxy integration│
                        │  Stage: prod             │
                        └───────────┬──────────────┘
                                    │  invoke
                                    ▼
                        ┌──────────────────────────┐
                        │  DATA TIER               │
                        │  Lambda function         │
                        │  DynamoDB (partition key:│
                        │  userId)                 │
                        │  Custom inline IAM policy│
                        └──────────────────────────┘


   Separate analytics path:

   S3 bucket (dataset + manifest.json)  ───▶  Amazon QuickSight  ───▶  Dashboards
```

---

## The tiers

| Stage | Tier | Services | Write-up |
| --- | --- | --- | --- |
| 1 | Presentation | S3, CloudFront, OAC | [Website delivery with CloudFront](01-presentation-tier-cloudfront.md) |
| 2 | Logic | API Gateway, Lambda | [REST API with Lambda + API Gateway](02-logic-tier-api-gateway.md) |
| 3 | Data | Lambda, DynamoDB, IAM | [Data retrieval with Lambda + DynamoDB](03-data-tier-lambda-dynamodb.md) |
| — | Analytics | S3, QuickSight | [Data visualization with QuickSight](04-analytics-quicksight.md) |

---

## Why serverless throughout

No EC2 instances, no capacity planning, no patching. The trade-off accepted here is cold-start latency and vendor coupling, in exchange for zero idle cost and no server management — appropriate for an application with intermittent traffic.

The security posture is the part worth noting: **the S3 bucket serving the front end is never public.** CloudFront reaches it through Origin Access Control, which means the only path to those objects is through the distribution. This was a deliberate choice made after directly comparing it against S3 static website hosting, which requires public bucket access. That comparison is documented in the presentation tier write-up.

---

## What went wrong, and what fixed it

Every stage broke at least once. These are the failures that taught the most:

| Stage | Failure | Root cause | Fix |
| --- | --- | --- | --- |
| Presentation | `Access Denied` on the distributed site | Distribution origin access set to public; bucket was not public | Switched to Origin Access Control, then applied the generated bucket policy |
| Presentation | `403 Forbidden` on S3 static hosting | Unblocking public access alone is insufficient | Bucket policy also had to grant public read |
| Logic | API returned an error at the prod invoke URL | Lambda had no DynamoDB permission, and the target table did not exist yet | Resolved in the data tier stage: created the table, wrote a scoped inline policy |
| Data | Permissions unclear from managed policies | Needed narrower access than the managed policies offered | Wrote a custom inline IAM policy — first time doing so |

---

## Production gaps

Honest assessment of what this architecture would need before carrying real traffic:

- **No authentication.** The API is open. Cognito or a Lambda authorizer would be the next addition.
- **No throttling or usage plans** on API Gateway — an open endpoint invoking Lambda is a cost risk.
- **No infrastructure as code.** Everything was built through the console. Terraform or CDK would make it reproducible and reviewable.
- **No CI/CD.** Lambda code was edited in the console rather than deployed from a repository.
- **No custom domain or TLS certificate** on the distribution.
- **DynamoDB access is a direct key lookup only** — no query patterns designed for access at scale, no secondary indexes.
- **Observability is default CloudWatch logs**, with no structured logging, metrics, or alarms.

---

## Note on provenance

These were self-directed hands-on builds completed by following structured project guides, then extended beyond the guided steps where noted in each write-up — the S3-versus-CloudFront hosting comparison, the load-time measurement, and the API documentation work were all project extensions.
