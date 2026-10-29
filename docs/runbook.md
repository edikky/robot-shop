# Runbook: RobotShopAvailabilitySLOBreach

**Alert:** Robot Shop availability fell below the 99% SLO (5 minute window).
**Impact:** Customers may not be able to load the shop or use the catalogue, cart or payment.

## 1. Confirm the problem (2 minutes)
- Grafana dashboard "Robot Shop - Requests, Errors, Latency": which panel changed, and when did it start?
- `kubectl get pods -n robot-shop` : look for pods that are not Running, or that keep restarting.

## 2. Mitigate first
- If a pod is crashing: `kubectl rollout restart deploy/<name> -n robot-shop`
- If a deployment has no replicas: `kubectl scale deploy/<name> -n robot-shop --replicas=1`
- If the problem began right after a release: `helm rollback robot-shop <previous-revision> -n robot-shop`

## 3. Find the cause
- `kubectl describe pod <pod> -n robot-shop` : read the Events at the bottom.
- `kubectl logs deploy/<name> -n robot-shop --tail=50`
- Check the data stores (mongodb, mysql, redis, rabbitmq) are Running.

## 4. Confirm recovery
- Availability on the dashboard returns to 100%.
- The alert shows RESOLVED in Slack (this can take up to 5 minutes after the fix).

## 5. Afterwards
Write a blameless postmortem: timeline, impact, root cause, what to improve.