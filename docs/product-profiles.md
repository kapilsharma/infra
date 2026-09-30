# Product infrastructure profiles

Reference data gathered from each product's repo (README/CLAUDE/ADRs/docker-compose/IaC) on
2026-09-29. This is the input to the hosting decision in
[0001-hosting-strategy.md](0001-hosting-strategy.md). Cost figures are **unverified estimates**
copied from each project's own docs.

> **Scope note:** The active portfolio is **GBPN, GMTC, LIQ**. **oppe is out of scope** (release
> deferred) — its profile is kept below for reference only and is **not** part of the current
> sizing or provisioning. It can be added to the shared box later without re-architecting.

## At-a-glance matrix

| Dimension | GBPN | GMTC | LIQ | oppe |
|---|---|---|---|---|
| Product | B2B2C business-partner network / marketplace | Talent cloud (ATS + agency + marketplace) | Leadership assessment SaaS (360°) | India URL shortener |
| Frontend | React 19 SPA (Vite), **static** | React 19 (Vite), **static** | React 19 SPA (Vite) **+ Astro landing**, static | **Next.js 15** (SSR/ISR) — needs a Node server |
| Backend | **Hono 4** | **Hono 4** | **Hono** | **Fastify 5** (not Hono) |
| Data layer | Drizzle ORM | `pg` + zod/OpenAPI | Drizzle ORM | **`pg` + raw SQL** (not an ORM) |
| Postgres | **16** | **17** | **16** | **15** |
| Node | 22 | 22 | 22 | **20** |
| Monorepo | pnpm workspaces | pnpm workspaces (`node-linker=hoisted`) | Turborepo | Turborepo |
| Auth | own session + Argon2id | jose JWT + argon2 + TOTP | Better Auth (PG sessions) | own RS256 JWT + argon2 |
| Multi-tenancy | Postgres **RLS** (2 roles) | Postgres **RLS** + txn-local ctx (2 roles) | role-scoped (PG sessions) | per-account rows |
| Worker | **pg-boss — persistent, always-on** | Lambda + SQS + **cron sweeps** | SQS + cron (all **stubbed**, in-proc) | nightly rollup cron (**not wired**) |
| Object storage | **S3 required** (presigned uploads) | **S3 required** (presigned uploads) | S3 for reports (**stubbed**) | none |
| Cache | none (PG) | none (in-proc) | none | Redis designed but **not wired** |
| Payments | Razorpay (stub) | none | none | Razorpay (stub) |
| Other egress | SMTP (stub), Razorpay webhook | **none — deliberately no NAT** | SES (stub) | Razorpay, Safe Browsing, MaxMind |
| Target region | AWS (ap-south implied) | **ap-south-1** | **ap-south-1** | India (ap-south) |
| Prod DB design | RDS + Lightsail container | RDS + **RDS Proxy** | Aurora Serverless v2 + Proxy | managed PG (+ replica) |
| Existing IaC | shell scripts (Lightsail) | **mature Terraform, unapplied** | Terraform **skeleton** (commented) | **none** |
| IaC maturity | low | high | low | none |

## Cross-product observations

- **Not one stack.** Three products are Hono; **oppe is Fastify + Next.js**. All four are Node
  HTTP servers, so they containerize the same way, but oppe's web tier is a running Next.js
  server, not static assets like the other three. This corrects the gemini.md premise ("3
  products, React + Hono + PostgreSQL").
- **Postgres spread 15 → 17.** A single shared server should run **Postgres 17** with one
  **database per product**; PG 17 runs the older products' SQL unchanged in all but exotic cases
  (verify on first migrate).
- **Every product isolates tenants at the DB layer** (RLS or role scoping) and wants a dedicated
  app role separate from the migration/owner role → one shared instance, **database-per-product
  with per-product roles**, never a shared schema.
- **Connection-pooling semantics matter.** GMTC hard-depends on transaction-pooling (it disables
  prepared statements and sets tenant context transaction-locally) — that is exactly what a
  self-hosted **pgbouncer in transaction mode** provides, for free, in place of RDS Proxy /
  Aurora Proxy.
- **Only GBPN needs an always-on worker** (pg-boss). GMTC/LIQ/oppe workers are cron/sweep-driven
  and can be timers. This is the single biggest reason a pure-serverless model does not fit the
  whole portfolio today.
- **S3-compatible object storage** is a hard requirement for GBPN and GMTC (browser presigned
  PUT/GET with CORS), stubbed-but-designed for LIQ, and unused by oppe.
- **Egress / NAT.** GMTC deliberately needs zero outbound so it can skip a NAT gateway (~$35/mo).
  The others do call out (Razorpay, SMTP/SES, Safe Browsing, MaxMind). On a VM with a public IP,
  outbound is free — this sidesteps the NAT-gateway cost that hurts Lambda-in-VPC designs.
- **All target ap-south-1 (Mumbai) / India.** Region is consistent (modulo GMTC's known
  `AWS_REGION=eu-west-1` default bug and LIQ's hard-coded `ap-south-1`).

## Per-product detail

### GBPN — heaviest to host
- Two always-on processes off one image: Hono **API** (:8787) + **pg-boss worker** (:8788, SLA
  sweeps/escalations/mail drain) + one-off **migrate** task. Sessions in PG (Argon2id).
- **~2 GB RAM / 2 vCPU floor for the API alone** — Argon2id at OWASP baseline (19 MiB/hash) means
  a login burst can OOM a 512 MB box. Login is CPU-bound.
- Requires its own Postgres (roles `gbpn` owner / `gbpn_app` NOBYPASSRLS + a `pgboss` schema),
  an S3 bucket (KYC/doc uploads), TLS-terminating proxy (Secure cookie → plain HTTP breaks
  login), and a public Razorpay webhook.
- No Terraform; `hosting/` has Lightsail + Caddy shell scripts (verified locally only).
- **Correction flagged:** `hosting/README.md` / `SCALING.md` claim pg-boss is "not shipped" —
  false; it is a live dependency. And CLAUDE.md (A95: worker → SQS+Lambda) contradicts
  SCALING.md (containers-not-Lambda). Reconcile before building.

### GMTC — most infra-mature
- Lambda **api** + **worker** + **migrate** (arm64), RDS Postgres 17 behind **mandatory RDS
  Proxy**, SQS+DLQ (not in request path — DB rows drained by EventBridge cron sweeps), S3
  presigned uploads. No cache/search/NAT.
- Terraform is complete and thoughtful: AWS **Organization** + 3 accounts (test/stage/prod),
  **GitHub OIDC** deploy roles, modules (network/database/storage/api/worker/migrate/edge/
  observability), `envs/test` wired. **Unapplied — no AWS account exists yet.**
- **RDS Proxy is load-bearing** (txn-local `set_config`, prepared statements off). Single-origin
  `/v1` cookie routing. Homegrown auth (jose/argon2/TOTP).
- This repo is effectively a **ready-made per-product serverless blueprint** for the scale-out
  phase, and its Org/OIDC pattern is reusable portfolio-wide.

### LIQ — lightest to host today
- **Only hard dependency is Postgres.** SQS/SES/S3 are stubbed in-process (`USE_AWS=true`
  *throws*; `LocalQueue`/`LocalEmailProvider`/`LocalStorageProvider` run jobs in-proc). Runs
  today on Postgres + 1–2 Node processes + static hosting.
- API **explicitly designed to move to Fargate/ECS with no rewrite**; workers are plain SQS/cron
  consumers → collapsible to one small container. Extra **Astro static** landing site.
- Terraform `infra/` is a **commented-out skeleton** (no modules/outputs/state) — near-greenfield.
- Prod design leans on **Aurora Serverless v2 (0.5 ACU floor) + RDS Proxy** = per-product
  standing cost that argues for shared Postgres instead. Region hard-coded `ap-south-1`.

### oppe — OUT OF SCOPE (release deferred; reference only, not provisioned now)
- Three small Node processes + Postgres: **web** (Next.js :3000), **api** (Fastify :4000),
  **redirect** (Fastify :8787, latency-critical). Cloudflare Worker is the *intended* redirect
  edge (wrangler scaffold only, not built) — runs as plain Node today, portable later.
- Postgres 15, `pg` + raw SQL. **Redis deferred** (in-memory rate limiter; PG-authoritative
  counters). No object storage, **no email provider** (gap). Razorpay + Safe Browsing + MaxMind.
- **No IaC, no prod Dockerfiles, no deploy pipeline.** Needs apex `oppe.in` + `api.`/`app.`
  subdomains with cross-subdomain cookies on `.oppe.in`.
- Delaying release lowers urgency, not sizing: treat as "3 small Node services + a DB, no special
  services at launch"; add it to the shared box whenever ready.
