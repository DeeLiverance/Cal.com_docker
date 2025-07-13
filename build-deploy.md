# Cal.com Docker Build & OCI Deployment Guide

> CI/CD pipeline at `.github/workflows/ci-cd.yml`


> **Step 0:** If you haven’t set up your local tools or logged in to Oracle Cloud yet, **stop and follow `env.md` first.**  
> Once `oci setup config` and `docker login $OCIR_NS.ocir.io` both work, come back here.

---

## 1  Scope
Goal: take the official Cal.com Docker repo (<https://github.com/calcom/docker>) and run a working container on Oracle Cloud Infrastructure (OCI) — Sydney region, free tier.

## 2  Prerequisites
*(Your machine should already have these after finishing `env.md`.)*

| Tool | Version |
|------|---------|
| Docker Engine | 24.x |
| OCI CLI | 3.x |
| Terraform | ≥1.7 |
| Git | ≥2.40 |

---

## 3  Local environment file
Create `.env` in the project root (only for local testing):

```
DATABASE_URL=postgresql://USER:PASSWORD@HOST:PORT/cal
REDIS_URL=redis://HOST:6379
NEXTAUTH_SECRET=replace-me
SMTP_USER=
SMTP_PASS=
```

> Production secrets will live in **OCI Vault** (see plan.md §4).

---

## 4  Build the image locally

> In CI, this build is automated via `.github/workflows/ci-cd.yml`
```bash
# 1. Clone the Docker repo
git clone https://github.com/calcom/docker.git cal-docker
cd cal-docker

# 2. (Optional) pin Cal.com version
export CAL_VERSION=2.15.5
sed -i "s/ARG CAL_VERSION=.*/ARG CAL_VERSION=$CAL_VERSION/" Dockerfile

# 3. Build
DOCKER_BUILDKIT=1 docker build -t wingspanai/cal:$CAL_VERSION .

# 4. Smoke‑test
docker run -p 3000:3000 --env-file ../.env wingspanai/cal:$CAL_VERSION
# visit http://localhost:3000
```

---

## 5  Push the image to Oracle Container Registry (OCIR)

> In CI, this push is automated via `.github/workflows/ci-cd.yml`
```bash
export OCIR_NS=$(oci os ns get | jq -r '.data')
docker tag wingspanai/cal:$CAL_VERSION $OCIR_NS.ocir.io/wingspanai/cal:$CAL_VERSION
docker push $OCIR_NS.ocir.io/wingspanai/cal:$CAL_VERSION
```
*(If you prefer Docker Hub, tag & push there instead.)*

---

## 6  Provision OCI infrastructure (Terraform)
```bash
cd ../iac
cp example.auto.tfvars prod.au.auto.tfvars   # fill in tenancy_id, compartment_id, image tag
terraform init
terraform apply -var-file=prod.au.auto.tfvars
```
Outputs include `lb_public_ip` — create / update DNS `calendar.wingspanai.com.au` → that IP.

---

## 7  One‑off console steps
1. **TLS** – attach your certificate to the Load Balancer (listener 443).  
2. **Min instances** – set **minInstances=1** on the Container Instance to avoid cold starts.  
3. **Vault secrets** – add env vars to OCI Vault and map them into the Container Instance spec. See `runbooks/monitoring.md` for probe configs and `runbooks/rollback.md` for rotation & rollback drills.

---

## 8  Verify deployment
```bash
curl -I https://calendar.wingspanai.com.au/healthz   # expect HTTP/1.1 200 OK
```
Log in, create the first organisation, and book a test slot.

---

## 9  Troubleshooting quick table

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| 502 Bad Gateway from LB | Container failed health probe | `oci logs tail --service container` |
| “Database is starting” loop | Wrong `DATABASE_URL` or firewall | Check security lists, rotate password |
| Invite emails not sent | SMTP creds missing | Add secrets in Vault, redeploy |

---

## 10  Next steps
- Automate blue/green rollout via **OCI DevOps**.  
- Add GitHub Action to build, push, and `terraform apply` on every tag (see `.github/workflows/ci-cd.yml`).  
- Integrate WingSpanAi CRM API once the admin org exists.

---

*Last updated: 2025‑07‑13 (AWST)*
