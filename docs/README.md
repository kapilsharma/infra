# Infra decisions

Decision record for the shared infrastructure that hosts the product portfolio's **test/staging**
environment while the products are pre-revenue, with a per-product path to scale later.

**Active scope:** GBPN, GMTC, LIQ. **oppe is out of scope** (release deferred).

## Documents

- **[0001-hosting-strategy.md](0001-hosting-strategy.md)** — the decision. One shared arm64 VM
  running Docker Compose (Caddy + pgbouncer + Postgres 17 + MinIO + per-product app/worker
  containers), one database per product, per-product scale-out path. Refines
  [../gemini.md](../gemini.md) Option C.
- **[product-profiles.md](product-profiles.md)** — the evidence: per-product tech stack, services,
  dependencies, resource footprint and existing IaC that the decision is based on.
- **[corrections-and-next-steps.md](corrections-and-next-steps.md)** — corrections to the gemini.md
  premise, per-product blockers to fix, portfolio standardization, open decisions, and build order.

## TL;DR

- Per-product serverless now would cost **$150+/mo** and still couldn't host GBPN's always-on
  worker. One shared AWS `t4g.medium` hosts all three for **~$18–28/mo**.
- Single Postgres 17, **database + roles per product** (preserves each product's RLS isolation);
  **pgbouncer** replaces RDS Proxy for free; public-IP VM avoids the NAT-gateway tax.
- Each product keeps its own scale-out stack: GBPN → Fargate/App Runner (persistent worker);
  GMTC, LIQ → Lambda (GMTC's Terraform is already built). No app code changes to leave the box.

## Decisions (confirmed 2026-09-29)

- Cloud/host: ✅ **AWS `t4g.medium`** (arm64, ~$18–28/mo).
- Scope: ✅ **one shared test/staging box** (GBPN, GMTC, LIQ); prod deferred.
- Object storage: ✅ **real per-product AWS S3 buckets**.
- Still open: **parent domain / DNS provider** for per-product subdomains.
