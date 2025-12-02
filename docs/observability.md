# Observability Guide

This document outlines telemetry, alerting, and operational runbooks for the system.

## Telemetry
Capture the following metrics with Prometheus or OpenTelemetry exporters and tag them with `service`, `region`, and `environment` labels for slicing.

- **Request volume**: `http_requests_total` counter broken down by endpoint, HTTP method, and status class.
- **Error rate**: `http_requests_errors_total` counter and `http_request_error_ratio` calculated as errors/total over a 5m window; add specific counters for signature validation failures.
- **Latency**: `http_request_duration_seconds` histogram with P50/P90/P99 summaries.
- **Queue depth**: `message_queue_depth` gauge and `message_queue_age_seconds` histogram for enqueue-to-dequeue age.
- **Distributed tracing**: propagate W3C trace context; sample at least 5% of traffic, 100% for requests touching crypto/signature paths. Include spans for queue enqueue/dequeue and DB calls.

## Alerting
Recommended alert rules (PromQL-style examples):

- **Signature verification failures**: alert when `increase(signature_verification_failures_total[5m]) / increase(http_requests_total[5m]) > 0.02` for 10m.
- **Key rotation errors**: alert on `increase(key_rotation_failures_total[15m]) > 0` or when the latest rotation timestamp exceeds the expected schedule by 15m.
- **Queue saturation**: alert when `message_queue_depth > 0.8 * queue_capacity` for 10m or `message_queue_age_seconds_bucket` shows P99 age > target SLA.
- **Database latency**: alert when `histogram_quantile(0.95, rate(db_query_duration_seconds_bucket[5m])) > 250ms` for 10m.

All alerts should include: service name, environment, affected region, recent related logs, and runbook links.

## Runbooks

### Device re-key / key rotation
1. Validate the rotation schedule and target key ID.
2. Drain traffic from the instance if possible (remove from load balancer).
3. Trigger rotation via the control plane or CLI; record the operation ID.
4. Verify new key presence via status endpoint and ensure signing/verification succeeds.
5. Re-enable traffic and monitor signature error rate and key rotation error alerts for 30 minutes.
6. If issues persist, roll back to the previous key version and escalate to security.

### Agent log collection
1. Locate the agent host or container ID.
2. Collect recent logs: `journalctl -u agent.service -n 500` or `docker logs --tail 500 <agent>`.
3. Export structured logs with timestamps and request IDs; redact secrets.
4. Upload logs to the central log bucket with case/incident ID for correlation with traces.

### Release rollback
1. Identify the last known good version and corresponding artifact hash.
2. Halt deployments and drain traffic from faulty instances.
3. Roll back using the deployment tool (e.g., `helm rollback`, `kubectl rollout undo`, or CI/CD pipeline action).
4. Clear canary/feature flags that forced the new version.
5. Run smoke tests and monitor error rate, latency, and queue depth metrics for 30 minutes.
6. Once stable, re-enable deployments and document the incident.
