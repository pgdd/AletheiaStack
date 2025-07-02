# 🚀 AletheiaStack.com (Stateless AI Safety Blog)

This project sets up a **minimal, serverless, cost-efficient blog infrastructure** using AWS services, with the intention of supporting future AI Safety–oriented features.

> ✅ This is a technical starter kit for building a **secure, observable, resilient publishing system**. No LLM or AI-specific components are included at launch.

---

## 🧽 Project Goals

- ✅ Stateless architecture (no containers, no persistent compute)
- ✅ Fully serverless on AWS (S3, API Gateway, Lambda, DynamoDB, Route 53)
- ✅ Designed for 100+ concurrent authors
- ✅ No caching (CloudFront is excluded by design for full observability)
- ✅ Future-ready for AI Safety workflows
- ✅ Git workflow designed for **atomic commits** and clean staging

> This blog isn't just about content — it's about demonstrating **operational discipline** suitable for future LLM applications.

---

## 🌐 Architecture Overview

```bash
		                         [ Authors / Users ]
		                                |
		                                v
		                         +---------------+
		                         |  Route 53 DNS | <== Domain: aletheiastack.com
		                         +---------------+
		                                |
		                                v
		                         +----------------+
		                         |  Frontend SPA  | <== React / Next.js (SSR / SSG)
		                         +----------------+
		                                |
		                                v
		                    +-------------------------+
		                    |   Amazon CloudFront     | <== CDN & caching (optional)
		                    +-------------------------+
		                                |
		                                v
		              +--------------------------------------+
		              |        Amazon API Gateway (REST)     |
		              +--------------------------------------+
		                                |
		                                v
		                         +-------------+
		                         | AWS Lambda  | <== Stateless backend (Node.js)
		                         +-------------+
		                                |
		         +----------------------+--------------------+
		         |                                           |
		         v                                           v
		 +---------------------+                  +---------------------+
		 |     Amazon S3       | <== Static       |    DynamoDB         | <== Metadata: posts, authors
		 | (images, HTML, etc) |    content       |    (NoSQL DB)       |
		 +---------------------+                  +---------------------+
```

---

## 🖥️ Why No EC2 (for Now)?

We deliberately avoid EC2 or containerized compute in early phases:

| Goal | Main point |
|--------|---------------|
| Stateless Architecture | All logic is executed per-request via Lambda; no persistent state needed |
| No Maintenance Overhead | EC2 requires patching, scaling, monitoring — not justified for our use case |
| Cold Start Tolerance | Latency is acceptable at current scale (<100 authors, ≤ ~10 req/s) |
| Billing Model | Pay-per-use with Lambda is far cheaper than reserved or on-demand EC2 |
| Observability | Full traceability is built-in with X-Ray, CloudWatch, API Gateway |

🛑 **We will revisit EC2 or container options only if:**

- Latency becomes unacceptable under load
- Stateful operations emerge (e.g. long-lived processes, queues)
- Heavy LLM inference or GPU-bound tasks are introduced

> If that day comes we will weigh **AWS Fargate** or **App Runner** (fully managed containers) alongside traditional EC2.

> This keeps the system minimal, observable, and deployable from day one

---

## 💰 Cost Estimation (Baseline)

| Component        | Estimate @ 100 authors               |
|------------------|---------------------------------------|
| Lambda           | 100 reqs × 300ms @ 128MB = ~$0.00025 |
| API Gateway      | 100 REST calls = ~$0.00004           |
| DynamoDB         | On-demand write = ~$0.005–$0.01      |
| S3               | 100 PUT + 100 GET = ~$0.01           |
| Route 53         | ~$0.50/month for hosted zone         |
| Total            | **<$0.02 per content update cycle**  |

*Estimates exclude the AWS free tier (e.g., 1 M Lambda requests and 400 k GB-s of compute per month).*

> 🌟 Goal: keep all dynamic ops under $5/month before AI integration.

---

## 🧪 Observability Checklist

- [x] Logs (Lambda, API Gateway)
- [x] Tracing (X-Ray enabled)
- [ ] CloudWatch dashboards for ops
- [ ] Alert on cost overage (Budgets + SNS)

---

## 👣 Example User Flow

```bash
1. Reader lands on blog URL (via aletheiastack.com / Route 53)
2. Optional: CloudFront edge location forwards request
3. Frontend (SPA via S3) is loaded into browser
4. User navigates: homepage, author page, post detail, etc.
5. Client issues API request to API Gateway
6. Lambda fetches post metadata from DynamoDB
7. Images or content blobs pulled directly from S3
8. React frontend renders full post
9. CloudWatch logs + X-Ray traces are captured
10. Entire lifecycle is observable + cost-bounded
```

---

## 🔧 Tech Stack

| Layer             | Tech                           | Notes                                                     |
|------------------|--------------------------------|-----------------------------------------------------------|
| DNS & Domain      | Route 53 + aletheiastack.com   | AWS-managed domain & routing                             |
| Frontend         | Next.js (React, SSR, SSG)       | Deployed to S3 via static export or SSR (via Lambda@Edge) |
| API Gateway      | Amazon API Gateway              | Connects `/api/*` HTTP calls from the SPA to AWS Lambda (Next.js runtime bypassed)    |
| Compute          | AWS Lambda                      | Stateless backend logic, triggered by API Gateway         |
| Storage (media)  | Amazon S3                       | Blog images and static content                            |
| Metadata store   | DynamoDB                        | Posts, authors, versioning                               |
| CDN (optional)   | CloudFront                      | Optional edge delivery if needed                          |

> 🔄 In **Next.js 15**, new App Router APIs and enhanced `server-actions` allow for more flexibility. But in this setup, all backend API logic is externalized through API Gateway → Lambda for full observability, tracing, and platform independence.

## 🛣️ Next.js 15 API Features vs. Amazon API Gateway (REST)

Next.js 15 ships with the shiny **App Router**, **Server Actions**, and in-process **edge/serverless functions**. For **AletheiaStack** we deliberately **leave those runtime APIs turned off** and route every `/api/*` call straight to **Amazon API Gateway (REST)**. Why?

1. **Observability by default** — Gateway + Lambda give us CloudWatch Logs, X-Ray traces, and discrete metrics on each invocation.
2. **AWS-native guardrails** — IAM, throttling, WAF rules, and API Keys live in API Gateway. Next's built-ins would skip that layer.
3. **Cost & cold-start profile** — We pay ~0.4 ¢ per 1 k REST calls; Lambda is billed in 1 ms blocks. No vendor lock-in.
4. **Hard separation of concerns** — The React SPA stays a pure frontend artifact. The backend is language-agnostic and can evolve independently.
5. **Future multi-tenant authors** — Fine-grained rate limiting and auth live at the edge of our system, not inside the Node runtime.

### How the request flow works

```bash
Browser → Route 53 (api DNS) → API Gateway → Lambda → DynamoDB → Lambda → API Gateway → Browser
```

1. During `next build && next export` **no Next.js API code is generated**.
2. The browser issues `fetch("/api/posts")` (or any REST path).
3. DNS or a base path mapping sends that request to API Gateway, skipping the Next.js runtime entirely.
4. API Gateway maps `/posts` to a Lambda function that executes JavaScript/TypeScript logic.
5. The JSON payload returns to the SPA, which updates React state in the usual way.

If we later want to experiment with **Next.js Server Actions** or Vercel-style edge functions, we simply swap the DNS target or add rewrite rules. **The contract stays `/api/*`.**

> 🗒️ **Keep it simple:** at launch we have exactly three moving parts — S3 (static site), API Gateway + Lambda (JSON), and DynamoDB (metadata). Everything else can be bolted on when justified.

---

## ❌ Why No Caching (Yet)?

Caching (CloudFront, Lambda@Edge) is disabled in Phase 1:
- ⚠️ Full observability is required — every request must hit origin
- ⚠️ No risk of stale content  especially important if AI-generated
- ✅ Easier debugging, auditability, and logging

> In future phases, caching may be added with full control over consistency and freshness. For example:
>
> - **CloudFront + Lambda@Edge**: for SSR delivery with traceable invalidations
> - **Redis / ElastiCache**: for ephemeral query acceleration (e.g. search, feeds)
> - **DAX** (DynamoDB Accelerator): low-latency read-through cache for hot keys

Caching will only be enabled once content safety guarantees, freshness controls, and real-time logging are in place.

---

## 🤔 Why DynamoDB (and not RDS)?

DynamoDB is the preferred choice for serverless, highly concurrent workloads:

| Feature           | Description                                                             |
|------------------|-------------------------------------------------------------------------|
| On-Demand Mode   | Auto-scales to match request traffic instantly                          |
| Provisioned Mode | Manual control of read/write capacity units (RCUs/WCUs)                |
| Partitions       | Each partition supports 3,000 RCU or 1,000 WCU and 10GB of storage      |
| Hot Partition    | Avoid overloading same partition key; ensure key distribution          |

> **Best practice:** we use distributed IDs or composite keys to spread writes evenly.

Other advantages:
- **No maintenance**: no servers, patches, or backups to manage
- **Pay-per-request**: scales cost-efficiently from zero to high scale
- **Built-in retry/backoff**: handles transient errors automatically
- **Replication**:
  - **Within-region**: all data is **synchronously replicated across 3 Availability Zones** by default (multi-AZ), ensuring high availability and durability
  - **Cross-region**: optional via **Global Tables**, allowing multi-region sync with last-writer-wins conflict resolution
- **Point-In-Time Recovery (PITR)**: allows rollback to a previous good state

---

## 📊 DynamoDB Schema (ASCII)

```bash
				                    +---------------------+
				                    |     Posts Table     |
				                    +---------------------+
				                    | PK: POST#<id>       |
				                    | SK: META            |
				                    | title               |
				                    | author_id           |
				                    | created_at (ts)     |
				                    | updated_at (ts)     |
				                    +---------------------+
				                    | PK: POST#<id>       |
				                    | SK: BODY            |
				                    | content             |
				                    +---------------------+

				                    +---------------------+
				                    |   Authors Table     |
				                    +---------------------+
				                    | PK: AUTHOR#<id>     |
				                    | SK: PROFILE         |
				                    | name                |
				                    | bio                 |
				                    | joined_at (ts)      |
				                    +---------------------+
```

#### 🔍 Schema Design Notes

- We use **high-cardinality partition keys** (e.g. `authorId#timestamp`) to spread writes evenly
- We consider **write sharding** for high-throughput writes (e.g. `articleId#shard01 ... shard10`)
- We use **composite keys** to separate metadata from content
- We include **timestamps** for ordering, conflict resolution, and change tracking
- We avoid hot partitions to preserve latency SLAs
- All data is **replicated across 3 Availability Zones by default**

---

## 👩‍💻 Why Next.js (not NestJS yet)?

| Criteria            | Next.js                        | NestJS                                  |
|--------------------|--------------------------------|------------------------------------------|
| Primary Purpose     | Static/SSR frontend            | Structured backend logic                 |
| Serverless Ready    | ✅ Easily deploys to S3/CDN     | ❌ Harder to use with Lambda per route   |
| Code Complexity     | Simple, clean React-based      | Requires full backend project setup      |
| Dev Velocity        | High, minimal config           | Slower, requires modules & boilerplate   |
| LLM Integration     | Easy via `pages/api/` routes   | Can be done, but overkill at first       |
| When to Consider    | N/A                            | When we need complex backend workflows, auth, or persistent services |

> ✅ **Next.js is the perfect fit for frontend-first, stateless publishing**. We can later integrate LLMs using lightweight serverless endpoints.

---

## 🧱 Stateless Frontend-Backend Separation

Our architecture deliberately separates the **frontend (Next.js)** from any backend logic beyond basic API handling. By deploying frontend assets to **S3**, routing via **API Gateway**, and executing business logic via **Lambda**, we achieve a **fully stateless, scalable, and observable system** — without the need for EC2 or containers.

This works well **as long as:**

- all compute can remain short-lived and event-driven,
- authentication/state is minimal,
- API traffic stays within burstable limits.

> 🧠 If in the future introducing **long-lived services**, **persistent sessions**, **queuing systems**, or **complex orchestrations** (e.g. background processing, media conversion, or LLM chaining), then a structured backend like **NestJS** running on **EC2 or containers** may be a right option — _but only once justified by workload complexity_.

---

## 🌪 Backend Evolution (Future-Proofing)

We may eventually transition to a structured backend (e.g., NestJS on ECS/Fargate or EC2) if:

- API grows complex (auth, RBAC, workflows)
- Persistent connections / streaming / queuing are required
- High throughput or background jobs emerge

Until then, Lambda + Next.js API routes provide the optimal velocity and observability.

---

## 🚧 GitFlow + Atomic Commit Philosophy

> We adopt **GitFlow-lite** principles with explicit separation among `main` (production), `staging` (release-candidate testing), and `dev` (ongoing integration). All changes are atomic.

### Branch Workflow

- `main`: production deployable, CI/CD to prod
- `staging`: release-candidate testing, CI/CD to dedicated staging stack
- `dev`: merge feature branches here (integration only)
- `feat/*`, `fix/*`, `doc/*`: atomic branches per scope

---

## ⚙️ CI/CD & GitFlow Pipeline (ASCII Diagram)

```bash
	                             [ dev / feat/* branches ]
                                           |
                                           v
                                     +------------+
                                     |   Staging  | <== Preview stack (S3, API, DB)
                                     +------------+
                                           |
                                           v
                                     Manual QA / review
                                           |
                                           v
                                     +------------+
                                     |   Main     | <== Production stack (S3, API, DB)
                                     +------------+
                                           |
                                           v
	                             GitHub Actions / CI Runner
                                           |
                                           v
                             +--------------------------+
                             | Build + Deploy to AWS    |
                             | (S3, API Gateway, etc.)  |
                             +--------------------------+
```

> CI/CD relies on GitHub Actions, using workflows triggered on pushes to `main` or `staging`, deploying the full stack via AWS CLI or CDK.

All deploys are reproducible and tracked. It will also validate schema compatibility and metadata diffs before every release.

---

## 👤 Pierre Giddio

Maintained by **Pierre Giddio**, AI Safety engineer & technologist.

> 🌀 Project codename: **AletheiaStack** — from the Greek "ἀλήθεια" (truth, disclosure).
