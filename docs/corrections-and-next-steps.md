# Corrections & next steps

Companion to [0001-hosting-strategy.md](0001-hosting-strategy.md). Active scope: **GBPN, GMTC,
LIQ** (oppe deferred / out of scope).

## A. Corrections to the gemini.md premise

gemini.md ([../gemini.md](../gemini.md)) is directionally right (Option C — shared VM + Dockerized
Postgres) but rests on a few inaccurate assumptions:

1. **"3 products, all React + Hono + PostgreSQL."** True enough for the three in scope — all Hono,
   all Postgres, all static React frontends. (For the record, the deferred oppe is Fastify +
   Next.js + PG 15 + raw SQL — so the "one stack" assumption does not generalize to the whole org.)
2. **`t4g.small` (2 GB) per box is undersized.** GBPN's API alone wants ~2 GB under an Argon2id
   login burst. A shared box for three apps + Postgres + pgbouncer + Caddy needs **~4 GB
   (`t4g.medium`)**. gemini.md's own $16–24/mo range still roughly holds at that size on AWS.
3. **"Two boxes now (staging + prod)"** is premature pre-revenue. Build **one shared test/staging
   box** first; add prod only when a product has real users.
4. **The DB is not casually shareable as one schema.** Every product uses RLS / a dedicated
   `NOBYPASSRLS` app role. Correct model = one Postgres **instance**, **one database + role set per
   product**.
5. **RDS Proxy / connection-pooling concern is real but cheaply solved** — self-hosted **pgbouncer
   (transaction mode)** on the box gives GMTC the pooling semantics it hard-depends on, at no cost.
6. **NAT-gateway cost is avoided by construction** on a public-IP VM — no need to frame it as a
   Lambda-VPC tax to pay.

## B. Per-product issues to fix before they hit shared infra

### GBPN
- **Stale hosting docs:** `hosting/README.md` / `SCALING.md` claim pg-boss / background jobs are
  "planned, not shipped" — **false**; pg-boss 12 is a live dependency and `apps/api/src/worker.ts`
  exists. Update before using those docs to plan the deploy.
- **Contradiction to reconcile:** CLAUDE.md A95 (worker → SQS+Lambda) vs `SCALING.md`
  (containers-not-Lambda). The shared-box plan assumes **containers** (pg-boss worker as an
  always-on container) — confirm that's the intended near-term runtime.
- **First-admin bootstrap:** a migrated-but-unseeded DB has no admin login, and `db:seed` is
  destructive (truncates 49 tables, plants a well-known password). Need a safe first-admin script
  before any real deploy.
- **Rotate the in-repo `gbpn_app` password** (`gbpn_app`) immediately post-migration.
- Prod requires **TLS** (Secure cookie) — Caddy covers this; never expose the API over plain HTTP.

### GMTC
- **Region mismatch:** infra targets `ap-south-1` but `gmtc-backend` defaults `AWS_REGION=eu-west-1`
  — align before deploy.
- **Secrets in Terraform state** (JWT/MFA keys as Lambda env in test) — must move to Secrets
  Manager hydration before stage; relevant if shared tooling/state is introduced.
- **`db.t4g.micro` unverified** against RDS Proxy + Performance Insights — may need `t4g.small`.
  (Moot on the shared box, but relevant at scale-out.)
- **Incomplete deploy pipeline:** `terraform.yml`, `deploy-backend.yml`, `deploy-frontend.yml` are
  referenced by IAM trust policies but not yet written (renaming a workflow is an IAM change).
- **Reusable asset:** GMTC's AWS **Organization + accounts + GitHub-OIDC** pattern is the best
  starting point for the portfolio's account/CI backbone — adopt it rather than reinventing.

### LIQ
- **`infra/` Terraform is a commented-out skeleton** — treat as a design sketch, not deployable.
- **AWS providers unimplemented in code** (`USE_AWS=true` throws). The provider seam
  (`packages/core/src/providers`) is where to point queue/email/storage at shared equivalents
  (shared Postgres, on-box SMTP catcher, MinIO/S3) without touching route/worker code.
- **Region** hard-coded `ap-south-1` across infra vars — fine (matches GMTC), just note it's fixed.
- **Better Auth pinned `~1.3.9`** (bumping needs a zod 3→4 + drizzle 0.38→0.45 migration) — don't
  assume a version bump is free.

## C. Portfolio-wide standardization to decide

- **Postgres 17** as the shared server version (GBPN/LIQ are on 16, GMTC on 17). Verify each
  product's migrations apply cleanly against PG 17 on first run.
- **Region ap-south-1 (Mumbai)** for the shared box (all three already target it).
- **Subdomain scheme** per product per env, e.g. `gbpn.test.<domain>`, `gmtc.test.<domain>`,
  `liq.test.<domain>` — need a parent domain + DNS (Route53 or Cloudflare). **Open: what domain?**
- **Deploy mechanism**: GitHub Actions → SSH `docker compose pull && up` on the box, using the
  OIDC/account pattern borrowed from GMTC. Confirm.
- **Backups from day one**: nightly `pg_dump` per database, copied off-box (to S3). No managed
  backups when self-hosting Postgres.

## D. Decisions (confirmed 2026-09-29)

1. **Cloud/host:** ✅ **AWS** (Lightsail/EC2 `t4g.medium`, ~$18–28/mo).
2. **Scope now:** ✅ **one shared test/staging box** (GBPN, GMTC, LIQ); prod deferred.
3. **Object storage:** ✅ **real per-product AWS S3 buckets** (not MinIO).

Still open:

- **Parent domain / DNS provider** for the per-product subdomains (Route53 vs Cloudflare, and
  which domain) — not yet known; blocks TLS + subdomain routing.

## E. Suggested build order (once decisions above are settled)

1. Stand up the repo skeleton: `modules/{compute,network,dns}`, `compose/`, `envs/test/`
   (see [0001-hosting-strategy.md](0001-hosting-strategy.md#infrastructure-as-code-plan)).
2. Terraform the shared box: `t4g.medium` VM + EIP + security group + cloud-init (Docker +
   Compose), DNS records, per-product **S3 buckets** (with CORS) + an instance role granting the
   box scoped access to them.
3. Author `compose/docker-compose.yml`: Caddy + pgbouncer + Postgres 17, then per-product
   `api` / `worker` containers built from each repo's image (object storage points at real S3).
4. Create per-product databases + owner/app roles; wire `DATABASE_URL`s through pgbouncer.
5. Fix the per-product blockers in §B (GBPN first-admin + password rotation; GMTC region; LIQ
   provider seam).
6. CI: GitHub Actions build → push image → SSH deploy; nightly `pg_dump` → S3.
7. Point each product's `WEB_ORIGIN` / API URL / cookie settings at its subdomain; verify TLS +
   login end-to-end per product.
