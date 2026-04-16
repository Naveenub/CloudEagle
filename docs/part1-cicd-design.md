# Part 1 – CI/CD Pipeline Design for `sync-service`

## 1. Branching Strategy

### Branch → Environment Mapping

```
feature/*  ──► (no deployment — CI only)
     │
     └──► PR to develop
               │
           develop  ──► QA environment  (auto-deploy on merge)
               │
               └──► PR to staging
                         │
                     staging  ──► Staging environment  (auto-deploy on merge)
                         │
                         └──► PR to main  (requires 2 approvals + passing staging tests)
                                   │
                                   main  ──► Production  (manual approval gate)
```

### Rules

| Branch | Protection | Who Can Merge | Auto-Deploy |
|--------|-----------|---------------|-------------|
| `feature/*` | None | Author | ❌ |
| `develop` | PR required, 1 review | Any engineer | ✅ → QA |
| `staging` | PR required, 1 review + QA green | Senior engineer | ✅ → Staging |
| `main` | PR required, 2 reviews + staging green | Team lead | ⚠️ Manual gate → Prod |

### Preventing Accidental Prod Deployments

- `main` branch is **protected** in GitHub — direct pushes are blocked for everyone including admins.
- Jenkins pipeline for `main` has an `input` step requiring a named approver to click **"Deploy to Production"** — the pipeline parks itself and waits.
- A separate `PROD_DEPLOY_APPROVERS` Jenkins credential (a comma-separated list of GitHub usernames) is validated before the `input` step is surfaced.
- Deployments to prod are only triggered from Jenkins jobs scoped to the `main` branch — no other branch can reach the prod deploy stage.

---

## 2. Jenkins Pipeline

### High-Level Stages

```
Checkout → Build & Unit Test → Static Analysis → Docker Build
    → Push to Artifact Registry → Deploy to Environment
    → Smoke Test → [Manual Gate if prod] → Mark Stable / Rollback
```

### PR vs Merge Behaviour

| Event | Stages Run | Deploys? |
|-------|-----------|----------|
| PR opened / updated | Checkout, Build, Unit Test, Static Analysis | ❌ |
| Merge to `develop` | All stages | ✅ QA |
| Merge to `staging` | All stages | ✅ Staging |
| Merge to `main` | All stages + Manual Approval gate | ✅ Prod (after approval) |

### Rollback Strategy

1. **Automated rollback** — if the post-deploy smoke test fails, the pipeline immediately re-deploys the previously tagged Docker image (`stable-<env>` tag stored in Jenkins credentials store).
2. **Manual rollback** — a dedicated `rollback` Jenkins job accepts an image tag as a parameter and redeploys it to the chosen environment.
3. **Blue/Green natural rollback** — since the old (blue) environment is kept alive for 10 minutes post-cutover, traffic can be switched back at the load balancer with zero downtime if monitoring alerts fire.

---

## 3. Configuration Management

### Env-Specific Configs

- Application config is managed via **Spring Boot profiles** (`application-qa.yml`, `application-staging.yml`, `application-prod.yml`).
- Profile-specific YAML files are stored in a **dedicated private Git repo** (`sync-service-config`) and pulled during the deploy stage — not baked into the Docker image.
- The config repo is checked out using a Jenkins deploy key (SSH credential stored in Jenkins Credentials Manager).

### Secrets Handling

| Secret | Storage | How It Reaches the App |
|--------|---------|------------------------|
| MongoDB URI (with creds) | GCP Secret Manager | Mounted as env var via Workload Identity |
| External API keys | GCP Secret Manager | Mounted as env var via Workload Identity |
| Jenkins deploy keys | Jenkins Credentials (SSH) | Used only at pipeline runtime |
| Docker registry creds | Jenkins Credentials (Username/Password) | Used only at pipeline runtime |

- **No secrets in Git** — `.gitignore` blocks `*.env`, `*secret*`, and `application-prod.yml` at the source.
- GCP **Workload Identity Federation** links the GKE service account to the Secret Manager secrets — no static credentials are stored on the nodes.
- Secrets are **versioned** in Secret Manager; rotation triggers a new deployment automatically via a Cloud Function → Pub/Sub → Jenkins webhook chain.

---

## 4. Deployment Strategy

### Chosen Strategy: Blue/Green

**Justification:**

| Strategy | Downtime | Rollback Speed | Resource Cost | Why Not? |
|----------|----------|---------------|---------------|----------|
| Recreate | Yes | Medium | Low | Unacceptable downtime |
| Rolling | Minimal | Slow (gradual) | Low | Mixed versions in prod = harder debugging |
| **Blue/Green** | **Zero** | **Instant** | Medium | ✅ Best for startup SLA + fast rollback |

CloudEagle serves customers in real-time; even a 30-second outage during a deploy can impact renewal workflows. Blue/Green gives instant traffic cutover and instant rollback by just flipping the load balancer target.

### Zero-Downtime Approach

1. **Blue** = currently live environment.
2. Jenkins deploys the new image to the **Green** environment (separate GKE deployment / node pool).
3. Smoke tests run against Green via an internal test endpoint.
4. If smoke tests pass → update the GCP HTTP(S) Load Balancer backend service to point to Green.
5. Blue stays warm for **10 minutes** (configurable).
6. If monitoring alerts fire within 10 minutes → flip load balancer back to Blue (< 5 second cutover).
7. After 10 minutes of clean metrics → Blue is decommissioned (or becomes the next Green).
