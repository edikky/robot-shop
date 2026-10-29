Robot Shop — Availability & Incident Response Runbook

Project: Capstone 2 — Observability on AWS EKS
Application namespace: robot-shop
Monitoring namespace: monitoring
Alert: RobotShopAvailabilitySLOBreach
Severity: page
Runbook owner: Robot Shop operations team

1. Purpose and trigger

This runbook describes how to investigate and recover from Robot Shop endpoint availability incidents using Prometheus, Grafana, Loki, Tempo, and Kubernetes.

The Prometheus recording rule robot_shop:availability:ratio_5m calculates the average success ratio of configured Blackbox Exporter probes over five minutes. The alert RobotShopAvailabilitySLOBreach becomes eligible to fire when this ratio is below 0.99 for one minute. It measures synthetic endpoint availability, not the success rate of every customer request.

The probes check the web entry point and selected application endpoints. Their target configuration is maintained in monitoring/probe.yaml.

2. Acknowledge and assess impact

Confirm the alert name, time, severity, and whether it is FIRING or RESOLVED in the configured Slack channel and Alertmanager.

Check Prometheus for the current five-minute availability ratio:

robot_shop:availability:ratio_5m

Identify which probes are failing:

probe_success{job="robot-shop-probe"}

Review the HTTP status and probe duration:

probe_http_status_code{job="robot-shop-probe"}
probe_duration_seconds{job="robot-shop-probe"}

Determine whether the failure affects only one endpoint or the web entry point as a whole. Note when the issue began and whether requests are returning 5xx errors.

Dashboard: Grafana dashboard Robot Shop — Requests, Errors, Latency. Its synthetic probe failure panel is not the same as the actual HTTP 5xx rate.

3. Kubernetes triage (WSL terminal)

kubectl get nodes
kubectl get deployments -n robot-shop
kubectl get pods -n robot-shop -o wide
kubectl get svc -n robot-shop
kubectl get events -n robot-shop --sort-by=.lastTimestamp | tail -30

For a service with failing pods, inspect its deployment and recent logs, replacing <service> with the affected deployment name:

kubectl describe deployment/<service> -n robot-shop
kubectl logs -n robot-shop deployment/<service> --since=15m --tail=150

Look for CrashLoopBackOff, OOMKilled, readiness probe failures, unavailable endpoints, failed image pulls, database connectivity issues, and application exceptions. Use kubectl describe pod <pod> -n robot-shop for pod-specific events.

4. Correlate logs and traces

Loki — Grafana Explore: Select the Loki data source. Find the affected web-container logs:

{namespace="robot-shop",container="web"}

Filter web errors:

{namespace="robot-shop",container="web"} |~ "\\\" 5[0-9][0-9] [0-9]+"

Use the time range around the incident. Inspect the URL, status code, and timestamp. Note that the existing Nginx access-log format includes HTTP status but does not record $request_time; HTTP latency cannot be calculated from those logs yet.

Tempo — Grafana Explore: Select Tempo and search for traces from instrumented services:

{ resource.service.name = "robot-shop-cart" }

{ resource.service.name = "robot-shop-catalogue" }

For a cross-service trace:

{ resource.service.name = "robot-shop-cart" } && { resource.service.name = "robot-shop-catalogue" }

Open a trace and inspect the waterfall, slow spans, and error tags. At present, Cart and Catalogue are instrumented; lack of spans from another service does not prove that service was healthy.

5. Recovery actions

Choose a recovery action only after identifying the cause. Do not restart the entire application reflexively.

Pod crash or memory exhaustion: Inspect events, memory limits, and logs. Repair the underlying configuration in Helm values/templates, then roll out the correction.

Readiness failure: Check health endpoint responses, startup behavior, resource limits, and dependent services.

5xx from a backend: Investigate the affected service and its dependencies; use Loki error logs and Tempo spans where available.

Problem following a Helm release: Review helm history robot-shop -n robot-shop. If the newest release is responsible and reverting is appropriate, roll back to a previously known-good revision: helm rollback robot-shop <revision> -n robot-shop --wait --timeout 5m.

The Robot Shop application is managed with Helm from K8s/helm; prefer making persistent fixes in source-controlled manifests. The load generator is separately managed from K8s/load-deployment.yaml.

6. Validate recovery

kubectl get deployments -n robot-shop
kubectl get pods -n robot-shop

Recheck in Prometheus:

probe_success{job="robot-shop-probe"}
robot_shop:availability:ratio_5m

Confirm the previously failing targets return probe_success = 1, service pods are ready, and the availability ratio recovers. Because the rule uses a five-minute window, the aggregate ratio may take time to return to 1. Confirm Alertmanager eventually marks the alert RESOLVED and Slack receives a resolution notification. Document the observed times rather than assuming recovery from a single successful request.

7. Communication and incident record

Record the following in the incident ticket or postmortem:

Start time, detection time, acknowledgement time, recovery time (with timezone).

Affected endpoints/services and observed customer impact.

Alert name and the values of the failing metrics.

Key logs, trace IDs, and deployment or pod events.

Root cause (or clearly state not yet established).

Mitigation actions taken, commands used, and final verification.

Follow-up actions, owners, and deadlines.

8. Controlled alert test notes

The capstone included a controlled alert sent directly to Alertmanager, labeled stage="capstone-test". Slack displayed both FIRING and RESOLVED notifications. This validates Alertmanager routing and Slack delivery; it does not, by itself, establish that the availability rule automatically fired during a real outage. A temporary Prometheus test rule was created and subsequently removed; the permanent robot-shop-slo rule remains installed. Do not confuse test evidence with an actual production incident.

9. Known limitations and security

Loki and Tempo use nonpersistent local storage for this lab; stored observability data may be lost when their pods are replaced.

Instrumentation currently covers Cart and Catalogue, not every microservice.

The standard Nginx Prometheus exporter measures aggregate request rate, not HTTP 5xx percentage or request-latency distribution.

Alertmanager's Slack webhook belongs in a Kubernetes Secret; never commit its URL or raw secret data to Git.

EKS worker nodes and associated AWS services may incur charges while running. Shut down resources through the appropriate AWS infrastructure workflow when the capstone environment is no longer needed.