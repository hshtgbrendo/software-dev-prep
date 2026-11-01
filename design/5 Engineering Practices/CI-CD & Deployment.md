---
tags: [design, engineering-practices, devops]
aliases: [CI/CD, Deployment Strategies, Docker, Kubernetes, Feature Flags, Infrastructure as Code, Git Workflow]
---
# CI/CD, Deployment & Infrastructure

> [!summary] What you'll learn
> How code gets from your laptop to production **safely, many times a day**: **Git workflows** (trunk-based vs GitFlow), **continuous integration and delivery**, **deployment strategies** (rolling, blue-green, canary, shadow), **feature flags**, **zero-downtime database migrations**, **containers (Docker)** and **orchestration (Kubernetes)**, **infrastructure as code**, environments and the **Twelve-Factor App**, and a map of the main **cloud services**.
>
> **Why it matters:** most production incidents are caused by changes. These practices make each change small, verified, observable, and quickly reversible. You'll use them daily at any company.

Back to [[Design Index]] · Related: [[Reliability & Observability]] · [[Testing Strategy]]

---

## Version control workflows

### Trunk-based development
Everyone integrates small changes into the main branch (`main`/"trunk") **frequently**: at least daily, via short-lived branches (hours to a day or two) and quick reviews. **Unfinished features are merged but hidden behind feature flags.**
- ✅ Avoids painful long-lived branches and "merge hell"; keeps everyone working against current code; enables continuous delivery.
- ✅ Used at **Google, Meta**, and most high-performing teams; recommended by the **DORA** research program.
- ❌ Requires good automated tests, fast CI, and feature-flag discipline.

### GitHub flow
A lightweight variant: create a feature branch → open a PR → review and CI → merge to `main` → deploy. Very common for web apps.

### GitFlow
Long-lived `main` and `develop` branches plus `feature/*`, `release/*`, and `hotfix/*` branches.
- ✅ Structured for software with **versioned releases** (mobile apps that go through app-store review, desktop software, libraries, on-premise products).
- ❌ Heavyweight for continuously deployed web services; long-lived branches diverge.

### Git skills you're expected to have
- **Merge vs rebase**: merge preserves history with a merge commit; rebase replays your commits on top of the latest main for a linear history. Never rebase shared branches that others have pulled.
- **Squash merging**: combines a PR's commits into one on main (common default).
- **Resolving conflicts**, `git stash`, `git cherry-pick`, `git revert` (safe undo on shared branches) vs `git reset` (rewrites history).
- **`git bisect`**: binary-search through history to find the commit that introduced a bug.
- **Conventional commits**: `feat: add idempotency keys`, `fix: handle empty cart`, `chore:`, `docs:`. Enables automated changelogs and versioning.
- **Semantic versioning (SemVer)** for libraries and APIs: `MAJOR.MINOR.PATCH`. Increment **MAJOR** for breaking changes, **MINOR** for backward-compatible features, **PATCH** for backward-compatible fixes. (`^1.4.2` in package.json accepts any 1.x ≥ 1.4.2.)

---

## CI/CD

### Continuous Integration (CI)
**The problem:** when developers integrate rarely, integration becomes a painful, bug-filled event, and broken code sits unnoticed for days.

**The practice:** every push and every pull request automatically triggers a pipeline that **builds** the code and runs **linters, type checks, unit and integration tests, and security scans**. Code merges only when the pipeline is **green**. Problems are caught within minutes of being introduced, by the person who introduced them.

### Continuous Delivery vs Continuous Deployment
- **Continuous Delivery**: every change that passes the pipeline is **deployable** at any time. Releasing to production is a deliberate button click.
- **Continuous Deployment**: every change that passes the pipeline is **automatically deployed** to production, with no human gate. Requires strong automated testing, monitoring, and automatic rollback.

### A typical pipeline
```mermaid
flowchart LR
  A[Push / PR] --> B[Install + build] --> C[Lint, type check,<br/>unit tests] --> D[Integration tests] --> E[Security scans:<br/>deps, SAST, secrets] --> F[Build artifact<br/>Docker image, tagged] --> G[Deploy to staging] --> H[E2E + smoke tests] --> I[Canary in production] --> J[Full rollout]
  I -. metrics degrade .-> RB[Automatic rollback]
```
- **Build once, deploy the same artifact everywhere**: the exact Docker image tested in staging is the one promoted to production (tagged with the git commit SHA). Rebuilding per environment risks deploying something different from what was tested.
- **Keep CI fast** (ideally under ~10–15 minutes): cache dependencies, parallelize tests, run the slowest suites only where needed.
- **Tools:** GitHub Actions, GitLab CI, Jenkins, CircleCI, Buildkite; deployment tools like **Argo CD** (GitOps), Spinnaker, AWS CodePipeline.

```yaml
# .github/workflows/ci.yml: a minimal Node.js CI pipeline
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test }
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run lint && npm run typecheck
      - run: npm test -- --coverage
        env: { DATABASE_URL: postgres://postgres:test@localhost:5432/postgres }
```

### DORA metrics (how engineering performance is measured)
From Google's DORA (DevOps Research and Assessment) program, the four key metrics:
| Metric | Meaning | Elite teams |
|---|---|---|
| **Deployment frequency** | How often you deploy to production | On demand (many times per day) |
| **Lead time for changes** | Commit → running in production | Less than a day |
| **Change failure rate** | % of deploys causing a failure needing remediation | Low (roughly 0–15%) |
| **Time to restore service** | How quickly you recover from a failure | Less than an hour |

The research finding: **speed and stability go together**. Teams that deploy small changes frequently have *fewer* and *shorter* failures than teams doing big, infrequent releases.

---

## Deployment strategies

**The problem:** deploying new code means replacing running instances. Done naively, that causes downtime, and if the new version is broken, every user is affected at once. Deployment strategies control **how** traffic moves to the new version.

### Recreate
Stop all old instances, then start the new ones.
- ✅ Simple; no two versions running at once.
- ❌ **Downtime** during the switch. Only acceptable for dev environments or batch systems.

### Rolling update
Replace instances **a few at a time** (e.g., 25% at a time): start new ones, wait until they pass health checks, shift traffic, stop old ones, repeat.
- ✅ **No downtime**; no extra infrastructure. The **default in Kubernetes** Deployments.
- ❌ **Old and new versions run side by side** during the rollout, so they must be compatible (APIs, database schema, message formats). Rolling back means another rolling update (slower).

### Blue-green deployment
Run two identical production environments. **Blue** serves all traffic (current version); deploy the new version to **green**, test it, then **switch all traffic** to green at once (load balancer or DNS change). Blue stays ready as a fallback.
```mermaid
flowchart LR
  U[Users] --> LB{Load balancer / router}
  LB -->|"100% (switch)"| G["Green: v2 (new)"]
  LB -.->|"0%, standby for rollback"| B["Blue: v1 (old)"]
```
- ✅ **Instant cutover and instant rollback** (switch back to blue). Can test green in production conditions before switching.
- ❌ **Double the infrastructure** during deploys. All users switch at once, so a bug not caught in testing hits everyone. Database changes must work with both versions.

### Canary release
Route a **small percentage** of real traffic (e.g., 1%) to the new version, **compare its metrics** (error rate, latency, business metrics) against the stable version, and **gradually increase** (1% → 5% → 25% → 50% → 100%) if healthy. Roll back automatically if metrics degrade. (Named after canaries in coal mines, which warned miners of danger.)
- ✅ **Limits the blast radius**: a bad release affects only a small fraction of users, briefly. Validates with real traffic.
- ❌ Needs good **monitoring**, traffic-splitting infrastructure (service mesh, LB weights, or feature flags), and automation for analysis and rollback (Argo Rollouts, Flagger, Spinnaker's Kayenta).
- The **standard at large tech companies** for user-facing services.

### Shadow (dark) launch / traffic mirroring
**Copy** real production traffic to the new version, but **discard its responses** (users only see the old version's responses). Compare outputs and performance.
- ✅ Tests the new code under real load and real inputs with zero user impact. Great for rewrites and migrations ("does the new service return the same results?").
- ❌ Side effects (writes, emails, payments) must be disabled or sandboxed in the shadow; doubles load.

### A/B testing
Route users into cohorts that see different **product variants** and compare **business outcomes** (conversion, engagement). It's a product experimentation technique, not primarily a deployment safety mechanism, though it uses similar traffic-splitting machinery (usually feature flags).

| Strategy | Downtime | Rollback speed | Extra cost | Blast radius of a bad release | Typical use |
|---|---|---|---|---|---|
| Recreate | Yes | Slow | None | Everyone | Dev/test |
| **Rolling** | No | Medium | None | Grows as rollout proceeds | Default for many services |
| **Blue-green** | No | **Instant** | 2× during deploy | Everyone (after switch) | Critical apps needing instant rollback |
| **Canary** | No | Fast | Small | **Small** | Large-scale user-facing services |
| Shadow | No | N/A | Extra load | None | Rewrites, migrations, risky changes |

---

## Feature flags (toggles)

### The problem they solve
Without flags, **deploying** code and **releasing** a feature are the same event, so a half-finished feature can't be merged, every release is all-or-nothing, and turning off a broken feature requires an emergency redeploy.

### How they work
Wrap new behavior in a runtime check against a flag service:
```javascript
if (await flags.isEnabled("new-checkout-flow", { userId: user.id, country: user.country })) {
  return renderNewCheckout(cart);
}
return renderLegacyCheckout(cart);
```
Flags are configured in a dashboard or config service and evaluated per request, often with **targeting rules** (internal employees first, then 5% of users, then a country, then everyone).

### What they enable
- **Decouple deploy from release**: merge and deploy unfinished code safely ("dark"), and release when ready, which is what makes trunk-based development work.
- **Gradual rollouts** and **canary releases by user cohort**.
- **Kill switches**: instantly disable a feature causing an incident, without a deploy.
- **Dogfooding**: employees use new features before customers.
- **A/B experiments**.
- **Entitlements**: features by plan (premium-only).

### Pitfalls
- **Flag debt**: hundreds of stale flags make code hard to read and test (every flag doubles the number of possible code paths). **Remove flags** once a rollout completes; give each flag an owner and an expiry date.
- Test both paths while a flag is active.
- Flag evaluation must be fast and resilient (cache flag values locally; define safe defaults if the flag service is down).

**Tools:** LaunchDarkly, Unleash, Statsig, Split, Flagsmith, OpenFeature (a vendor-neutral standard), or in-house systems (most large companies have one).

---

## Zero-downtime database migrations: expand → migrate → contract

### The problem
During a rolling or canary deploy, **old and new versions of the app run at the same time** against the **same database**. If you rename a column in one step (`ALTER TABLE users RENAME COLUMN name TO full_name`), the old version (still running) immediately breaks because `name` no longer exists. And if you need to roll back, the old code can't run against the new schema.

### The solution: make every change backward compatible, in phases
Example: renaming `users.name` to `users.full_name`.
1. **Expand**: add the new column (nullable, no default that rewrites the table). Deploy app code that **writes to both** `name` and `full_name` but still **reads** `name`.
2. **Migrate (backfill)**: copy existing data from `name` to `full_name` in **batches** (avoid one giant locking update). Then deploy code that **reads** `full_name` (still writing both, for rollback safety).
3. **Contract**: once all running code uses only `full_name` and you're confident you won't roll back, stop writing `name`, and in a **later** deploy **drop** the old column.

```sql
-- Step 1 (expand)
ALTER TABLE users ADD COLUMN full_name TEXT;
-- Step 2 (backfill in batches, repeated until 0 rows updated)
UPDATE users SET full_name = name WHERE id IN (SELECT id FROM users WHERE full_name IS NULL LIMIT 10000);
-- Step 3 (contract, a later release)
ALTER TABLE users DROP COLUMN name;
```

**More rules for safe migrations:**
- **Additive changes** (new tables, new nullable columns, new indexes) are safe; **destructive changes** (drop, rename, change type, add NOT NULL) need the phased approach.
- **Build indexes without locking writes** (Postgres `CREATE INDEX CONCURRENTLY`).
- Beware long-running locks on big tables; test migrations against production-sized data.
- Run migrations as a **separate, automated pipeline step**, version-controlled with the code.
- **Tools:** Flyway, Liquibase, Prisma Migrate, Knex/Sequelize/TypeORM migrations, Rails/Django migrations, gh-ost and pt-online-schema-change for large MySQL tables.

The same expand/contract idea applies to **API changes** and **event/message schemas**: add the new field, migrate consumers, then remove the old one.

---

## Containers and orchestration

### Docker: the problem it solves
"It works on my machine." Apps depend on specific runtime versions, OS libraries, and configuration; differences between laptops, CI, and servers cause bugs and painful setup.

**A container image** packages the app **with everything it needs** (runtime, libraries, files) into a single immutable artifact. A **container** is a running instance of an image: an isolated process sharing the host's OS kernel (much lighter and faster to start than a virtual machine, which runs a whole guest OS).

```dockerfile
# Multi-stage build: build tools stay out of the final image
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci                       # cached layer unless package files change
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
USER node                        # don't run as root
EXPOSE 3000
CMD ["node", "dist/server.js"]
```
**Concepts to know:** image vs container; **layers** and layer caching (order Dockerfile steps from least to most frequently changing); **multi-stage builds** (small, secure images); **registries** (Docker Hub, Amazon ECR, Google Artifact Registry, GitHub Container Registry); **tags** (use immutable tags like the git SHA, not just `latest`); **`docker compose`** for running multi-container setups locally (app + Postgres + Redis); `.dockerignore`.

### Kubernetes: the problem it solves
Running containers on **many machines** raises questions: which machine runs which container? What restarts a crashed container? How do containers find each other? How do you roll out a new version or scale up under load? **Kubernetes (K8s)** is the standard **container orchestrator** that answers these. You **declare the desired state** in YAML ("run 3 replicas of this image, with these resources"), and Kubernetes continuously works to make reality match it ("reconciliation").

| Concept | What it is | Why it exists |
|---|---|---|
| **Cluster / Node** | A set of machines (nodes) managed by a control plane | Pool of compute |
| **Pod** | The smallest deployable unit: one or more tightly coupled containers sharing a network namespace and storage | Usually one app container (+ optional sidecars) |
| **Deployment** | Manages a set of identical, **stateless** pods: replica count, rolling updates, rollbacks | Self-healing + updates |
| **ReplicaSet** | Keeps N copies of a pod running (managed by a Deployment) | |
| **Service** | A **stable virtual IP and DNS name** that load-balances to the matching pods (pods come and go; the Service doesn't) | Service discovery |
| **Ingress / Gateway API** | HTTP(S) routing from outside the cluster to Services (host/path rules, TLS) | Entry point |
| **ConfigMap / Secret** | Configuration and sensitive values injected as env vars or files | Separate config from images |
| **StatefulSet** | Pods with **stable identities and persistent storage** | Databases, Kafka, ZooKeeper |
| **DaemonSet** | One pod on every node | Log collectors, monitoring agents |
| **Job / CronJob** | Run-to-completion and scheduled tasks | Batch work, migrations |
| **HPA** (Horizontal Pod Autoscaler) | Scales replicas based on CPU, memory, or custom metrics | Elasticity |
| **Namespace** | Logical isolation within a cluster | Teams, environments |
| **Probes** | Liveness, readiness, and startup checks | See [[Reliability & Observability#Health checks]] |
| **Resource requests/limits** | CPU/memory reserved and capped per container | Scheduling and isolation |
| **Helm / Kustomize** | Packaging and templating of manifests | Reuse across environments |

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders-api }
spec:
  replicas: 3
  selector: { matchLabels: { app: orders-api } }
  strategy: { type: RollingUpdate, rollingUpdate: { maxUnavailable: 0, maxSurge: 1 } }
  template:
    metadata: { labels: { app: orders-api } }
    spec:
      containers:
        - name: api
          image: registry.example.com/orders-api:3f9a2c1      # immutable git-SHA tag
          ports: [{ containerPort: 3000 }]
          envFrom: [{ secretRef: { name: orders-secrets } }]
          resources: { requests: { cpu: "250m", memory: "256Mi" }, limits: { memory: "512Mi" } }
          readinessProbe: { httpGet: { path: /readyz, port: 3000 }, periodSeconds: 5 }
          livenessProbe:  { httpGet: { path: /healthz, port: 3000 }, periodSeconds: 10 }
---
apiVersion: v1
kind: Service
metadata: { name: orders-api }
spec:
  selector: { app: orders-api }
  ports: [{ port: 80, targetPort: 3000 }]
```
Managed Kubernetes: **EKS** (AWS), **GKE** (Google), **AKS** (Azure). Simpler container platforms that hide Kubernetes: AWS ECS/Fargate, Google Cloud Run, Azure Container Apps, Fly.io, Render, Railway. (Google runs **Borg** internally, Kubernetes' ancestor; big companies often have their own internal platforms, but the concepts transfer.)

---

## Infrastructure as Code (IaC)

### The problem
Infrastructure created by clicking through cloud consoles is **undocumented, unreviewable, and unreproducible**. Nobody knows exactly how production is configured, staging drifts from production, and recreating an environment after a disaster is guesswork.

### The practice
Define infrastructure (networks, databases, load balancers, IAM permissions, DNS, Kubernetes clusters) in **version-controlled code**, review it like application code, and apply it with tools that compute and execute the necessary changes.
```hcl
# Terraform: an S3 bucket for user uploads with versioning and blocked public access
resource "aws_s3_bucket" "uploads" {
  bucket = "keycapalley-user-uploads"
}
resource "aws_s3_bucket_versioning" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  versioning_configuration { status = "Enabled" }
}
resource "aws_s3_bucket_public_access_block" "uploads" {
  bucket                  = aws_s3_bucket.uploads.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```
- **Declarative tools**: **Terraform** (multi-cloud; **OpenTofu** is its open-source fork), AWS CloudFormation, Azure Bicep. **Programming-language tools**: AWS CDK, **Pulumi**.
- **Workflow:** `terraform plan` shows exactly what will change (reviewed in the PR), then `terraform apply` executes it. State files track what exists.
- **Benefits:** reproducible environments, code review and history for infrastructure, drift detection, disaster recovery, consistency across dev/staging/prod.
- **GitOps**: the Git repository is the **single source of truth** for what should be deployed; an agent (**Argo CD**, Flux) continuously syncs the cluster to match the repo. Deploying = merging a PR; rollback = reverting a commit.

---

## Environments and configuration

Typical progression: **local → dev → staging (production-like) → production** (sometimes with QA, preview environments per PR, or a pre-production canary stage).
- Keep **staging as similar to production as practical** (same infrastructure code, similar data shapes), but understand it's never identical, which is why canaries and monitoring in production matter.
- **Preview environments** (an ephemeral deployment per PR, as Vercel and Netlify do for frontends) make review much easier.
- **Configuration via environment variables** (or a config service), never hardcoded per environment; **secrets** from a secrets manager ([[Security Fundamentals#Secrets management]]).

### The Twelve-Factor App
A widely cited set of principles (from Heroku, 2011) for building cloud-friendly services:
1. **Codebase**: one codebase in version control, many deploys.
2. **Dependencies**: explicitly declared and isolated (`package.json` + lockfile).
3. **Config**: stored in the **environment**, not in code.
4. **Backing services** (databases, queues, caches) are attached resources, swappable via config.
5. **Build, release, run**: strictly separate stages.
6. **Processes**: run as **stateless** processes; persist state in backing services.
7. **Port binding**: the service is self-contained and exports HTTP on a port.
8. **Concurrency**: scale out by running more processes.
9. **Disposability**: fast startup and **graceful shutdown**.
10. **Dev/prod parity**: keep environments as similar as possible.
11. **Logs**: treat logs as **event streams** written to stdout and collected by the platform.
12. **Admin processes**: run one-off tasks (migrations, scripts) as one-off processes in the same environment.

---

## Cloud services map

The names differ by provider, but the building blocks are the same. AWS is the most common in job postings.

| Need | AWS | Google Cloud | Azure |
|---|---|---|---|
| Virtual machines | EC2 | Compute Engine | Virtual Machines |
| Managed Kubernetes | EKS | GKE | AKS |
| Simpler containers | ECS / Fargate, App Runner | Cloud Run | Container Apps |
| Serverless functions | Lambda | Cloud Run functions | Functions |
| Object storage | S3 | Cloud Storage | Blob Storage |
| Relational DB | RDS (Postgres/MySQL), Aurora | Cloud SQL, AlloyDB, Spanner | Azure SQL, Azure Database for PostgreSQL |
| NoSQL | DynamoDB, DocumentDB | Firestore, Bigtable | Cosmos DB |
| Cache | ElastiCache (Redis/Valkey/Memcached) | Memorystore | Azure Cache for Redis |
| Queues / pub-sub | SQS, SNS, EventBridge | Pub/Sub, Cloud Tasks | Service Bus, Event Grid |
| Streaming | Kinesis, MSK (Kafka) | Pub/Sub, Dataflow | Event Hubs |
| CDN | CloudFront | Cloud CDN | Front Door |
| DNS | Route 53 | Cloud DNS | Azure DNS |
| Load balancing | ALB (L7), NLB (L4) | Cloud Load Balancing | Application Gateway, Load Balancer |
| API gateway | API Gateway | API Gateway, Apigee | API Management |
| Data warehouse | Redshift, Athena | BigQuery | Synapse, Fabric |
| Secrets / keys | Secrets Manager, KMS | Secret Manager, Cloud KMS | Key Vault |
| Monitoring / tracing | CloudWatch, X-Ray | Cloud Monitoring, Cloud Trace | Azure Monitor, Application Insights |
| Identity & access | IAM, Cognito (end users) | IAM, Identity Platform | Entra ID |
| Workflows | Step Functions | Workflows | Logic Apps, Durable Functions |

### Regions and availability zones
- A **region** is a geographic area (e.g., `us-west-2` in Oregon) containing several **availability zones (AZs)**: physically separate datacenters with independent power and networking, connected by fast links.
- **Deploy across at least 2–3 AZs** for high availability (an AZ outage shouldn't take you down); most managed services offer multi-AZ options.
- Use **multiple regions** for disaster recovery and lower latency for global users, at significantly higher cost and complexity (cross-region data replication and consistency).

---

## Quick self-test
- [ ] Trunk-based development vs GitFlow: when would you use each? What makes trunk-based development safe?
- [ ] Continuous delivery vs continuous deployment.
- [ ] Why "build once, deploy everywhere"?
- [ ] What are the four DORA metrics, and what does the research say about speed vs stability?
- [ ] Rolling vs blue-green vs canary: tradeoffs, and when you'd choose each.
- [ ] How do feature flags separate deploy from release? What is flag debt?
- [ ] Walk through renaming a database column with zero downtime. Why can't you do it in one step?
- [ ] Image vs container; container vs VM. Why use multi-stage Docker builds?
- [ ] Pod vs Deployment vs Service vs Ingress in Kubernetes.
- [ ] What problem does infrastructure as code solve? What is GitOps?
- [ ] What's the difference between a region and an availability zone, and how many AZs should a highly available service use?

Back to [[Design Index]]
