# Ali — Security, Observability & Acceptance

**Ownership:** access controls, operational visibility, validation — IAM/MFA, WAF, HSS, LTS, CTS, Cloud Eye, SMN, flow logs, security/load tests, incident procedures.
**Pairs with:** everyone — each service owner helps you connect and validate their telemetry.
**Weeks:** 0–1 (IAM), 4 (WAF/HSS), 5 (observability), 6 (acceptance lead).

All names/tags: see `../02-naming-and-tagging.md`.

---

## AL-1 — IAM foundation (Week 0/1 — blocks everyone else)

1. Console → **Management → Identity and Access Management (IAM)**.
2. Secure the root/tenant account: strong password, **MFA**, store in the password manager, then stop using it.
3. Create **user groups** and attach system policies (least privilege):

| Group | Members | Policies (approximate console names) |
|---|---|---|
| `admins` | ali | Security Administrator, Tenant Administrator (scoped), IAM full |
| `network-ops` | alireza | VPC/ELB/NAT/DNS/CFW/Anti-DDoS FullAccess, ECS FullAccess (bastion) |
| `app-dev` | davood | CCE FullAccess, SWR FullAccess, LTS ReadOnly |
| `data-ops` | arashk | RDS/DCS/OBS/CBR FullAccess |
| `readonly` | everyone | Tenant ReadOnly (for cross-visibility) |

4. Create the four **individual users** (no shared accounts), force password reset + **MFA at first login**.
5. Create a **service user for the application**: `svc` account with programmatic access only, policy = write/read on the three app OBS buckets only. Give its access key to Davood for the `shop-obs-credentials` Secret. Set a 90-day key-rotation reminder.
6. Document: access-review calendar (quarterly), key rotation (90 days), no personal names in *resource* names (users are fine — resources are not).
7. **Verify (M0):** each member logs in with MFA and can open exactly their services; the app service user cannot open the console.

## AL-2 — IAM hardening pass (Week 1)

- Review Alireza's SGs against least privilege before Week 2 (gate for M1).
- Confirm nobody has used root since day 0 (check CTS as soon as AL-9 is up — retroactively).
- Write the four "recommended roles" mapping from the project doc (Admin/Developer/Operator/ReadOnly) into the runbook so future joiners get the right group.

## AL-4 — WAF (Week 4)

**Prerequisites:** ELB live (AR-5), DNS zone (AR-6), TLS cert available.

1. Console → **Security → Web Application Firewall → Buy/Enable** (cloud mode), policy name `pc-sa-shop-prod-waf-web-001`.
2. **Onboard the domain:** add `shop.<domain>`, upload the TLS certificate, set origin = ELB EIP :443.
3. WAF issues a **CNAME** → coordinate with Alireza to point the DNS record at the WAF CNAME (traffic then flows DNS → WAF → ELB).
4. Protection configuration:
   - Enable **OWASP Top 10 / basic web protection** rule set — start in **log/learning mode** for 3–5 days to catch false positives, then switch to **block**.
   - Enable CC (rate-limit) protection with a sane threshold for the shop.
   - Add geo-blocking only if the business asks for it.
   - Custom rules for app-specific paths (admin endpoints → office IPs only).
5. Ship WAF logs to LTS group `pc-sa-shop-prod-lts-audit-001`.
6. **Verify (M4):** `curl "https://shop.<domain>/?q=' OR 1=1--"` returns a WAF block page; the event appears in WAF logs. Legitimate flows are unaffected after learning mode.

## AL-5 — HSS on the bastion (Week 4)

1. Console → **Security → Host Security Service** → buy a quota (enterprise edition if budget allows, for IPS features).
2. Install the HSS **agent** on `pc-sa-shop-prod-ecs-bastion-001` (console gives the one-line install command; run it via SSH with Alireza).
3. Enable all protection modules: intrusion detection, vulnerability scan, baseline check, malware detection.
4. Run the first **baseline security check + vulnerability scan**; file fixes with Alireza (patching follows the CBR-backup-first rule).
5. Route HSS alarms → SMN `pc-sa-shop-prod-smn-security-001` (after AL-6).
6. **Verify:** HSS console shows the host as protected; a test event (e.g. 5 failed SSH logins) raises an alert.

## AL-6 — SMN topics (Week 5 — do first, everything alerts into these)

1. Console → **Management → Simple Message Notification** → create three topics:
   - `pc-sa-shop-prod-smn-alert-001` — critical alarms → **email + SMS** for all four members.
   - `pc-sa-shop-prod-smn-ops-001` — routine/ops notifications → email.
   - `pc-sa-shop-prod-smn-security-001` — CFW/WAF/HSS/Anti-DDoS events → email + SMS (Ali + Alireza at minimum).
2. Add subscriptions (from the contact list in prerequisites); every member must **confirm** the subscription email/SMS.
3. **Verify:** publish a test message to each topic; confirm receipt by all subscribers.

## AL-7 — Cloud Eye: dashboards & alarms (Week 5)

1. Console → **Management → Cloud Eye (CES)**. Metrics from ELB, CCE, RDS, DCS, OBS, CFW, NAT flow automatically once resources exist.
2. Build dashboard `pc-sa-shop-prod-dashboard-main-001` (ops view) with: ELB request count/latency/5xx, CCE node CPU/mem + pod status, RDS CPU/mem/connections/disk, DCS memory/hit-rate/connections, NAT bandwidth, CFW blocked connections. Optionally a second slim executive view.
3. Alarm rules (all → `smn-alert-001` unless noted):

| Alarm | Threshold |
|---|---|
| Node/RDS/DCS CPU | > 80 % for 5 min |
| Memory | > 85 % for 5 min |
| RDS disk free | < 20 % |
| ELB 5xx error rate | > 5 % |
| ELB response time | > 2 s |
| DCS memory | > 85 % / evictions rising |
| CFW blocked connections | > agreed threshold → `smn-security-001` |
| CBR backup failure | any → `smn-alert-001` |

4. **Verify (M5):** force one alarm (e.g. stress a pod with Davood) and confirm the SMS/email arrives.

## AL-8 — LTS: centralized logging (Week 5)

1. Console → **Management → Log Tank Service** → create log groups `pc-sa-shop-prod-lts-app-001` (retention 30–180 days per agreement) and `pc-sa-shop-prod-lts-audit-001` (retention ≥ 90 days).
2. Connect sources (each owner helps you — this is the pairing from the task sheet):
   - **CCE pod logs** (Davood): LTS add-on installed in DA-2 → stream per service (stdout/stderr).
   - **ELB access logs** (Alireza).
   - **CFW traffic + IPS logs** (Alireza, from AR-7).
   - **WAF logs** (yours, AL-4).
   - **RDS slow query log** (Arashk).
   - **Bastion system/SSH logs** (Alireza — install the LTS/ICAgent on the bastion).
3. Create saved searches: 5xx spikes, WAF blocks by rule, CFW denies by source IP, slow queries > 1 s, SSH logins.
4. **Verify:** each source shows fresh entries; a search for a known test event returns it.

## AL-9 — CTS: audit trail (Week 5)

1. Console → **Management → Cloud Trace Service** → enable the tracker `pc-sa-shop-prod-cts-audit-001`.
2. Deliver traces to Arashk's bucket `pc-sa-shop-prod-obs-audit-001`; retention **≥ 180 days**.
3. Set key-event notifications (IAM policy changes, SG changes, CFW policy changes, resource deletion) → `smn-security-001`.
4. **Verify:** perform a harmless change (e.g. tag edit) and find it in the trace list; retroactively confirm no root-account usage.

## AL-10 — VPC flow logs (Week 5, with Alireza)

1. VPC console → Flow Logs → create `pc-sa-shop-prod-flowlog-main-001` for all four subnets → LTS `lts-audit` group.
2. Saved search for rejected flows; baseline a day of normal traffic for anomaly comparison.

## Week 6 — acceptance (you coordinate the gate)

Run and record the full validation from the project doc §11.3:

1. **Functional** (with Davood): domain access, registration/login, browse, cart, order, payment, DB/cache/OBS operations.
2. **Security** (yours):
   - External port scan → only 80/443 (+22 from office).
   - WAF blocks SQLi/XSS test strings; CFW logs and blocks a disallowed port probe; IPS detection event visible.
   - SG spot checks (db reachable only from app SG); TLS config check (TLS 1.2+, valid chain); IAM permission spot checks; HSS active.
3. **Performance** (with Davood): load test against the domain; verify HPA scale-out/in, cache hit rate, DB latency; record baseline numbers in the runbook.
4. **Recovery** (Arashk's evidence report): restore drills done, RPO/RTO met.
5. Publish the **runbooks**: daily/weekly/monthly ops (roadmap table), P1–P4 incident response procedure with the SMN escalation path, common-issues playbook (project doc §12.5).
6. Sign off M6 → go-live, hypercare rotation for week 1.

## Ongoing after go-live

- **Daily:** Cloud Eye dashboards + active alarms review.
- **Weekly:** IAM access log review; error log analysis in LTS.
- **Monthly:** security audit, IAM access review, monitoring threshold tuning, compliance checks.
- **Quarterly:** full access review; incident-response tabletop drill.
