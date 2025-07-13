
# Environment Setup Guide (One-time)

> Complete these steps **before** following `build-deploy.md`.

---

## Quick Checklist

- [ ] Install local tools: Docker Desktop/Engine, OCI CLI, Terraform, Git, jq
- [ ] Configure OCI CLI & generate API keys
- [ ] Login to OCI Registry (OCIR)
- [ ] Populate GitHub secrets (see section 3)
- [ ] Run sanity checks (see section 4)

## 1. Setup Python virtual environment

To isolate Python dependencies (like the OCI CLI), create and activate a venv:

```bash
# Create venv:
#   Windows:
python -m venv .venv
#   Bash/Linux/macOS:
python3 -m venv .venv

# Activate venv:
#   Windows (PowerShell):
.\.venv\Scripts\Activate.ps1
#   Bash/Linux/macOS:
source .venv/bin/activate
```

## 2. Install local tools

| Tool | Command / link | Notes |
|------|----------------|-------|
| **Docker Desktop / Engine** | <https://docs.docker.com/get-docker/> | Build and run containers |
| **OCI CLI** | `pip install oci-cli` | Oracle Cloud CLI (inside venv) |
| **Terraform** | Download from <https://developer.hashicorp.com/terraform/downloads> | Infrastructure as code (add to PATH) |
| **Git** | <https://git-scm.com/> | Version control |
| **jq** | Windows: `choco install jq`; Linux: `apt-get install jq`; macOS: `brew install jq` | JSON helper used in scripts |

```bash
docker --version
oci --version
terraform --version
jq --version
```

---

## 2. Oracle Cloud account prep

### 2.1 Create a deployer user

1. Console → **Identity & Security → Users → Create User** (call it `wingspanai-deployer`).
2. Add that user to group `WingspanAiDeployers` and attach a policy:

```
Allow group WingspanAiDeployers to manage all-resources in compartment WingspanAi
```

### 2.2 Generate API keys & configure CLI

Configure OCI CLI and generate API keys:
```bash
oci setup config
```
This will:
- Create `~/.oci/config` with your OCIDs and region.
- Generate API key pair (`oci_api_key.pem` and `oci_api_key_public.pem`) in `~/.oci`.
Follow the CLI prompt to upload the public key to your Oracle Cloud user under **User Settings → API Keys**.

### 2.3 Log in to OCIR

```powershell
# In PowerShell:
$ns = oci os ns get | jq -r '.data'
docker login "$ns.ocir.io" --username "wingspanai/<user>" --password "<auth-token>"
```

For Bash/Linux/macOS:

```bash
export OCIR_NS=$(oci os ns get | jq -r '.data')
docker login $OCIR_NS.ocir.io --username "wingspanai/<user>" --password "<auth-token>"
```

---

## 3. GitHub secrets checklist

| Secret key | Value |
|------------|-------|
| `OCI_TENANCY_OCID` | Your tenancy OCID |
| `OCI_USER_OCID` | Deployer user OCID |
| `OCI_REGION` | `ap-sydney-1` |
| `OCI_FINGERPRINT` | From `~/.oci/config` |
| `OCI_PRIVATE_KEY` | Contents of `~/.oci/oci_api_key.pem` |
| `OCI_AUTH_TOKEN` | OCIR auth token |
| `OCI_NS` | Namespace value |
| `OCI_TENANCY_NAME` | Tenancy slug (e.g. wingspanai) |
| `OCI_USER_NAME` | User slug |

---

## 4. Sanity checks

```bash
oci iam region list | jq '.data[].name'
docker pull hello-world
docker tag hello-world $OCIR_NS.ocir.io/wingspanai/hello-world:test
docker push $OCIR_NS.ocir.io/wingspanai/hello-world:test
```

---

**Next step:** open `build-deploy.md` and start at *Scope / Step 4 – Build the image locally*.

*Last updated: 2025‑07‑13 (AWST)*
