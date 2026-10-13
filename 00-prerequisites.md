# 00 — Prerequisites

Complete everything on this page **before Week 1**. Owner for the whole page: **Ali** (IAM/access) with input from everyone.

## 1. Account & access

- [x] Pardis Cloud tenant/account created and billing enabled (quota/credit confirmed).
- [x] Root account secured: strong password, **MFA enabled**, credentials stored in the team password manager. Root is **never** used for daily work.
- [x] Four IAM users created — one per person, no shared accounts (see Ali's task file for exact steps):
  - `ali` → admin + security roles
  - `alireza` → network roles
  - `davood` → CCE/SWR/developer roles
  - `arashk` → RDS/DCS/OBS/CBR roles
- [x] MFA enabled for all four users.
- [x] IAM groups created: `admins`, `network-ops`, `app-dev`, `data-ops`, `readonly`.
- [x] Everyone can log in to the Pardis Cloud console and sees only their permitted services.



## 2. Quotas — verify (and request increases early, they take time)


| Resource          | Minimum needed                    |
| ----------------- | --------------------------------- |
| VPCs              | 1                                 |
| Subnets           | 4                                 |
| ECS instances     | 1 (bastion) + CCE node flavors ×3 |
| EIPs              | 2 (NAT Gateway, ELB)              |
| CCE clusters      | 1 (3+ worker nodes)               |
| RDS instances     | 1 (MySQL 8.0+)                    |
| DCS instances     | 1 (Redis 5.0+)                    |
| OBS buckets       | 3–4                               |
| ELB instances     | 1                                 |
| CBR vault storage | ≥ 2× bastion disk size            |




## 3. Domain & certificates

- [ ] Domain name for the shop registered and transferable to Pardis Cloud DNS (or NS records can be delegated).
- [ ] SSL/TLS certificate obtained (public CA) for the domain — needed by WAF and the ELB HTTPS listener. Have the cert + private key + chain ready to upload.



## 4. Application readiness (Davood + dev team)

- [x] Source code for the three services (frontend, backend API, order service) accessible to the team — `Bookland/` repo (`frontend/`, `backend/`, `order-service/`).
- [x] Each service has a working `Dockerfile`; images build and run locally — `docker compose up --build -d` then `Bookland/scripts/smoke-test.sh`.
- [x] Configuration externalized: DB/Redis/OBS endpoints and credentials come from env vars — contract in `Bookland/.env.example` and `Bookland/docs/application-inventory.md`.
- [x] Each service exposes **health endpoints** — `/healthz` + `/readyz` on all three; backend-api also at `/api/healthz` + `/api/readyz` (readiness checks MySQL `SELECT 1` + Redis `PING`).
- [x] Database schema + migration scripts ready (users, product catalog + inventory in `shop`; orders, order items, payments in `shop_orders`) — Django migrations; SQL export via `Bookland/scripts/export-sql-migrations.sh` into `Bookland/db/migrations/`.



## 5. Workstations (everyone)

- [ ] `kubectl` installed (matching the CCE Kubernetes version).
- [ ] Docker installed (Davood at minimum).
- [ ] **KooCLI** (Pardis Cloud CLI — AWS CLI equivalent) installed and configured with your personal IAM access key: `hcloud configure` style setup. Rotate keys every 90 days.
- [ ] SSH keypairs generated per person (`ed25519`). Public keys collected for the bastion. **Password SSH login will be disabled.**
- [ ] Corporate/office public IP ranges collected — needed for bastion and CFW allow rules.



## 6. Agreements to lock in at the kickoff meeting

- [ ] Naming values fixed: `bu = sre`, `app = shop`, `env = prod` (and `lab` if a rehearsal environment is built first). See `02-naming-and-tagging.md`.
- [ ] Mandatory tag set agreed (Organization, BusinessUnit, Application, Environment, Role — CostCenter optional).
- [ ] CIDR plan confirmed: VPC `10.0.0.0/16`; subnets `10.0.1.0/24` public, `10.0.2.0/24` CCE, `10.0.3.0/24` database, `10.0.4.0/24` storage.
- [ ] Log retention agreed: app logs 30–180 days, CFW/security logs ≥ 90 days, CTS audit ≥ 180 days.
- [ ] Backup targets agreed: RDS retention 7–35 days, bastion CBR daily / 30-day retention; **RPO and RTO defined and written down**.
- [ ] Incident severity ladder adopted (P1–P4, per project document §12.4) and an on-call/notification order agreed.
- [ ] A shared runbook location chosen (git repo — this folder is a good start).



## 7. Communication

- [ ] SMN subscription emails/phone numbers collected from all four members (for alerts topics).
- [ ] Weekly 30-min sync scheduled (agenda template in `01-roadmap.md`).