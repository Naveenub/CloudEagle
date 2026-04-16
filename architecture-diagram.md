# Architecture Diagram – `sync-service` on GCP

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            INTERNET                                      │
└──────────────────────────┬───────────────────────────────────────────────┘
                           │ HTTPS :443
                           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Cloud Armor (WAF + DDoS Protection)                                     │
│  • OWASP Top 10 rule sets                                                │
│  • Rate limiting per IP                                                  │
└──────────────────────────┬───────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  GCP HTTPS Load Balancer                                                 │
│  • Managed SSL certificate (Let's Encrypt via GCP)                       │
│  • Global Anycast IP                                                     │
│  • Health checks on /actuator/health                                     │
└──────────────────────────┬───────────────────────────────────────────────┘
                           │ HTTP :8080 (internal)
                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  GCP VPC: cloudeagle-vpc (10.0.0.0/16)                                              │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │  GKE Autopilot Cluster: cloudeagle-prod-cluster                              │   │
│  │  Region: us-central1                                                         │   │
│  │                                                                              │   │
│  │  Namespace: prod                                                             │   │
│  │  ┌──────────────────────────────────────────────────────────────────────┐    │   │
│  │  │  Deployment: sync-service-blue  (active)                             │    │   │
│  │  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐          │    │   │
│  │  │  │   Pod (blue)   │  │   Pod (blue)   │  │   Pod (blue)   │          │    │   │
│  │  │  │ sync-service   │  │ sync-service   │  │ sync-service   │          │    │   │
│  │  │  │ :8080          │  │ :8080          │  │ :8080          │          │    │   │
│  │  │  └────────────────┘  └────────────────┘  └────────────────┘          │    │   │
│  │  │                  replicas: 2–10 (HPA)                                │    │   │
│  │  └──────────────────────────────────────────────────────────────────────┘    │   │
│  │                                                                              │   │
│  │  ┌──────────────────────────────────────────────────────────────────────┐    │   │
│  │  │  Deployment: sync-service-green (standby / next deploy target)       │    │   │
│  │  └──────────────────────────────────────────────────────────────────────┘    │   │
│  │                                                                              │   │
│  │  Service: sync-service-active  (selector: color=blue ← flips on deploy)      │   │
│  │  HPA: min=2, max=10, CPU target=60%                                          │   │
│  │  ServiceAccount: sync-service-sa (Workload Identity bound)                   │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │  GCP Managed Services (same VPC / private access)                            │   │
│  │                                                                              │   │
│  │  ┌─────────────────────┐    ┌──────────────────────┐                         │   │
│  │  │  Secret Manager     │    │  Artifact Registry   │                         │   │
│  │  │  • mongodb-uri      │    │  • sync-service imgs │                         │   │
│  │  │  • jwt-secret       │    │  (per-commit tags)   │                         │   │
│  │  │  • api-keys         │    └──────────────────────┘                         │   │
│  │  └─────────────────────┘                                                     │   │
│  │                                                                              │   │
│  │  ┌─────────────────────┐    ┌──────────────────────┐                         │   │
│  │  │  Cloud Logging      │    │  Cloud Monitoring    │                         │   │
│  │  │  • App JSON logs    │    │  • RED metrics       │                         │   │
│  │  │  • Audit logs       │    │  • HPA metrics       │                         │   │
│  │  │  • Access logs      │    │  • Alerts → Slack    │                         │   │
│  │  └─────────────────────┘    └──────────────────────┘                         │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                           │ VPC Peering (private, port 27017)
                           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  MongoDB Atlas (GCP-hosted, us-central1)                                 │
│  • M30 cluster (3-node replica set)                                      │
│  • No public endpoint — VPC peer only                                    │
│  • Continuous backup, 30-day retention                                   │
└──────────────────────────────────────────────────────────────────────────┘


────────────────────────────────────────────────────────────────────────────
CI/CD FLOW (separate plane)
────────────────────────────────────────────────────────────────────────────

  Developer
     │  git push / PR
     ▼
  GitHub
     │  webhook
     ▼
  Jenkins (GCP VM, private subnet)
     │
     ├─── PR event ──► [Checkout → Build → Test → SonarQube] (no deploy)
     │
     └─── Merge event ──► [Checkout → Build → Test → SonarQube
                               → Docker Build → Push to Artifact Registry
                               → Deploy to GKE (rolling / blue-green)
                               → Smoke Tests]
                                    │
                                    └─── PROD only: Manual Approval Gate
                                         (Jenkins input step, Slack notification)
```

## Environment Layout

```
┌──────────────────────────────────────────────────────┐
│  GCP Project: cloudeagle-nonprod                     │
│  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │  GKE: qa-cluster │  │  GKE: staging-cluster    │  │
│  │  ns: qa          │  │  ns: staging             │  │
│  │  min pods: 1     │  │  min pods: 2             │  │
│  └──────────────────┘  └──────────────────────────┘  │
│  MongoDB Atlas: M10 (qa) + M10 (staging)             │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  GCP Project: cloudeagle-prod                        │
│  ┌──────────────────────────────────────────────┐    │
│  │  GKE Autopilot: prod-cluster                 │    │
│  │  ns: prod  |  Blue/Green deploy              │    │
│  │  min pods: 2, max: 10                        │    │
│  └──────────────────────────────────────────────┘    │
│  MongoDB Atlas: M30 replica set                      │
└──────────────────────────────────────────────────────┘
```

## Key Architecture Decisions Summary

| Decision | Choice | Reason |
|----------|--------|--------|
| Compute | GKE Autopilot | No node ops, pay-per-pod, startup-friendly |
| Deployment | Blue/Green | Zero-downtime + instant rollback |
| Database | MongoDB Atlas (VPC peer) | Managed reliability, private network only |
| Secrets | GCP Secret Manager + Workload Identity | No static creds anywhere |
| Ingress | Cloud Armor → HTTPS LB → nginx ingress | Security + managed SSL |
| Observability | Cloud Logging + Cloud Monitoring | GCP-native, zero setup overhead |
| CI/CD | Jenkins + GitHub webhooks | Matches existing stack |
| Environments | Separate GCP projects (nonprod / prod) | Blast radius containment |
