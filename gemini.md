# Architectural Decision Record (ADR): Multi-App Hosting Strategy

This document serves as the comprehensive architectural review and decision framework for deploying three distinct products (React + Hono + PostgreSQL) during their initial, low-traffic startup phase. It evaluates the tradeoffs between AWS Lambda and AWS EC2, outlines cost structures, and highlights strategies for database and infrastructure organization.

## 1. Executive Summary & Core Context

• Goal: Host 3 products across 2 environments (Staging and Production) while keeping initial costs as close to zero as possible.
• Stack: React (Frontend), Hono (Backend API), PostgreSQL (Database).
• Core Constraint: Minimal budget (Self-funded startup without immediate income).
• The Problem: Serverless architectures (AWS Lambda) introduce a connection-pooling bottleneck when paired with traditional databases (PostgreSQL), which usually requires costly proxies. Persistent architectures (AWS EC2) eliminate this but introduce manual server maintenance.

## 2. The Comprehensive Architecture Matrix

Dimension	Option A: Serverless Stack (Hono on AWS Lambda)	Option B: Ephemeral Stack + External Serverless DB (Lambda + Neon/Supabase)	Option C: Monolithic Persistent Stack (All-in-One EC2 Instance) — RECOMMENDED
Compute Type	Ephemeral, scales on demand.	Ephemeral, scales on demand.	Persistent, fixed virtual machine (t4g.small).
Database Type	AWS RDS (PostgreSQL).	Neon / Supabase (Serverless Postgres).	Local PostgreSQL (Dockerized inside the same EC2).
Database Connection Strategy	Requires specialized drivers or expensive AWS RDS Proxy to avoid crashing on Lambda scale-ups.	Bypassed via native HTTP/WebSocket connection pooling APIs.	Standard TCP connection pooling works natively out of the box.
Network Latency	🟢 Ultra-Low (<2ms): Both compute and DB live within the same AWS VPC/Region.	⚠️ Moderate (20ms–40ms): Cross-cloud latency (AWS to Neon/Supabase).	🚀 Zero Latency: Network hops are entirely internal (localhost).
Hidden Costs	💸 High: Requires AWS API Gateway and a NAT Gateway ($32+/mo) to allow VPC Lambdas to talk to external APIs.	📉 Low Initially: Bypasses NAT Gateways if run outside VPC, but API Gateway fees apply after 1M requests.	🟢 None: Public IPv4 data transfer is the only minor variable.
Monthly Cost (3 Apps × 2 Envs)	$45 - $60+ / month	$0 / month (Until free tiers expire, then jumps to $90 - $150/mo).	💰 $16 - $24 / month total (Two tiny EC2 instances: one for Staging, one for Prod).
Infrastructure Complexity	🌪️ High: Demands advanced AWS VPC networking, subnets, and IAM configurations.	🛠️ Moderate: Requires managing multiple cloud providers and tokens.	🪵 Low: Standard Linux administration using Docker Compose and Nginx.

## 3. Deep Dive into the Options

- **Option A:** Pure AWS Serverless (Hono on Lambda + AWS RDS)
While Lambda's compute layer falls within a generous free tier, placing Lambda inside a private VPC to securely connect to AWS RDS introduces severe hidden costs. A NAT Gateway is required to allow those Lambdas to reach the public internet (for Stripe, Auth0, etc.), costing a minimum of ~$32/month just to sit idle.
- **Option B:** Hybrid Serverless (AWS Lambda + Third-Party Serverless DB)
Using providers like Neon or Supabase handles connection pooling over HTTP seamlessly. However, because your code sits on AWS and your database sits on another cloud, every database query suffers a 20-40ms roundtrip penalty. Furthermore, mapping 3 apps × 2 environments across individual tier accounts means you will rapidly exceed free limits, forcing you into expensive per-project subscription tiers.
- **Option C:** The "All-in-One" EC2 Box (Highly Recommended for Bootstrapping)
You host all three Hono applications and a single PostgreSQL instance containerized inside one single EC2 instance for production, and a second identical small instance for staging.
• Logical Separation: You run a single PostgreSQL server instance, but inside it, you create three completely distinct databases (db_product_a, db_product_b, db_product_c).
• Isolation: You use unique database users and passwords for each app, ensuring App A cannot read App B's data.

## 4. Architectural Cleanliness & The Migration Plan

Opting for the All-in-One EC2 Box does not lock you into a monolithic nightmare. Because Hono is runtime-agnostic, moving to a fully distributed, high-scale architecture later requires zero application code changes.

When a specific app experiences massive traction, follow this clean migration path:

```
[ Step 1: Baseline ]    EC2 Instance ──> Hosts [App A, B, C] + Local Postgres [DB A, B, C]
                                                │
[ Step 2: Scale DB ]     EC2 Instance ──> Hosts [App A, B, C] ──(Migrate Data)──> Dedicated AWS RDS Instance
                                                │
[ Step 3: Scale Apps ]   App A (Dockerized) ──> Moved to AWS Lambda / AWS App Runner
```

1. Decouple the Database: Run a standard pg_dump on the EC2, spin up an AWS RDS instance, run pg_restore, and update the app's environment variable.
2. Decouple the Compute: Wrap the Hono app into a Docker container and point it toward AWS App Runner or AWS Lambda.

## 5. Infrastructure as Code (IaC) Consolidation Strategy

The Current State: Multiple Terraform Projects
Currently, having separate Terraform projects for each application results in duplicated resource definitions (VPCs, Security Groups, IAM Roles) and makes it incredibly difficult to share resources like a single database server or an EC2 host instance.

The Target State: A Unified Terraform Monorepo

You should consolidate your infrastructure into a single Terraform project, organized using Terraform Workspaces or Directories to handle environments (Staging vs. Production).
Recommended Directory Structure:

```text
infra/
├── modules/
│   ├── compute/          # Reusable EC2 configuration (Instance, IAM, EIP)
│   ├── network/          # Reusable VPC, Subnets, Security Groups
│   └── database/         # Optional (For when you migrate to RDS later)
├── staging/
│   ├── main.tf           # Instantiates network & compute modules for Staging
│   ├── variables.tf
│   └── outputs.tf
└── production/
    ├── main.tf           # Instantiates network & compute modules for Prod
    ├── variables.tf
    └── outputs.tf
```

Use code with caution.

Why This Works Better:

- Resource Sharing: You can write a single EC2 resource block in production/main.tf and pass its configuration down to all three application deployment steps.
- DRY (Don't Repeat Yourself): Your networking configurations and security rules are defined once in the modules/ folder and inherited identically by both Staging and Production layers, eliminating environment drift.

## 6. Actionable Next Steps

To finalize this decision, evaluate the following implementation mechanics with your AI assistant:

1. Would you like a production-ready Docker Compose and Nginx configuration blueprint demonstrating how to run 3 Hono apps and 1 Postgres database securely on a single EC2 instance?
2. Do you want to draft the Unified Terraform Main File (main.tf) that provisions this shared EC2 architecture for your staging and production environments?
