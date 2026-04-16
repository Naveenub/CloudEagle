# Part 2 – Infrastructure Design for `sync-service` on GCP

## Overview

The infrastructure targets a startup context: **secure by default, horizontally scalable, and cost-conscious**. Every choice below is justified against those three constraints.

---

## 1. Compute Choice — GKE Autopilot

### Decision: GKE Autopilot ✅

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **GKE Autopilot** | No node management, pay-per-pod, built-in security hardening | Slightly less control over node config | ✅ Chosen |
| GKE Standard | Full control, cheaper at scale | Node management overhead, not startup-friendly | ❌ Over-engineered |
| Compute Engine (VMs) | Simplest to reason about | Manual scaling, no self-healing, high ops burden | ❌ Current pain point — moving away |
| Cloud Run | Zero infra, pay-per-request | Cold starts hurt long-running sync jobs, limited MongoDB connection pooling | ❌ Doesn't fit sync-service workload |

**Why Autopilot:** sync-service is a long-running Spring Boot service with persistent MongoDB connections. Cloud Run's ephemeral nature causes connection pool churn. GKE Autopilot gives Kubernetes semantics (deployments, services, HPA) without the toil of managing node pools — ideal for a small DevOps team.

### Auto-Scaling Configuration

```yaml
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sync-service-hpa
  namespace: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sync-service
  minReplicas: 2       # Always 2 for HA
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
```

---

## 2. MongoDB Hosting — MongoDB Atlas (GCP-hosted)

### Decision: MongoDB Atlas on GCP ✅

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **MongoDB Atlas (GCP)** | Managed, VPC peering, automated backups, free tier available | Monthly cost (M10+ for prod) | ✅ Chosen |
| Self-hosted on GKE | Full control | Ops-heavy: replication, backups, upgrades all manual | ❌ Too much ops burden |
| Firestore / Bigtable | Fully managed GCP-native | Not MongoDB — requires app rewrite | ❌ Not applicable |

**Why Atlas:** Atlas offers VPC peering directly into our GCP VPC, keeping traffic private. Automated backups, point-in-time restore, and auto-scaling storage are included. For a startup, paying Atlas to manage MongoDB reliability is far cheaper than the engineer-hours to do it manually.

- **Cluster tier:** M10 (qa/staging) → M30 (prod, with replica set)
- **Connection:** Private VPC peering — no public endpoint exposed
- **Backups:** Continuous cloud backup with 7-day retention (qa/staging), 30-day (prod)

---

## 3. Networking — VPC, Ingress, and Security

### VPC Design

```
GCP Project: cloudeagle-prod
│
└── VPC: cloudeagle-vpc (10.0.0.0/16)
    │
    ├── Subnet: gke-nodes        (10.0.1.0/24)  — GKE Autopilot pods
    ├── Subnet: gke-services     (10.0.2.0/24)  — GKE services
    ├── Subnet: reserved-atlas   (10.0.3.0/24)  — MongoDB Atlas VPC peer
    └── Subnet: bastion          (10.0.4.0/28)  — Jump host (SSH access only)
```

### Ingress Path

```
Internet
    │
    ▼
Cloud Armor (WAF + DDoS)
    │
    ▼
GCP HTTPS Load Balancer  (SSL termination, managed cert)
    │
    ▼
GKE Ingress (nginx ingress controller)
    │
    ▼
sync-service Kubernetes Service (ClusterIP)
    │
    ▼
sync-service Pods
    │
    ▼
MongoDB Atlas (private VPC peer, port 27017)
```

### Firewall Rules

| Rule | Source | Destination | Port | Action |
|------|--------|-------------|------|--------|
| Allow HTTPS ingress | 0.0.0.0/0 | Load Balancer | 443 | Allow |
| Allow LB → GKE | LB IP range | gke-nodes subnet | 8080 | Allow |
| Allow GKE → Atlas | gke-nodes subnet | atlas-peer subnet | 27017 | Allow |
| Allow bastion SSH | VPN/IAP only | bastion subnet | 22 | Allow |
| Deny all other ingress | * | * | * | Deny |

---

## 4. Secrets & IAM

### Secrets Management

All secrets are stored in **GCP Secret Manager** and accessed at runtime via **Workload Identity** — no static credentials on nodes or in environment files.

```
Secret Manager Secrets
├── sync-service/mongodb-uri         (per environment version)
├── sync-service/jwt-secret
├── sync-service/external-api-key
└── jenkins/gcp-sa-key               (for CI/CD only)
```

### IAM Roles (Least Privilege)

| Principal | Role | Scope |
|-----------|------|-------|
| GKE Workload SA (`sync-service-sa`) | `roles/secretmanager.secretAccessor` | sync-service secrets only |
| GKE Workload SA | `roles/monitoring.metricWriter` | Project |
| GKE Workload SA | `roles/logging.logWriter` | Project |
| Jenkins SA | `roles/container.developer` | GKE clusters |
| Jenkins SA | `roles/artifactregistry.writer` | Artifact Registry repo |
| Developers | `roles/container.viewer` | GKE (read-only) |
| Developers | `roles/logging.viewer` | Logs Explorer |

### Workload Identity Setup (Kubernetes side)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sync-service-sa
  namespace: prod
  annotations:
    iam.gke.io/gcp-service-account: sync-service-sa@cloudeagle-prod.iam.gserviceaccount.com
```

---

## 5. Logging & Monitoring Stack

### Logging

| Layer | Tool | What's Captured |
|-------|------|-----------------|
| Application logs | **Cloud Logging** (via GKE auto-export) | Spring Boot structured JSON logs |
| Access logs | Cloud Logging (LB access logs) | HTTP request/response per request |
| Audit logs | Cloud Audit Logs | All GCP API calls (admin activity) |
| MongoDB logs | Atlas built-in + Atlas → Cloud Logging sink | Slow queries, connection events |

Spring Boot is configured to output **structured JSON** (`logstash-logback-encoder`) so Cloud Logging can parse and index fields like `traceId`, `userId`, `serviceName` natively.

### Monitoring & Alerting

| Signal | Tool | Alert Threshold |
|--------|------|-----------------|
| CPU / Memory | **Cloud Monitoring** (auto from GKE) | CPU > 80% for 5m |
| HTTP error rate | Cloud Monitoring + LB metrics | 5xx rate > 1% for 2m |
| Pod availability | GKE workload metrics | Available pods < minReplicas |
| MongoDB latency | Atlas built-in monitoring + alert | P99 > 200ms |
| Deployment health | Jenkins + Cloud Monitoring custom metric | Smoke test failure |

Alerts route to **PagerDuty** (prod) and **Slack #alerts** (qa/staging) via Cloud Monitoring notification channels.

### Dashboards

A **Cloud Monitoring Dashboard** is provisioned with:
- Request rate, error rate, latency (RED metrics)
- Pod count vs HPA target
- MongoDB connection pool utilisation
- JVM heap and GC metrics (via Spring Boot Actuator + Micrometer → Cloud Monitoring)

---

## 6. Cost Management (Startup Constraints)

| Measure | Estimated Saving |
|---------|-----------------|
| GKE Autopilot — pay only for running pods, not idle nodes | ~30–40% vs Standard with spare capacity |
| MongoDB Atlas M10 for non-prod, M30 for prod only | Avoids paying for prod-sized DB in every env |
| HPA min replicas = 1 in QA (2 in prod) | Halves QA compute spend |
| Committed Use Discounts (1-year) on sustained GKE usage | ~20–30% discount |
| Cloud Armor — standard tier (not managed protection) | Saves ~$3,000/month vs enterprise tier |

Estimated monthly cost for prod environment: **$400–$600/month** at typical sync-service load (before committed-use discounts).
