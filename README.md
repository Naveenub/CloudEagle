# CloudEagle – DevOps Assignment

**Candidate Submission | Spring Boot `sync-service` on GCP**

---

## Repository Structure

```
CloudEagle/
├── README.md                        # This file
├── docs/
│   ├── part1-cicd-design.md         # Part 1: CI/CD Design Document
│   └── part2-infrastructure.md      # Part 2: Infrastructure Design Document
├── jenkins/
│   └── Jenkinsfile                  # Declarative Jenkins Pipeline
└── infrastructure/
    ├── architecture-diagram.md      # ASCII + described architecture diagram
    └── terraform-outline.md         # IaC outline (structure & key resources)
```

---

## Quick Summary

| Part | Topic | Key Choices |
|------|-------|-------------|
| 1 | CI/CD | GitFlow branching · Jenkins declarative pipeline · Blue/Green deploy |
| 2 | Infrastructure | GKE Autopilot · MongoDB Atlas · Cloud Armor · Cloud Operations Suite |

---

## How to Navigate

- **Part 1 design doc** → [`docs/part1-cicd-design.md`](docs/part1-cicd-design.md)
- **Jenkinsfile** → [`jenkins/Jenkinsfile`](jenkins/Jenkinsfile)
- **Part 2 design doc** → [`docs/part2-infrastructure.md`](docs/part2-infrastructure.md)
- **Architecture diagram** → [`infrastructure/architecture-diagram.md`](infrastructure/architecture-diagram.md)
