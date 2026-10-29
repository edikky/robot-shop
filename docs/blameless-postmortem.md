# Robot Shop — Blameless Incident Postmortem

**Incident:** Shipping service memory exhaustion during test load

**Status:** Resolved during capstone testing; documented retrospectively

**Environment:** Robot Shop on Amazon EKS, namespace `robot-shop`

**Severity:** Test-environment reliability incident; formal severity and customer impact were not measured

**Date and duration:** Occurred during the October 2026 capstone implementation. Precise start, recovery timestamps, and duration were not captured for this report.

## Executive summary
During load testing of the Robot Shop deployment, the Java-based Shipping service experienced a memory exhaustion / heap out-of-memory condition under a higher load-generator setting (15 clients). This exposed an insufficient memory allocation or JVM sizing for that workload. The team adjusted the Shipping Helm resource configuration and readiness behavior, redeployed, and subsequently ran the load generator at a lower, stable setting of five clients.

This was a capstone test incident, **not** evidence of production customer impact. The available records do not establish whether the same application workload would remain stable with 15 clients after the changes.

## Impact
- The Shipping service became unreliable during the higher-load test.
- The incident threatened shipping-dependent operations in the test application.
- The number of failed transactions, exact outage duration, and user impact were **not measured**.
- An HTTP 500 payment error was separately observed in Nginx/Loki logs. The available evidence does **not** establish that the payment error was caused by the Shipping memory incident; it should be investigated as a separate symptom.

## Timeline (sequence, not exact timestamps)
1. A higher load-generator setting (15 clients) was exercised against Robot Shop.
2. Shipping exhibited JVM heap out-of-memory behavior.
3. The team examined service health and adjusted the Helm configuration for Shipping.
4. Shipping resource requests/limits were updated and readiness timing was increased.
5. The Helm release was upgraded; the Shipping deployment returned to a healthy state.
6. A separate load deployment was used with five clients and a 512 MiB memory limit; this setting remained stable in subsequent checks.

Exact timestamps and log extracts should be added if retrievable from retained Kubernetes events or logs. Do not invent timings.

## Contributing factors and technical analysis
The observed failure mode was JVM memory exhaustion under the 15-client test. A plausible contributing factor was the mismatch between available container/JVM memory and peak workload demand. The evidence supports the need for additional memory headroom; it does **not** establish the precise internal JVM heap setting, memory leak, or single definitive root cause.

Relevant applied Shipping configuration:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 768Mi
  limits:
    cpu: 500m
    memory: 1536Mi
```

The Shipping readiness configuration was also relaxed to allow more startup time. Check the current chart values for the exact readiness fields.

## Detection and diagnosis
- Kubernetes deployment/pod health checks and application logs identified instability.
- Prometheus/Grafana were available for infrastructure and synthetic health investigation.
- Grafana Loki/Alloy supported centralized application log inspection.
- Tempo instrumentation demonstrated Cart-to-Catalogue traces, **not** a Shipping trace for this incident.
- The later Slack FIRING and RESOLVED messages came from a **manual Alertmanager test**; they should not be presented as an automatic alert from this incident.

## Response and recovery
The service configuration was updated through the Helm chart, and the deployment was brought back to a healthy state. The load profile was reduced to five clients for stable demonstration. This recovery proves operation under the later test conditions; it does not prove the original 15-client scenario was fixed.

## What went well
- Kubernetes and Helm allowed repeatable configuration changes and health verification.
- Basic logs and metrics exposed actionable stability issues.
- The incident led to practical resource and readiness tuning.

## What could improve
- Capture exact incident timestamps, pod termination reasons, relevant JVM logs, and load-test request/latency metrics.
- Correlate load generation with Shipping memory usage and failure rates.
- Add JVM heap and GC telemetry and application-level error/latency metrics.
- Test recovery at the **same** 15-client load before claiming full remediation.
- Verify an automatic Prometheus alert through Alertmanager to Slack in a controlled drill.

## Follow-up actions
| Action | Priority | Status |
|---|---|---|
| Increase Shipping memory request/limit and readiness allowance | High | Completed |
| Validate healthy Shipping deployment after Helm upgrade | High | Completed |
| Run stable demo load at five clients | High | Completed |
| Re-test 15-client workload with captured memory and error metrics | High | Open |
| Add JVM memory/GC dashboards and app-level HTTP error/latency metrics | Medium | Open |
| Capture repeatable incident drill evidence and exact timeline | Medium | Open |

## Lessons learned
The goal is not to attribute fault to an individual. Load tests are valuable because they reveal capacity assumptions before production deployment. A resource adjustment is a useful mitigation, but it should be validated against the original failure conditions before being called a complete fix.

## Related materials
- `K8s/helm/` — deployment chart and resource configuration
- `K8s/load-deployment.yaml` — load generator
- `docs/runbook.md` — investigation and recovery procedures
- `monitoring/dashboard.json` — monitoring dashboard
- `logging/` and `tracing/` — telemetry configurations
