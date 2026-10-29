# Capstone 2 — Evidence and final submission checklist

Save screenshots in a repository folder such as `docs/evidence/` **only after checking them for secrets**. Do not include sensitive Slack webhook tokens, cloud credentials, or private identifiers.

| Stage | Evidence to capture | Status based on conversation |
| --- | --- | --- |
| 1: Deployment | `kubectl get deployments -n robot-shop` showing all services available | Previously demonstrated; current screenshot recommended |
| 2: Monitoring | Grafana six-panel dashboard with readable panel names | Demonstrated; save screenshot |
| 2: Monitoring | Prometheus targets: Blackbox and application ServiceMonitors | Previously demonstrated; save screenshot |
| 2: Monitoring | `nginx_up = 1` and HTTP request-rate graph | Demonstrated |
| 3: Logs | Grafana Loki query `{namespace="robot-shop"}` | Demonstrated |
| 3: Logs | Expanded Nginx HTTP 500 payment-request log entry | Screenshot captured |
| 4: Traces | Grafana Tempo trace search containing Cart and Catalogue | Screenshot captured |
| 4: Traces | Trace waterfall: 2 services / 23 spans / ~13.55 ms | Screenshot captured |
| 5: Alerting | Alertmanager route showing `stage="capstone-test"` | Screenshot captured |
| 5: Alerting | Slack **FIRING** and **RESOLVED** messages | Screenshot captured |
| 5: Alerting | Automatic Prometheus rule reaches firing and Alertmanager | **Not conclusively verified** — do not claim complete |
| Documentation | `docs/runbook.md` committed | User confirmed completed |
| Documentation | Final README + evidence descriptions | Pending repository integration |
| Repository | Verify `git status --short` and remote commit | Pending final check |

## Suggested repository evidence filenames

```text
docs/evidence/
  01-robot-shop-deployments.png
  02-grafana-dashboard.png
  03-prometheus-targets.png
  04-loki-http-500.png
  05-tempo-cross-service-trace.png
  06-alertmanager-test-alert.png
  07-slack-firing-resolved.png
```

## Final verification (no full rerun required)

```bash
kubectl get deployments -n robot-shop
kubectl get pods -n monitoring
kubectl get prometheusrules -n monitoring | grep robot-shop
helm list -A
git status --short
```

## Submission caveats to explain plainly

1. Synthetic probes provide availability/failure and probe-duration measurements, not full application HTTP error/latency instrumentation.
2. Loki and Tempo use non-persistent local storage in this lab.
3. OpenTelemetry distributed tracing is demonstrated across Cart and Catalogue, not every service.
4. The Slack notification test used direct Alertmanager API injection; automatic Prometheus-to-Slack delivery remains unverified end to end.

## Optional follow-up for a stronger submission

For full alerting evidence, perform a safe, temporary Prometheus rule test using an alertname matched by the existing Slack routing, watch it reach FIRING in Prometheus, confirm receipt in Slack, and remove the test rule afterward. Avoid taking down Robot Shop. Capture the results before claiming end-to-end alert evaluation is verified.
