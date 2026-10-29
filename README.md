# Robot Shop — Capstone 2: Kubernetes Observability

## Overview

This capstone deploys the Robot Shop microservices application to Amazon EKS and adds a practical observability stack for metrics, availability checks, centralized logs, distributed traces, and incident notifications.

> **Project status:** Core deployment, metrics collection, centralized logging, two-service distributed tracing, and Alertmanager-to-Slack delivery have been demonstrated. The limitations under **Verification boundaries** are part of the project record.

## Architecture

```text
                                Amazon EKS
┌────────────────────────────────────────────────────────────────┐
│ robot-shop namespace                                           │
│  Web (Nginx) ──> Cart ──> Catalogue                             │
│       │             │           │                              │
│       └────> other Robot Shop services                         │
│                                                                │
│ monitoring namespace                                           │
│  Blackbox Exporter ────────────────> Prometheus ──> Grafana     │
│  Nginx Exporter ───────────────────> Prometheus                 │
│  Cart / Payment /metrics ──────────> Prometheus                 │
│  Alloy <── Kubernetes pod logs ──> Loki ──────────> Grafana     │
│  Cart + Catalogue (OpenTelemetry) ──> Tempo ──────> Grafana     │
│  Prometheus alerts ──> Alertmanager ──> Slack                  │
└────────────────────────────────────────────────────────────────┘
```

The diagram shows the main monitoring paths. It is not an exhaustive diagram of every Robot Shop service interaction.

## Technology stack

| Area | Technology | Purpose |
| --- | --- | --- |
| Orchestration | Amazon EKS, Kubernetes, Helm | Application deployment |
| Metrics | kube-prometheus-stack | Prometheus, Grafana, Alertmanager |
| Availability | Prometheus Blackbox Exporter | HTTP endpoint probing |
| Web throughput | Nginx Prometheus Exporter | Nginx request and connection counters |
| Application metrics | Cart and Payment `/metrics` | Business activity counters |
| Logging | Grafana Alloy + Loki | Centralized container logs |
| Tracing | OpenTelemetry + Grafana Tempo | Tracing selected microservices |
| Alerting | PrometheusRule + Alertmanager + Slack | Availability alert routing and notification |

## Repository components

| Path | Contents |
| --- | --- |
| `K8s/helm/` | Robot Shop Helm chart and service configurations |
| `K8s/load-deployment.yaml` | Load-generator deployment |
| `monitoring/` | Probe, ServiceMonitor, Nginx exporter, dashboard, and alert configuration |
| `logging/loki-values.yaml` | Loki Helm values |
| `logging/alloy-values.yaml` | Alloy Kubernetes log collection settings |
| `tracing/tempo-values.yaml` | Tempo Helm values |
| `cart/Dockerfile.otel` | Instrumented Cart container build |
| `catalogue/Dockerfile.otel` | Instrumented Catalogue container build |
| `docs/runbook.md` | Incident response instructions |

## Implementation summary

### Stage 1 — Application deployment

Robot Shop was deployed to the `robot-shop` namespace via Helm. A separate load-generator deployment generates application requests. The team tuned Shipping readiness/resources and adjusted load-generator resource limits after encountering restarts under load.

### Stage 2 — Metrics and dashboards

Prometheus and Grafana were deployed using kube-prometheus-stack. Blackbox Exporter checks web application endpoints. ServiceMonitor scraping collects Cart and Payment metrics. An Nginx Prometheus Exporter provides `nginx_http_requests_total`, used to calculate web request throughput with:

```promql
sum(rate(nginx_http_requests_total[5m]))
```

The Grafana dashboard includes synthetic probe activity, synthetic probe failure rate, probe duration, Cart items added per second, units sold per second, and Nginx HTTP requests per second.

### Stage 3 — Centralized logging

Loki and Alloy collect Kubernetes application logs. In Grafana Explore, Robot Shop logs can be queried using LogQL:

```logql
{namespace="robot-shop"}
```

Example filter for Nginx HTTP 500 responses:

```logql
{namespace="robot-shop",container="web"} |= " 500 "
```

An HTTP 500 payment request was located in Grafana, demonstrating centralized error investigation.

### Stage 4 — Distributed tracing

Cart and Catalogue use custom OpenTelemetry-enabled images published as:

- `edikky/robot-shop-cart:otel-v1`
- `edikky/robot-shop-catalogue:otel-v1`

Traces are exported to Grafana Tempo through OTLP/HTTP. Grafana TraceQL can locate traces from either service and traces containing both:

```traceql
{ resource.service.name = "robot-shop-cart" } &&
{ resource.service.name = "robot-shop-catalogue" }
```

A Cart request trace showed **two services**, **23 spans**, and approximately **13.55 ms** duration in the captured evidence.

### Stage 5 — Alerting and response

The `robot-shop-slo` PrometheusRule monitors five-minute Blackbox probe availability and defines `RobotShopAvailabilitySLOBreach` for sustained availability below 99%. Alertmanager includes a `robot-shop-slack` route using a webhook Secret.

A controlled alert submitted **directly to Alertmanager** generated both **FIRING** and **RESOLVED** Slack messages. See `docs/runbook.md` for response procedures.

## Verification boundaries

- The synthetic error-rate and latency panels report **Blackbox probes**, not all production HTTP requests or a full request-latency histogram.
- Nginx Exporter confirms **aggregate request throughput**; it does not supply HTTP status-code breakdowns or HTTP request-duration histograms.
- A LogQL-based HTTP 5xx view can be built from access logs. The current Nginx access-log format does **not** include `$request_time`, so full HTTP request latency is not collected from those logs.
- The alert **routing and Slack delivery** test passed. A complete **Prometheus-rule-evaluation-to-Slack** test was **not conclusively evidenced**. The temporary test rule was removed; the permanent `robot-shop-slo` rule remains.
- Loki and Tempo were configured with **temporary filesystem storage** for this lab. Logs and traces may be lost when their pods are replaced; this is not a production retention design.
- Cart and Catalogue are instrumented. This is **not full distributed tracing of every Robot Shop service**.

## Operations and verification

Run these commands with your EKS Kubernetes context configured:

```bash
kubectl get deployments -n robot-shop
kubectl get pods -n monitoring
helm list -A
kubectl get prometheusrules -n monitoring
```

To access the Grafana UI locally:

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3001:80
```

To access Prometheus locally, use the service name exposed by your monitoring release and forward its port 9090. To access Alertmanager locally:

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093:9093
```

Refer to `docs/runbook.md` for troubleshooting, recovery, rollback, and incident documentation.

## Security and cost notes

- Never commit Slack webhook URLs, credentials, AWS keys, or kubeconfig files. Keep alert notification secrets in Kubernetes Secrets and do not print their contents in screenshots.
- Local port-forward URLs (`localhost`) are for the operator's machine, not public application endpoints.
- The EKS nodes and other AWS resources may continue accruing charges after local terminals are closed. Shut down infrastructure when it is no longer needed, following your course requirements.

## Evidence

See `docs/evidence-checklist.md` for the screenshots and verification commands required for a defensible submission.
