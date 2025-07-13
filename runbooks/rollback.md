# Rollback Playbook

This document describes the steps to roll back a Cal.com deployment on OCI.

## 1. Terraform Workspace Switch

```bash
cd iac
terraform workspace select <previous-env>  # e.g. dev, staging
terraform apply -var image_tag=<last-good-tag> -auto-approve
```

## 2. Load Balancer Backend Swap

- In OCI Console, navigate to the Load Balancer.
- Select the backend set and move traffic from the new backend to the previous one.
- Ensure health checks pass and sessions drain gracefully.

## 3. Post-Rollback Verification

- Check `/healthz` on the public LB IP.
- Run smoke tests against Cal.com endpoints.
- Confirm data integrity for bookings in Postgres.
- Validate Redis connectivity.

## 4. Incident Reporting

- Document timelines and actions in `runbooks/incident-<YYYY-MM-DD>.md`.
- Post-mortem reviews within 24 hours.
