# Pardis Cloud — Phase 1 E-Commerce Deployment Plan

Team: **Ali, Alireza, Arashk, Davood** (DevOps)
Project: Phase 1 — Single VPC architecture for an online e-commerce application on **Pardis Cloud**.

Traffic path being built:

```
Users → DNS → Anti-DDoS → CFW → WAF → ELB → Ingress → Frontend / Backend API / Order Service (CCE)
                                                        → RDS MySQL / DCS Redis / OBS
```

## Documents in this plan

| File | What it contains |
|---|---|
| [`00-prerequisites.md`](00-prerequisites.md) | Everything that must exist **before** anyone touches the console |
| [`01-roadmap.md`](01-roadmap.md) | 6-week roadmap, milestones, dependency graph, weekly sync agenda |
| [`02-naming-and-tagging.md`](02-naming-and-tagging.md) | Exact resource names and mandatory tags for every resource in this project |
| [`tasks/alireza-network.md`](tasks/alireza-network.md) | **Alireza** — Network & cloud foundations (VPC, subnets, NAT/EIP, SGs, bastion, ELB, DNS, CFW, Anti-DDoS) |
| [`tasks/davood-application.md`](tasks/davood-application.md) | **Davood** — CCE & application delivery (Docker, SWR, CCE, manifests, Ingress, autoscaling, rollback) |
| [`tasks/arashk-data-storage.md`](tasks/arashk-data-storage.md) | **Arashk** — Data, storage & recovery (RDS, DCS, OBS, backups, CBR, restore testing) |
| [`tasks/ali-security-observability.md`](tasks/ali-security-observability.md) | **Ali** — Security, observability & acceptance (IAM/MFA, WAF, HSS, Cloud Eye, LTS, CTS, SMN, testing) |

## Ownership summary (from `tasks_team.ods`)

| Person | Owns | Pairs with |
|---|---|---|
| **Alireza** | The environment and its network connections | Davood on ELB/Ingress, Ali on network security |
| **Davood** | Getting the application built and running | Arashk on application/data integration |
| **Arashk** | Application data and recoverability | Davood on app behavior, Ali on recovery evidence |
| **Ali** | Access control, operational visibility, validation | Everyone (each owner helps Ali validate their telemetry) |

## Assumptions & notes

1. **Environment**: examples in this plan use `env = prod`. If you build a lab first (recommended), do the identical build with `env = lab` — the naming convention requires strict environment separation.
2. **Business unit**: this plan uses `bu = sre` (allowed values in the convention: `sa, tm, inf, bd, hr, ai, sre`). Change once, consistently, if your BU code differs.
3. **Application code**: `app = shop`.
4. **Backup service naming discrepancy**: the project document says "CBS (Cloud Backup Service)" but the official Pardis service catalog (`Services.xlsx`) lists **CBR (Cloud Backup and Recovery)** — same service, AWS Backup equivalent. This plan uses **CBR**.
5. The project document mentions "Huawei Cloud" in a few places — Pardis Cloud consoles mirror Huawei Cloud consoles, so Huawei Cloud documentation is a useful secondary reference when a console screen is unclear.
6. Pardis ↔ AWS service mapping (for anyone coming from AWS): ECS=EC2, CCE=EKS, SWR=ECR, OBS=S3, EVS=EBS, CBR=AWS Backup, EIP=Elastic IP, Cloud Eye (CES)=CloudWatch, LTS=CloudWatch Logs, CTS=CloudTrail-style auditing, SMN=SNS, DCS=ElastiCache, DNS=Route53.
