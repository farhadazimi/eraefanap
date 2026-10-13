# Davood — CCE & Application Delivery

**Ownership:** getting the application built and running — application inventory, Docker images, SWR, CCE, manifests, configuration, Ingress, replicas, autoscaling, deployment/rollback.
**Pairs with:** Arashk on application/data integration, Alireza on ELB/Ingress.
**Weeks:** 2–3 are yours; deployment/rollback duty afterwards.

All names/tags: see `../02-naming-and-tagging.md`.

---

## DA-0 — Application inventory (Week 0, part of prerequisites)

> **Done** — the inventory lives in the Bookland repo: `Bookland/docs/application-inventory.md`
> (runtime/ports/resources per service, env → ConfigMap/Secret mapping, probe paths,
> Redis key patterns + TTLs, OBS object layout). Egress note for Alireza: the app needs
> **no third-party APIs** — only RDS 3306, DCS 6379, OBS 443 and in-cluster HTTP.

For each of the three services (frontend, backend-api, order-service) record in this repo:

- Runtime/language and version, exposed port, resource footprint estimate.
- External dependencies: MySQL, Redis, OBS, third-party APIs (each needs an egress path — tell Alireza).
- Config parameters → will become ConfigMap entries; credentials → Secrets.
- Health endpoints (`/healthz`, `/readyz`) — if missing, get them added **now**; probes and ELB checks depend on them.
  Implemented: all three serve `/healthz` + `/readyz`; backend-api also mirrors them at `/api/healthz` + `/api/readyz` for the ingress `/api` path.

## DA-1 — SWR: container registry (Week 2)

**SWR = SoftWare Repository for Container (Pardis equivalent of ECR).**

1. Console → **Containers → Software Repository for Container (SWR)** → Organizations → **Create Organization**: `pc-sre-shop-prod`.
2. Generate a login command: SWR console → *Generate Login Command* (long-term login token) — it produces a `docker login swr.<region>.pardis...` command.
3. Build, tag, push each image. Tag with the **git commit SHA or semver — never deploy `latest`**:

```bash
docker build -t swr.<region>.pardis.cloud/pc-sre-shop-prod/frontend:v1.0.0 ./frontend
docker push  swr.<region>.pardis.cloud/pc-sre-shop-prod/frontend:v1.0.0
# repeat for backend-api and order-service
```

4. Set repositories to **private**; grant the CCE cluster pull permission (SWR → repository → permissions, or use an imagePullSecret).
5. Enable image vulnerability scanning if available in the SWR console; fix criticals before deploying.
6. **Verify:** all three repos show the pushed tags in the SWR console (M2 evidence).

## DA-2 — CCE cluster (Week 3)

**Prerequisites:** Alireza's VPC/subnets/SGs (AR-1, AR-3).

1. Console → **Containers → Cloud Container Engine (CCE) → Create Cluster**:
   - Name `pc-sre-shop-prod-cce-main-001`; latest stable Kubernetes; VPC = project VPC; node subnet = `pc-sre-shop-prod-snet-app-001`; network model per console default (VPC network is fine for Phase 1).
   - HA control plane if budget allows (recommended for prod).
2. Create node pool `pc-sre-shop-prod-nodepool-app-001`:
   - **3 nodes minimum**, spread across AZs if the region offers them; flavor sized from DA-0 estimates (start 4 vCPU / 8 GB per node).
   - Node SG: `pc-sre-shop-prod-sg-app-001`. Key pair login, same key policy as bastion.
3. Install add-ons from the CCE add-on marketplace: metrics-server (needed for HPA), the CCE ingress/ELB controller, and the LTS log collection add-on (Ali needs it in AL-8).
4. Get `kubeconfig` from the cluster console; configure `kubectl` **from the bastion host** (cluster API should not be exposed publicly — choose private API endpoint).
5. **Verify:** `kubectl get nodes` from the bastion shows 3 Ready nodes.

## DA-3 — Namespaces, ConfigMaps, Secrets (Week 3)

**Prerequisites:** Arashk's RDS/DCS/OBS endpoints and credentials (AK-1..3).

1. `kubectl create namespace shop-prod`
2. ConfigMap `shop-config`: non-secret config — RDS endpoint, DCS endpoint, OBS endpoint + bucket names, log level, feature flags.
3. Secrets (`shop-db-credentials`, `shop-redis-credentials`, `shop-obs-credentials`): DB user/password (the **app user** Arashk creates, not root), Redis password, OBS access/secret key of the app's IAM user (from Ali). Create with `kubectl create secret generic` — don't commit secret values to git.
4. Create the SWR `imagePullSecret` in the namespace if repo-level permission wasn't granted in DA-1.

## DA-4 — Deployments, Services, probes, autoscaling (Week 3)

Write manifests (keep them in this repo, `k8s/` directory) — one Deployment + Service per component:

- **Deployment** `frontend` / `backend-api` / `order-service`:
  - `replicas: 2` minimum each (HA).
  - Image from SWR with the explicit version tag.
  - `envFrom` the ConfigMap; secrets mounted as env vars.
  - **Resource requests and limits set** (from DA-0 estimates) — mandatory, HPA won't work without requests.
  - **Liveness probe** on `/healthz`, **readiness probe** on `/readyz`.
  - `strategy: RollingUpdate`, `maxUnavailable: 0` for zero-downtime deploys.
- **Service** (ClusterIP) per component; frontend is the only one exposed via Ingress; backend-api and order-service are reached internally (`http://backend-api.shop-prod.svc`).
- **HPA** per Deployment: `minReplicas: 2`, `maxReplicas: 6`, target CPU 70 %.

```bash
kubectl apply -f k8s/ -n shop-prod
kubectl get pods -n shop-prod -w     # all Running and Ready
```

## DA-5 — Ingress (Week 3, pair with Alireza)

1. Create `shop-ingress` in `shop-prod`:
   - Path `/` → `frontend`; `/api` → `backend-api` (adjust to actual routing design).
   - Use the CCE ELB ingress annotations so the ingress binds to Alireza's ELB `pc-sre-shop-prod-elb-public-001` (or lets CCE create it — decide together, then apply convention name/tags to the ELB).
   - TLS terminates at the ELB (cert handled in AR-5); ingress serves HTTP behind it.
2. **Verify with Alireza:** `curl https://<elb-eip>/` and `/api/healthz` return correctly (`/api/readyz` must also be 200 — it proves RDS + DCS connectivity) → Milestone M3.

## DA-6 — Deployment & rollback procedure (Week 3, document; used forever after)

Write the runbook (in this repo) covering:

1. **Release:** build → push new tag to SWR → `kubectl set image deployment/backend-api backend-api=swr.../backend-api:v1.1.0 -n shop-prod` (or re-apply manifest) → watch rollout: `kubectl rollout status deployment/backend-api -n shop-prod`.
2. **Rollback:** `kubectl rollout undo deployment/backend-api -n shop-prod` — test this once before go-live so it's proven, not theoretical.
3. **Pre-release checklist:** Arashk informed if a release includes DB migrations (migrations run first, backward-compatible); on-demand RDS backup before schema changes.
4. Release windows and who approves prod deploys.

## Week 6 — your part of validation

- Drive fixes for functional test failures (register/login/browse/cart/order/uploads).
- Pair with Ali on the load test: confirm HPA scales pods out under load and back in after; record response times before/after cache warm-up (with Arashk).
- Demonstrate one full rollback during the validation window.

## Ongoing after go-live

- Own deployments/rollbacks; keep manifests in git as the single source of truth.
- Keep base images updated; rebuild + redeploy monthly or on critical CVEs (SWR scan reports).
- Review pod resource usage in Cloud Eye monthly with Ali; right-size requests/limits.
