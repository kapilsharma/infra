# ADR 0001 — Bootstrap hosting strategy for the product portfolio

- **Status:** Accepted (2026-09-29). Confirmed: **AWS `t4g.medium`** · **one shared test box** ·
  **real AWS S3 buckets**. One item still open: parent domain / DNS provider.
- **Date:** 2026-09-29
- **Supersedes / refines:** [../gemini.md](../gemini.md) (keeps its Option C direction; corrects
  its premise, sizing, and the "two boxes now" scope)
- **Context source:** [product-profiles.md](product-profiles.md)

## Context

Three pre-revenue products in active scope — **GBPN, GMTC, LIQ** — self-funded, no income yet, all
needing a shared **test/staging** environment now so they can be exercised end-to-end. A fourth
product, **oppe, is out of scope** (release deferred; not sized or provisioned for here) — it can
be added to the same box later with no re-architecting. This `infra` repo is its own git repo and
owns the shared infrastructure.

The goal is **near-zero standing cost now**, with a **clean, per-product path to scale** once a
product earns revenue. gemini.md already recommended Option C (one shared VM + Dockerized
Postgres). The four-product analysis confirms that direction and sharpens it.

## Why not per-product serverless now

Each product's *own* prod design points at AWS serverless (Lambda/Aurora/RDS Proxy/CloudFront).
Doing that now, per product, multiplies exactly the standing costs the bootstrap phase must avoid:

- **RDS Proxy** (GMTC, LIQ) — standing hourly cost, ~⅓–½ of GMTC's ~$42–57/mo test estimate.
- **Aurora Serverless v2** (LIQ) — 0.5 ACU floor never scales to zero.
- **NAT gateway** (~$35/mo) the moment any VPC-Lambda needs outbound (Razorpay, SES, Safe
  Browsing, MaxMind) — i.e. everything except GMTC.
- Three-plus separate stacks to operate before the first dollar.

Rough portfolio serverless test cost ≈ 3 × $42–57 **+** NAT/proxy overheads = well over
**$150/mo**. And it *still* would not host **GBPN's always-on pg-boss worker** cleanly.

## Decision (recommended)

**One shared, arm64 Linux VM running Docker Compose** hosts the whole portfolio's **test/staging**
environment:

```text
                         Internet (per-product subdomains, TLS)
                                     │
                          ┌──────────┴──────────┐
                          │   Caddy (reverse     │  auto Let's Encrypt TLS,
                          │   proxy + TLS)       │  host- and /path-based routing
                          └──────────┬──────────┘
      ┌───────────────────┬─────────────┼─────────────────┐
   gbpn-api            gmtc-api       liq-api
   gbpn-worker        (gmtc-sweep)  (liq-worker)
   (+ static SPAs served by Caddy for gbpn / gmtc / liq; Astro for liq)
      └───────────────────┴─────────────┼─────────────────┘
                          ┌──────────┴──────────┐
                          │  pgbouncer (txn mode)│  ← gives GMTC its required
                          └──────────┬──────────┘     proxy semantics for free
                          ┌──────────┴──────────┐
                          │  Postgres 17         │  one instance, one DATABASE
                          │  db_gbpn  db_gmtc    │  per product, per-product
                          │  db_liq              │  owner + NOBYPASSRLS roles
                          └─────────────────────┘     (db_oppe added later)

   Object storage: real AWS S3 buckets (gbpn / gmtc / liq) — presigned PUT/GET via instance role
```

Key component choices:

1. **Compute:** one VM — **AWS Lightsail or EC2 `t4g.medium` (4 GB / 2 vCPU, arm64)** — *not* the
   `t4g.small` (2 GB) gemini.md assumed. GBPN's API alone wants ~2 GB under an Argon2id login burst;
   2 GB cannot also hold Postgres + two other apps + workers. 4 GB is the safe floor for the three
   in-scope products on one test box.
2. **Database:** single **Postgres 17** container, **one database per product**, each with a
   dedicated owner (migrations) role and a `NOBYPASSRLS` app role — preserves every product's RLS
   isolation. `pg_dump`-per-database is the future lift-to-RDS unit.
3. **Pooling:** **pgbouncer in transaction mode** in front of Postgres. Free replacement for RDS
   Proxy / Aurora Proxy; satisfies GMTC's hard requirement (prepared statements off, txn-local
   tenant context) and GBPN's per-request `SET LOCAL`.
4. **Ingress / TLS:** **Caddy** (auto Let's Encrypt). One subdomain per product per env
   (`gbpn.test.…`, `gmtc.test.…`, `liq.test.…`, plus oppe's apex + `api.`/`app.`). Routes static
   SPA vs `/api`|`/v1` to the right container; keeps API ports internal; handles Razorpay
   webhooks. GBPN already uses Caddy.
5. **Object storage (test):** **real per-product AWS S3 buckets** (`gbpn`, `gmtc`, `liq`), created
   by Terraform with CORS for the app origins; the box's app containers use presigned PUT/GET via
   an instance role (no static keys). Chosen over on-box MinIO for prod-parity and one less moving
   part. Same buckets (or per-account equivalents) carry over at scale-out.
6. **Workers:** run each as its own container off the product's image. GBPN's pg-boss worker is
   **always-on**; GMTC/LIQ/oppe are **cron/systemd-timer**-driven sweep entrypoints.
7. **Caches/queues/search:** none provisioned. Redis (oppe) and SQS/EventBridge (GMTC/LIQ) stay
   deferred — every product already runs without them today (PG-backed or in-process).

### Scope for *now*

Provision **one shared test/staging box** for **GBPN, GMTC and LIQ** only. **oppe is out of
scope** — do not size or provision for it now; it slots onto the same box later (add `db_oppe` + its
containers) with no re-architecting. **Defer a separate prod box** until a product is ready for real
users — at which point that product graduates to its own stack (below) rather than sharing.

## Infrastructure-as-Code plan

- This repo provisions the **shared box and its surroundings** only: the VM + EIP + security
  group, DNS records, IAM/OIDC for CI, and (if used) S3 buckets. App containers are deployed *onto*
  the box via Docker Compose driven by a thin GitHub Actions → SSH step (or pull-based), decoupled
  from Terraform — mirroring how GMTC already separates code deploys from infra.
- Layout follows gemini.md's intent, using **directories per environment** (clearer than
  workspaces, and matches GMTC's `envs/` convention):

  ```text
  infra/
  ├── docs/                 # these decision docs
  ├── modules/
  │   ├── compute/          # VM, EIP, security group, cloud-init (Docker + Compose)
  │   ├── network/          # minimal: default VPC + SG (no NAT)
  │   └── dns/              # per-product records
  ├── compose/              # the shared docker-compose.yml + Caddyfile + pgbouncer + per-product .env
  └── envs/
      ├── test/             # instantiates modules for the shared test box  ← build this first
      └── prod/             # added later, per-product or shared
  ```
- **Do NOT discard the products' own Terraform.** GMTC's mature serverless tree (Org + OIDC +
  Lambda/RDS/CloudFront modules) and LIQ's skeleton become each product's **scale-out stack**.
  GMTC's AWS Organization + GitHub-OIDC pattern is reusable as the portfolio's account/CI backbone.

## Path to scale (per product, when it earns revenue)

Lift **one** product off the shared box into its own stack — the model differs by product and that
is fine (do not force one model on all):

- **Static frontends** (GBPN/GMTC/LIQ, + LIQ Astro) → S3 + CloudFront.
- **APIs:**
  - GBPN → containers on **Fargate / App Runner** (needs the persistent pg-boss worker; its
    `SET LOCAL` tenancy also prefers a long-lived pooled connection).
  - GMTC, LIQ → **Lambda** per their existing / designed Terraform (GMTC's is ready).
- **Database** → dedicated **RDS Postgres 17** via `pg_dump`/`pg_restore` of that one database;
  add RDS Proxy / pgbouncer as the design requires.
- **Object storage** → dedicated per-product S3 buckets (already the app contract).
- **Async** → the product's own SQS/EventBridge (GMTC/LIQ) or pg-boss-on-Fargate (GBPN).

No application code changes are required to leave the shared box (Hono/Fastify are
runtime-agnostic; each app reads a `DATABASE_URL` and storage/queue behind interfaces).

## Consequences

**Positive**
- All four products testable for roughly **one small VM's cost** (~$18–28/mo on AWS for the whole
  portfolio; ~$6–9/mo on a budget VPS — see knob) vs $150+/mo for per-product serverless.
- Sidesteps NAT gateway, RDS Proxy and Aurora-floor costs entirely.
- Natively supports GBPN's always-on worker, which serverless-only cannot.
- Matches local dev (docker-compose everywhere) → high prod-parity, low surprise.
- Each product keeps an independent, clean scale-out path; no lock-in to the shared box.

**Negative / risks**
- **Single box = single failure domain** for test/staging (acceptable for non-prod; snapshot +
  restore is the recovery plan). Not for real prod without at least backups + a rebuild script.
- **Noisy-neighbour**: an Argon2id login storm or a heavy export in one product can starve others.
  4 GB mitigates; set per-container memory limits; upsize before prod users arrive.
- **You operate a Linux box** (patching, backups, TLS renewal is auto via Caddy). Low, but non-zero.
- **Self-managed Postgres** has no managed backups — must script `pg_dump` + off-box copy (e.g. to
  S3) from day one.
- **Version skew** (GBPN/LIQ on PG 16, GMTC on PG 17; all on Node 22) is contained by per-container
  images + a PG 17 server, but first migrations must be verified against PG 17.

## Decisions (confirmed 2026-09-29)

1. **Cloud/host** — **AWS** (Lightsail/EC2 `t4g.medium`, ~$18–28/mo). Keeps the S3/Lambda scale-out
   path frictionless and matches existing AWS Terraform experience.
2. **Scope now** — **one shared test/staging box** for GBPN, GMTC, LIQ. A separate prod box is
   deferred until a product has real users.
3. **Object storage** — **real per-product AWS S3 buckets** now (not on-box MinIO), for prod-parity
   and fewer moving parts.

Still open:

- **Parent domain / DNS provider** for the per-product subdomains (Route53 vs Cloudflare, and which
  domain). Needed before TLS + subdomain routing can be wired.

These are recorded as open in [corrections-and-next-steps.md](corrections-and-next-steps.md);
the recommended defaults above are AWS · one test box · MinIO.
