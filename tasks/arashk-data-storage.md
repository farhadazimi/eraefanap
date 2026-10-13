# Arashk — Data, Storage & Recovery

**Ownership:** application data and recoverability — RDS MySQL, DCS Redis, OBS, schemas/migrations, data-access configuration, native backups, CBR, restore testing.
**Pairs with:** Davood on application behavior, Ali on recovery evidence.
**Weeks:** 2, 3, 5 are yours; restore drills forever after.

All names/tags: see `../02-naming-and-tagging.md`.

> Note: the project document calls the backup service **CBS**; the official Pardis catalog calls it **CBR (Cloud Backup and Recovery)**. Same service — look for *CBR* in the console.

---

## AK-1 — RDS MySQL (Week 2)

**Prerequisites:** Alireza's db subnet + `sg-db` (AR-1, AR-3).

1. Console → **Databases → RDS → Create Instance**:
   - Engine **MySQL 8.0+**; name `pc-sre-shop-prod-rds-mysql-001`.
   - VPC = project VPC, subnet = `pc-sre-shop-prod-snet-db-001`, SG = `pc-sre-shop-prod-sg-db-001`.
   - Storage: **SSD with autoscaling enabled**; instance class sized from expected load (start modest — 2 vCPU / 8 GB — Cloud Eye data will justify resizing).
   - Single instance for Phase 1 (primary/standby is Phase 2), **no public access**.
   - Set the root password → team password manager.
2. Instance → Backups & Restorations: **automated daily backup**, window ~01:00, **retention 7–35 days per the agreed RPO** (write the chosen value in the runbook). Point-in-time recovery comes with this.
3. Enable **slow query log** (→ LTS later with Ali, AL-8).
4. Create the application database and a **least-privilege app user** (no root for the app):

```sql
CREATE DATABASE shop CHARACTER SET utf8mb4;
CREATE USER 'shop_app'@'10.0.2.%' IDENTIFIED BY '<strong-password>';
GRANT SELECT, INSERT, UPDATE, DELETE ON shop.* TO 'shop_app'@'10.0.2.%';
-- a separate migration user gets DDL rights:
CREATE USER 'shop_migrate'@'10.0.1.%' IDENTIFIED BY '<strong-password>';
GRANT ALL PRIVILEGES ON shop.* TO 'shop_migrate'@'10.0.1.%';

-- the order-service has its own database on the same instance, with the same
-- app/migrate split (matches Bookland/db/mysql-init/01-databases.sql locally):
CREATE DATABASE shop_orders CHARACTER SET utf8mb4;
CREATE USER 'orders_app'@'10.0.2.%' IDENTIFIED BY '<strong-password>';
GRANT SELECT, INSERT, UPDATE, DELETE ON shop_orders.* TO 'orders_app'@'10.0.2.%';
CREATE USER 'orders_migrate'@'10.0.1.%' IDENTIFIED BY '<strong-password>';
GRANT ALL PRIVILEGES ON shop_orders.* TO 'orders_migrate'@'10.0.1.%';
```

5. **Verify (M2):** from the bastion, `mysql -h <rds-endpoint> -u shop_app -p` connects; from anywhere else it must fail. Hand endpoint + app credentials to Davood (for `shop-db-credentials`).

## AK-2 — DCS Redis (Week 2)

1. Console → **PaaS/Middleware → Distributed Cache Service (DCS) → Create**:
   - **Redis 5.0+**; name `pc-sre-shop-prod-dcs-redis-001`; master/standby if budget allows, else single-node.
   - VPC/subnet/SG: db subnet + `pc-sre-shop-prod-sg-cache-001`.
   - Set a Redis password (→ password manager). Memory: start 4 GB.
2. Enable **persistence** (AOF/RDB per console options) — the project requires durability for sessions/carts.
3. Set eviction policy `volatile-lru` and document that the app must set **TTLs** on cache keys (agree defaults with Davood: sessions 30 min, catalog cache 5–15 min).
   Implemented key patterns/TTLs (full table in `Bookland/docs/application-inventory.md`): Django session cache 1800 s; `catalog:products:v1` 300 s; `cart:*` / `wishlist:*` hashes/sets 7 days; `idempotency:orders:*` 3600 s (order-service).
4. **Verify (M2):** from the bastion, `redis-cli -h <dcs-endpoint> -a <password> ping` → `PONG`. Hand endpoint + password to Davood.

## AK-3 — OBS buckets (Week 2)

Console → **Storage → Object Storage Service (OBS) → Create Bucket**, four times:

| Bucket | Access | Versioning | Encryption | Lifecycle |
|---|---|---|---|---|
| `pc-sre-shop-prod-obs-images-001` | private + **signed URLs** (or public-read if the team prefers) | on | on | none |
| `pc-sre-shop-prod-obs-invoices-001` | **private** | on | **on (required)** | move to cold storage after 1 year |
| `pc-sre-shop-prod-obs-backup-001` | private | on | on | expire objects > 90 days (align with RPO) |
| `pc-sre-shop-prod-obs-audit-001` | private | on | on | retain ≥ 180 days (CTS requirement) |

1. Directory layout inside buckets (per project doc): `/images/products/`, `/images/avatars/`, `/invoices/{year}/{month}/`, `/backups/database/`.
   The app currently writes: backend-api → `images/products/<file>` in the images bucket; order-service → `invoices/{year}/{month}/order-{id}.json` in the invoices bucket (S3-compatible API; credentials via `OBS_ACCESS_KEY`/`OBS_SECRET_KEY`).
2. Bucket policies: only the app's IAM user (Ali creates it, AL-2) may write; deny public write everywhere; invoices/backup/audit deny public read.
3. **Verify (M2):** upload/download a test object with the app credentials via KooCLI or console; public URL of an invoice object must return access-denied. Hand endpoint + bucket names to Davood.

## AK-4 — Schema & migrations (Week 3)

**Prerequisites:** AK-1, bastion (AR-4).

1. Put migration scripts in this repo (`db/migrations/`, numbered) covering the project's required schema: user accounts/profiles, product catalog, order transactions, inventory, payment records.
2. Run migrations from the bastion as `shop_migrate` (or wire them into Davood's deploy pipeline as an init job — decide together).
3. Indexing: define indexes with the dev team for the known hot paths (login by email, product lookups, orders by user/date). Document them.
4. **Verify:** Davood's backend pods pass their readiness checks talking to the schema; a test order lands in the `orders` table (M3 evidence).

## AK-5 — Native backups verification + OBS protection (Week 5)

1. RDS: confirm the first automated backups exist (instance → Backups list); do one **manual restore to a new instance**, spot-check a table, delete the restored instance. Record duration → this is your **RTO evidence**.
2. DCS: verify persistence files/backup schedule in the DCS console (native backup for master/standby).
3. OBS: confirm versioning is active by overwriting a test object and restoring the previous version.
4. Export **CFW policies and other configuration** (Alireza provides) into `pc-sre-shop-prod-obs-backup-001/backups/config/` — the project requires config backup too.

## AK-6 — CBR: bastion disk backup (Week 5)

**Prerequisites:** bastion exists (AR-4). Scope note (from the project doc): RDS/DCS use their **native** backups; OBS uses versioning; **CBR is for EVS disks** (bastion, and CCE node data disks only if the app ever writes persistent local data — it shouldn't; state belongs in RDS/OBS).

1. Console → **Storage → Cloud Backup and Recovery (CBR) → Create Vault**:
   - Server/disk backup vault, name `pc-sre-shop-prod-cbr-backup-001`, size ≥ 2× bastion disk total.
2. **Associate** the bastion ECS (its EVS system + data disks) with the vault.
3. Create policy `pc-sre-shop-prod-policy-backup-001`: **daily at 02:00, retain 30 days**; bind it to the vault.
4. Take one **on-demand backup now** to prove the pipeline; check it completes.
5. Agree the pre-patch procedure with Alireza: he never patches the bastion before you take an on-demand backup.

## Week 6 — restore tests (your milestone gate)

1. **Bastion restore drill:** restore the latest CBR backup of the bastion system disk **to a new disk/server**, boot it, verify SSH keys and tools are intact. Record time taken (RTO evidence). Tear down the copy.
2. **RDS point-in-time drill:** restore the prod instance to a point 1 hour back into a **new instance**, verify a known row, delete it.
3. Deliver a one-page **recovery evidence report** to Ali (screenshots, durations, RPO/RTO met yes/no) — he needs it for acceptance.

## Ongoing after go-live

- **Daily:** check CBR console — nightly backup succeeded; check RDS backup list.
- **Weekly:** review slow query log; verify backup integrity.
- **Monthly:** full restore test (alternate bastion/RDS); review OBS lifecycle transitions and storage costs; DB maintenance (analyze/optimize hot tables).
- Own the **RPO/RTO document**; update it whenever retention or schedules change.
