# Cal.com on Oracle — Project Plan

> **Docs flow**
> 1. `env.md` – one-time tool & OCI setup
> 2. `build-deploy.md` – build, push, deploy
> 3. This plan – architecture & ops standards

## Quick Start Checklist

- [ ] env.md: install OCI CLI, Docker, Terraform & authenticate to OCI
- [ ] build-deploy.md: build, tag & push the Cal.com Docker image
- [ ] plan.md Step 0: build & push image (per build-deploy.md §§4-5)
- [ ] plan.md Steps 1–6: provision VCN, DB, Redis, Storage, CI, LB via Terraform
- [ ] plan.md: configure backups, monitoring & alarms

## 1. Objectives
1. **Deploy a self-hosted Cal.com instance** in Oracle Cloud Infrastructure (OCI) Sydney region.
2. **Keep infra in the free-tier** until production traffic > 250 daily bookings.
3. **Expose an admin API** so WingSpanAi CRM can create sub-accounts and white-label settings.
4. **Reuse OCI for ERPNext** on the same VCN, but isolate databases and secrets.
5. **Automate provisioning** with Terraform kept in a separate repo/folder (`/iac`).

## 2. Success Criteria
| Metric | Target |
| ------ | ------ |
| First booking latency | ≤ 1 s p95 |
| Cold-start failures | 0 during first 30 days |
| Backup recovery point | ≤ 15 min data loss |
| Free-tier budget | AU $0/mo for ≤ 10 active users |
| Cal.com version | vX.Y.Z (explicit for upgrades) |

## 3. High-Level Architecture
```
[ Client ] ─┬──▶  [ OCI Load Balancer (HTTPS+WS) ] ───▶ [ Container Instance: Cal.com ]
           │                                        │
           │                                        ├──▶  PostgreSQL (OCI Database with PG)
           │                                        │
           │                                        └──▶  Redis (OCI Cache)
           │
           └──▶  [ Object Storage ]  (ICS files, avatars)
```

*Docker-built Cal.com image is pulled from **Docker Hub**.*

## 4. Prerequisites  *(see env.md for full commands)*
- **OCI tenancy & user** with `Administrator` policy.
- **CLI**: `oci` + `docker` + `terraform` installed locally.
- **DNS zone** for `calendar.wingspanai.com.au` in Cloudflare (or OCI DNS).
- **TLS cert** (Let’s Encrypt or ACM-imported).

## 5. Provisioning Flow (console summary)
0. **Build & push Cal.com image** – follow build-deploy.md steps 4-5
1. **Networking** – Create VCN `wingspan-prod`, public + private subnets.
2. **PostgreSQL** – Autonomous PG `cal-db`, 2 OCPU, 50 GB.
3. **Redis** – Cache cluster `cal-redis`, 1 GB, single AZ.
4. **Object Storage** – Bucket `cal-assets`.
5. **Container Instance** – 2 OCPU Ampere A1, pull `docker.io/wingspanai/cal:latest`.
  - Or pin via OCI image digest in Terraform for immutability.
6. **Load Balancer** – Flexible LB, HTTPS 443 → CI 3000, WS pass-through.
7. **Backups** – Enable automatic PG snapshots; weekly bucket export of Redis RDB.
8. **Monitoring & Alarms** – CPU > 75 %, latency > 500 ms, LB 5xx rate.

*Detailed **Terraform** equivalents live in `/iac/main.tf`.*

## 6. Terraform Directory (`/iac`)
```
/iac
 ├─ versions.tf
 ├─ variables.tf
 ├─ network.tf       # VCN + subnets
 ├─ db.tf            # PostgreSQL
 ├─ redis.tf         # Cache
 ├─ compute.tf       # Container Instance + LB
 ├─ storage.tf       # Bucket & policies
 └─ outputs.tf
```
Apply with:
```bash
cd iac
terraform init
terraform plan -var-file=prod.au.tfvars
terraform apply
```
_Terraform apply example lives in build-deploy.md §6._

## 7. Backup & Restore Strategy
| Component | Backup method | Frequency | Restore test |
| --------- | ------------- | --------- | ------------ |
| PostgreSQL | Auto-snapshots | Daily | Monthly point-in-time restore into `cal-db-restore` |
| Redis | RDB export to Object Storage | Hourly | Quarterly import drill |
| Object Storage | Lifecycle versioning | n/a | Spot-check random file monthly |

## 8. Integration Hooks
- **WingSpanAi CRM** – Use Cal.com Admin API `/v1/organizations` to create tenant + branding.
- **HubSpot** – Cal.com native OAuth; map contact fields.
- **ERPNext** – Webhook `booking.created` → Frappe REST to create `Calendar Booking` DocType.

## 9. Timeline (draft)
| Week | Milestone |
| ---- | --------- |
| 1 | Finalise plan.md + Terraform skeleton |
| 2 | Network + DB + Redis provisioned via Terraform (Dev) |
| 3 | Container image built/tested locally; CI deployed (Stage) |
| 4 | DNS + SSL live; smoke tests pass |
| 5 | WingSpanAi CRM integration PoC |

## 10. Open Questions / Risks
1. Free-tier Redis availability SLA? Consider paid tier if Redis is mission-critical.
2. OCI email relay limits for booking notifications.
3. Cal.com upgrade cadence ⇒ CI restarts; automate zero-downtime rollouts via blue/green.

## 11. VS Code Extensions

| Extension | ID | Purpose |
|-----------|-----------------------------|------------------------------------------------|
| Docker | ms-azuretools.vscode-docker | Build, run, debug Cal.com containers; dev-container workflow |
| Remote – Containers | ms-vscode-remote.remote-containers | Launch a `.devcontainer` with standard Node, Terraform, OCI CLI versions |
| Remote – SSH | ms-vscode-remote.remote-ssh | Edit or hot-patch directly on Oracle VMs/Container Instances |
| HashiCorp Terraform | hashicorp.terraform | HCL syntax, lint, fmt, and `terraform validate` on save |
| OCI Dev Tools | oracledevs.oci-devops-toolkit | Snippets for OCI resources, log/metric view, profile switcher |
| ESLint | dbaeumer.vscode-eslint | Enforce Next.js + TypeScript lint rules; auto-fix via Husky |
| Prettier | esbenp.prettier-vscode | Consistent code formatting aligned with project style guide |
| Tailwind CSS IntelliSense | bradlc.vscode-tailwindcss | Class autocomplete, config validation, design tokens hover |
| Prisma | Prisma.prisma | Schema autocomplete + quick navigation for DB layer |
| Jest Runner | firsttris.vscode-jest-runner | Run/Debug individual Jest or Vitest tests via code lens |
| GitLens | eamodio.gitlens | Enhanced blame, history, and PR insights for faster reviews |
| Markdown All in One | yzhang.markdown-all-in-one | TOC generation, linting, shortcuts for `plan.md` & docs |
| Error Lens | usernamehw.errorlens | Inline surfacing of TypeScript/HCL errors for rapid fixes |

## 12. Hardening & CI/CD Checklist

- [x] CI/CD pipeline configured (`.github/workflows/ci-cd.yml`)
- [x] Monitoring runbook added (`runbooks/monitoring.md`)
- [x] Rollback playbook added (`runbooks/rollback.md`)
- [ ] Secret rotation process documented (update `env.md` & Terraform)
- [ ] Backup & restore drills documented (plan.md §7 & runbooks)
- [ ] Observability SLO definitions completed (`runbooks/monitoring.md`)


---

*Last updated: 2025-07-13 (AWST)*

Next: kitchen setup → env.md; recipe → build-deploy.md
