# Monitoring Runbook

This document outlines the observability and alerting setup for the Cal.com OCI deployment.

## Load Balancer Health Checks

- HTTP probe path: `/healthz`
- Expected response: `200 OK`
- Configure in OCI Load Balancer health check settings.

Oracle Docs: https://docs.oracle.com/en-us/iaas/Content/Balance/Tasks/healthcheck.htm

## Container Readiness Probes

- Endpoint: `/api/status`
- Expected response: `200 OK`
- Configure as a custom HTTP probe in the backend set.

Oracle Docs: https://docs.oracle.com/en-us/iaas/Content/ContainerEngine/Concepts/healthchecks.htm

## Metrics & Alarms

| Metric                   | Threshold       | Alert Action             |
|--------------------------|-----------------|--------------------------|
| p95 HTTP latency         | ≥ 1s            | PagerDuty / Email Alert  |
| HTTP 5xx error rate      | ≥ 1%            | PagerDuty / Email Alert  |
| Redis connection errors  | ≥ 3 in 5 min    | PagerDuty / Slack Alert  |

Configure alarms in OCI Monitoring.
Oracle Docs: https://docs.oracle.com/en-us/iaas/Content/Monitoring/Concepts/monitoringoverview.htm
