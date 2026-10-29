# Robot Shop — Availability SLO and Error Budget

## Purpose and scope
This document defines a **proposed** availability service-level objective (SLO) for the Robot Shop capstone deployment on Amazon EKS. It distinguishes the target from measurements that have actually been demonstrated. There is no external customer SLA.

## Service-level indicator (SLI)
**Synthetic availability**, measured by Prometheus Blackbox Exporter probes of the Robot Shop endpoints. For an agreed set of endpoints and a reporting period:

```text
Availability (%) = 100 × successful probe results / total valid probe results
```

A successful probe is a `probe_success` sample equal to `1`; an unsuccessful probe equals `0`. The indicator measures **probe-observed reachability**, not end-to-end checkout success or all real-user requests.

Scope: the five configured Blackbox probe targets (`monitoring/probe.yaml`). Keep the target set and scrape interval consistent when calculating a reporting period. If individual targets are weighted equally, pool their successful and total probe observations; exclude missing/invalid samples only with explicit documentation.

## Objective and window
- **Proposed SLO:** at least **99.0%** successful synthetic probes.
- **Measurement window:** rolling **30 days**.
- **Error budget:** at most **1.0% failed synthetic probes** in that window.
- **Budget in time-equivalent terms:** 30 days × 24 hours × 60 minutes × 1% = **432 minutes = 7 hours 12 minutes**. This is an illustration assuming continuous/evenly spaced observations, **not** an observed downtime value.

Example: for 100,000 valid probe results over 30 days, the error budget is 1,000 failed results. If 600 fail, 400 remain. These numbers are hypothetical, not measured production results.

## Prometheus queries
The existing recording rule in `monitoring/alert-rules.yaml` is:

```promql
robot_shop:availability:ratio_5m
```

It summarizes approximately five minutes of probe success and can support **short-term alerting**. It is **not** the 30-day SLO calculation.

A 30-day synthetic availability estimate using observed probe samples (subject to retention and a stable target set):

```promql
100 * sum(sum_over_time(probe_success{job="robot-shop-probe"}[30d]))
  / sum(count_over_time(probe_success{job="robot-shop-probe"}[30d]))
```

Failed-probe fraction over the same interval:

```promql
1 - (
  sum(sum_over_time(probe_success{job="robot-shop-probe"}[30d]))
    / sum(count_over_time(probe_success{job="robot-shop-probe"}[30d]))
)
```

These queries assume the label `job="robot-shop-probe"` matches the deployed Blackbox series. Verify in Prometheus before using the figures in a submission. A full 30-day assessment requires 30 days of retained probe data; a deployment that has run for only a few days cannot provide that history.

## Alert implementation and verification
`RobotShopAvailabilitySLOBreach` is configured to fire if the short-term availability recording rule falls below `0.99` for one minute. Alertmanager's Slack receiver was verified with a **manually submitted test alert**, including FIRING and RESOLVED notifications. An automatically evaluated Prometheus breach routed to Slack has **not** been conclusively demonstrated.

The short-term alert threshold is an operational signal; it does not directly equal the rolling 30-day error-budget policy.

## Error-budget response policy (proposal)
- **Under 25% budget consumed:** normal release and maintenance process.
- **25–50% consumed:** inspect failed probe targets, correlated application logs, and recent releases.
- **50–100% consumed:** prioritize reliability work and review nonessential changes.
- **Budget exhausted:** investigate recurrence, consider a temporary release pause, and document a recovery plan.

These thresholds are proposed operating guidance and are not automated by the current deployment.

## Known limits and future improvements
Synthetic probes cannot guarantee successful checkout or capture application-wide request latency. Add request-level HTTP success/latency instrumentation, validate the service targets, retain at least 30 days of metrics, and implement error-budget burn-rate alerts if required.

## References within this repository
- `monitoring/alert-rules.yaml`
- `monitoring/probe.yaml`
- `monitoring/dashboard.json`
- `monitoring/alertmanager-slack.yaml`
- `docs/runbook.md`
