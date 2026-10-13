# 02 — Naming & Tagging Reference

Source: *Pardis Cloud Resource Naming Convention V0.1*. Pattern (mandatory, exact order):

```
pc-<bu>-<app>-<env>-<resource>-<role>-<seq>
```

Fixed values for this project: `bu = sre`, `app = shop`, `env = prod` (use `lab` for the rehearsal environment — never mix environments in one name).

## Mandatory rules (checked at every weekly sync)

1. **Lowercase only** — no uppercase anywhere.
2. **Hyphen** is the only separator — no `_`, `.`, or spaces.
3. Sequence is always **exactly 3 digits** (`001`, `002` …).
4. Every resource carries the **full mandatory tag set** (below).
5. Name must state application, environment, and role — no generic names (`test`, `server1`, `mybucket` are forbidden).
6. **No personal names** in resources (`ali-db` is forbidden).
7. prod and non-prod strictly separated.

## Mandatory tags (apply to every resource)

| Tag key | Value for this project |
|---|---|
| `Organization` | `pardis` |
| `BusinessUnit` | `sre` |
| `Application` | `shop` |
| `Environment` | `prod` (or `lab`) |
| `Role` | per-resource (see table) |
| `CostCenter` | optional, e.g. `cc-1024` |

## Canonical resource names for Phase 1

Copy names exactly from this table when creating resources. If you need a second instance of anything, increment the sequence (`002`).

### Network (Alireza)

| Resource | Name | Notes |
|---|---|---|
| VPC | `pc-sa-phase1-shop-prod-vpc-main-001` | 10.0.0.0/16 |
| Public subnet | `pc-sa-phase1-shop-prod-snet-public-001` | 10.0.1.0/24 — NAT GW, bastion |
| CCE subnet | `pc-sa-phase1-shop-prod-snet-app-001` | 10.0.2.0/24 — cluster nodes/pods |
| Database subnet | `pc-sa-phase1-shop-prod-snet-db-001` | 10.0.3.0/24 — RDS, DCS |
| Storage subnet | `pc-sa-phase1-shop-prod-snet-storage-001` | 10.0.4.0/24 |
| Route table (private) | `pc-sa-phase1-shop-prod-rtb-app-001` | default route → NAT GW |
| NAT Gateway | `pc-sa-phase1-shop-prod-nat-main-001` | |
| EIP for NAT | `pc-sa-phase1-shop-prod-eip-nat-001` | |
| EIP for ELB | `pc-sa-phase1-shop-prod-eip-elb-001` | |
| ELB | `pc-sa-phase1-shop-prod-elb-public-001` | listeners 80/443 |
| SG — ELB | `pc-sa-phase1-shop-prod-sg-elb-001` | 80,443 from 0.0.0.0/0 |
| SG — CCE nodes | `pc-sa-phase1-shop-prod-sg-app-001` | from ELB SG only |
| SG — RDS | `pc-sa-phase1-shop-prod-sg-db-001` | 3306 from CCE SG (+bastion) |
| SG — DCS | `pc-sa-phase1-shop-prod-sg-cache-001` | 6379 from CCE SG (+bastion) |
| SG — Bastion | `pc-sa-phase1-shop-prod-sg-bastion-001` | 2222, 443 from office IPs only |
| DNS public zone | `pc-sa-phase1-shop-prod-dns-main-001` | zone for the shop domain |

### Compute & containers (Alireza / Davood)

| Resource | Name | Notes |
|---|---|---|
| Bastion ECS | `pc-sa-phase1-shop-prod-ecs-bastion-001` | public subnet, key auth only |
| CCE cluster | `pc-sa-phase1-shop-prod-cce-main-001` | 3+ nodes |
| CCE node pool | `pc-sa-phase1-shop-prod-nodepool-app-001` | |
| SWR organization | `pc-sa-phase1-shop-prod` | SWR orgs don't take the full pattern; keep the prefix |
| Images | `pc-sa-phase1-shop-prod/frontend`, `.../backend-api`, `.../order-service` | tag = git SHA or semver, never only `latest` |

### Data & storage (Arashk)

| Resource | Name | Notes |
|---|---|---|
| RDS MySQL | `pc-sa-phase1-shop-prod-rds-mysql-001` | MySQL 8.0+, db subnet |
| DCS Redis | `pc-sa-phase1-shop-prod-dcs-redis-001` | Redis 5.0+, db subnet |
| OBS images bucket | `pc-sa-phase1-shop-prod-obs-images-001` | public-read or signed URLs |
| OBS invoices bucket | `pc-sa-phase1-shop-prod-obs-invoices-001` | private, encrypted |
| OBS backups bucket | `pc-sa-phase1-shop-prod-obs-backup-001` | private, versioned |
| OBS logs/audit bucket | `pc-sa-phase1-shop-prod-obs-audit-001` | CTS trace destination |
| CBR vault | `pc-sa-phase1-shop-prod-cbr-backup-001` | bastion EVS disks |
| Backup policy | `pc-sa-phase1-shop-prod-policy-backup-001` | daily 02:00, keep 30 days |

### Security & observability (Ali / Alireza)

| Resource | Name | Notes |
|---|---|---|
| CFW instance | `pc-sa-phase1-shop-prod-cfw-main-001` | VPC boundary, IPS on |
| WAF instance/policy | `pc-sa-phase1-shop-prod-waf-web-001` | OWASP Top 10 rules |
| LTS group — app | `pc-sa-phase1-shop-prod-lts-app-001` | CCE pod logs |
| LTS group — security | `pc-sa-phase1-shop-prod-lts-audit-001` | CFW/WAF/HSS/flow logs |
| CTS tracker | `pc-sa-phase1-shop-prod-cts-audit-001` | → obs-audit bucket, 180d |
| SMN — critical alerts | `pc-sa-phase1-shop-prod-smn-alert-001` | email + SMS |
| SMN — ops | `pc-sa-phase1-shop-prod-smn-ops-001` | email |
| SMN — security | `pc-sa-phase1-shop-prod-smn-security-001` | email + SMS |
| Cloud Eye dashboard | `pc-sa-phase1-shop-prod-dashboard-main-001` | ops overview |
| VPC flow log | `pc-sa-phase1-shop-prod-flowlog-main-001` | all subnets → LTS |

### Kubernetes-internal names (Davood)

Kubernetes objects live inside the cluster and are not Pardis Cloud resources, but keep them consistent:

- Namespaces: `shop-prod` (and `shop-lab`).
- Deployments/Services: `frontend`, `backend-api`, `order-service`.
- Ingress: `shop-ingress`. ConfigMap: `shop-config`. Secrets: `shop-db-credentials`, `shop-redis-credentials`, `shop-obs-credentials`.
