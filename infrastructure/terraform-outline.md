# Terraform Infrastructure Outline

> This document outlines the IaC structure. Full Terraform code would be added in a real engagement — this shows resource planning and module structure.

## Module Structure

```
infrastructure/terraform/
├── environments/
│   ├── qa/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── gke-cluster/       # GKE Autopilot cluster + node config
│   ├── networking/        # VPC, subnets, firewall rules
│   ├── secrets/           # Secret Manager secrets + IAM bindings
│   ├── monitoring/        # Alert policies, dashboards, notification channels
│   └── artifact-registry/ # Docker image repository
└── shared/
    ├── atlas-peering.tf   # MongoDB Atlas VPC peering
    └── jenkins-sa.tf      # Jenkins service account + roles
```

## Key Resources per Environment

### networking module
- `google_compute_network` — VPC
- `google_compute_subnetwork` — gke-nodes, gke-services, atlas-peer
- `google_compute_firewall` — ingress/egress rules per subnet
- `google_compute_global_address` — static IP for load balancer

### gke-cluster module
- `google_container_cluster` — Autopilot mode, private cluster
- `google_service_account` — sync-service-sa (Workload Identity)
- `google_project_iam_member` — Secret Manager + Logging + Monitoring roles
- `google_service_account_iam_binding` — Workload Identity binding

### secrets module
- `google_secret_manager_secret` — mongodb-uri, jwt-secret, api-keys
- `google_secret_manager_secret_version` — initial versions (populated from CI)
- `google_secret_manager_secret_iam_member` — bind sync-service-sa to each secret

### monitoring module
- `google_monitoring_alert_policy` — CPU, error rate, pod count alerts
- `google_monitoring_notification_channel` — Slack webhook, PagerDuty
- `google_monitoring_dashboard` — RED metrics dashboard

## State Management

- Remote state in **GCS bucket** (`cloudeagle-terraform-state`)
- State locking via **GCS native locking** (no DynamoDB needed on GCP)
- Separate state files per environment (`qa/terraform.tfstate`, etc.)

## Deployment Flow

```bash
# Plan (run in CI on PR)
terraform plan -var-file=terraform.tfvars -out=tfplan

# Apply (run on merge to infra branch, manual approval)
terraform apply tfplan
```
