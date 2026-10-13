# Alireza — Network & Cloud Foundations

**Ownership:** the environment and its network connections — VPC, subnets, routes, NAT/EIP, security groups, bastion, ELB, DNS, CFW, Anti-DDoS.
**Pairs with:** Davood on ELB/Ingress, Ali on network security.
**Weeks:** 1–4 are yours; daily CFW log review after go-live.

All names/tags: see `../02-naming-and-tagging.md`. All console paths below refer to the Pardis Cloud web console (services are listed in the console's service catalog by their acronym).

---

## AR-1 — VPC, subnets, route tables (Week 1)

**Prerequisite:** M0 done (your IAM user has VPC permissions).

1. Console → **Networking → Virtual Private Cloud (VPC) → Create VPC**.
   - Name: `pc-sre-shop-prod-vpc-main-001`, CIDR: `10.0.0.0/16`, region/AZ per team agreement.
   - Create the first subnet in the same dialog: `pc-sre-shop-prod-snet-public-001`, `10.0.1.0/24`.
   - Apply the full mandatory tag set (Role: `main` / `public`).
2. VPC → Subnets → **Create Subnet** three more times:
   - `pc-sre-shop-prod-snet-app-001` — `10.0.2.0/24` (Role: `app`)
   - `pc-sre-shop-prod-snet-db-001` — `10.0.3.0/24` (Role: `db`)
   - `pc-sre-shop-prod-snet-storage-001` — `10.0.4.0/24` (Role: `backup`)
   - Enable DNS on each subnet; leave DHCP defaults.
3. VPC → Route Tables: create `pc-sre-shop-prod-rtb-app-001` and associate the app, db and storage subnets with it. (The default route to NAT is added in AR-2.)
4. **Verify:** all 4 subnets show *Available*; tags present on VPC and every subnet.

## AR-2 — NAT Gateway + EIP (Week 1)

1. Console → **Networking → Elastic IP → Assign EIP**: `pc-sre-shop-prod-eip-nat-001`, dedicated bandwidth sized for updates/API calls (start 10 Mbit/s; Cloud Eye will show if it needs more).
2. Console → **Networking → NAT Gateway → Create**:
   - Name `pc-sre-shop-prod-nat-main-001`, VPC = the project VPC, subnet = **public** subnet, small spec is fine for Phase 1.
3. Open the NAT Gateway → **Add SNAT Rule**: scope = subnet, add one SNAT rule each for the app, db, and storage subnets, all using the NAT EIP.
4. Edit `pc-sre-shop-prod-rtb-app-001`: add default route `0.0.0.0/0 → NAT gateway`.
5. **Verify:** launch a temporary ECS in the app subnet with no EIP, `curl -I https://example.com` succeeds, delete the test ECS. (This is Milestone M1 evidence.)

## AR-3 — Security groups (Week 1)

Console → **VPC → Access Control → Security Groups**. Create five groups, deny-by-default (delete permissive default rules), reference **other security groups as sources** instead of CIDRs wherever possible:

| SG | Inbound rules |
|---|---|
| `pc-sre-shop-prod-sg-elb-001` | TCP 80, 443 from `0.0.0.0/0` |
| `pc-sre-shop-prod-sg-app-001` | All TCP from `sg-elb` (health checks + traffic); node-to-node from itself |
| `pc-sre-shop-prod-sg-db-001` | TCP 3306 from `sg-app`; TCP 3306 from `sg-bastion` (admin) |
| `pc-sre-shop-prod-sg-cache-001` | TCP 6379 from `sg-app`; TCP 6379 from `sg-bastion` (admin) |
| `pc-sre-shop-prod-sg-bastion-001` | TCP 22 from **office IP ranges only** (from prerequisites list) |

Outbound: allow all for Phase 1 (CFW handles egress control at the boundary — AR-7). Tag every SG.

**Verify with Ali:** he'll review the rules against least-privilege before Week 2 starts.

## AR-4 — Bastion host (Week 2)

1. Console → **Computing → Elastic Cloud Server (ECS) → Create**:
   - Name `pc-sre-shop-prod-ecs-bastion-001`; small flavor (2 vCPU / 4 GB is enough); public image (latest LTS Linux); public subnet; SG `sg-bastion`; bind a new EIP or use the console-provided one.
   - **Login mode: key pair** — use the team keypair; passwords disabled.
2. Post-install hardening:
   - Add each member's public SSH key to `~/.ssh/authorized_keys`.
   - `PermitRootLogin no`, `PasswordAuthentication no` in `sshd_config`; restart sshd.
   - Install `mysql-client`, `redis-tools` (for Arashk), `kubectl` (for Davood).
   - Enable OS auditd / verbose SSH logging (Ali will pull these into LTS).
3. Hand over to **Ali** for HSS agent install (AL-5) and to **Arashk** for CBR backup binding (AK-6).
4. **Verify:** each teammate can `ssh` from the office range; connection from any other IP times out.

## AR-5 — ELB (Week 3, pair with Davood)

**Prerequisite:** Davood's Ingress (DA-5) exists — decide together whether ELB targets the ingress controller NodePorts or uses the CCE-integrated ELB ingress (preferred: let CCE create/manage the listener via ingress annotations, then rename/tag the ELB to convention).

1. Console → **Networking → Elastic Load Balance → Create**: dedicated LB, name `pc-sre-shop-prod-elb-public-001`, VPC + public subnet, bind `pc-sre-shop-prod-eip-elb-001`.
2. Listeners:
   - HTTPS 443: upload/select the TLS certificate (from prerequisites), backend = CCE node group / ingress NodePort.
   - HTTP 80: redirect to 443.
3. Health check: HTTP on the app's `/healthz`, interval 5 s, threshold 3. Backend SG must admit health checks (`sg-app` from `sg-elb` — already done in AR-3).
4. Enable ELB access logging (to LTS — coordinate with Ali, AL-8).
5. **Verify:** `curl -k https://<elb-eip>/` returns the frontend; kill one pod and confirm traffic keeps flowing (M3 evidence).

## AR-6 — DNS (Week 4)

1. Console → **Networking → Domain Name Service (DNS) → Public Zones → Create**: zone for the shop domain, name resource `pc-sre-shop-prod-dns-main-001`.
2. Delegate the domain: set the registrar's NS records to the Pardis DNS name servers shown in the zone.
3. Records: `A` (or CNAME per WAF onboarding — see below) for `shop.<domain>` → ELB EIP; optionally `www` CNAME.
4. **Order matters with WAF:** when Ali onboards the domain to WAF (AL-4), WAF gives a CNAME — the DNS record must then point to the **WAF CNAME**, and WAF forwards to the ELB EIP. Do this switch together with Ali.
5. **Verify:** `dig shop.<domain>` resolves; full path returns the app.

## AR-7 — Cloud Firewall (CFW) (Week 4)

1. Console → **Security → Cloud Firewall → Enable/Buy**: standard edition, attach to the EIPs (NAT EIP + ELB EIP) — name `pc-sre-shop-prod-cfw-main-001`.
2. Access control policies (deny-by-default):
   - Policy 1 — Internet → ELB EIP: permit TCP 80, 443. Logging **on**.
   - Policy 2 — Internet → anything else: deny. Logging **on**.
   - Policy 3 — office IP ranges → bastion EIP TCP 22: permit. Logging **on**.
   - Egress: permit established/necessary outbound (OS updates, external APIs), deny + log the rest.
3. Enable **IPS in prevention mode** (not detect-only) with automatic signature updates.
4. Enable log shipping to LTS group `pc-sre-shop-prod-lts-audit-001` (coordinate with Ali).
5. Document every rule with a one-line business justification (keep in this repo).
6. **Verify:** port scan from outside shows only 80/443 (and 22 from office); scan attempts appear in CFW logs; run a test with Ali where a blocked connection triggers review.

## AR-8 — Anti-DDoS (Week 4)

1. Console → **Security → Anti-DDoS**: confirm **Anti-DDoS Basic** is active on both EIPs (it is enabled by default for EIPs — verify, don't assume).
2. Set traffic-cleaning thresholds appropriate to expected legitimate peak traffic.
3. Configure alarm notification → SMN topic `pc-sre-shop-prod-smn-security-001` (Ali creates topics in AL-6; thresholds can be set now, notifications wired in Week 5).
4. Evaluate whether Anti-DDoS Pro is warranted before a marketing launch; note the decision in the runbook.

## Ongoing after go-live

- **Daily:** review CFW blocked connections and IPS alerts in LTS.
- **Weekly:** CFW policy review — remove unused rules, tighten broad ones.
- **Monthly:** patch bastion (only after Arashk takes an on-demand CBR backup); review SG rules with Ali.
