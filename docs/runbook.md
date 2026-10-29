# Runbook: robot-shop storefront alerts

Every alert in `observability/slo-rules.yaml` links here through its `runbook_url`
annotation.

## RobotShopWebDown

All blackbox probes to `http://web.robot-shop.svc.cluster.local:8080/` have failed for two
minutes. This is the blunt one: the storefront is not answering.

1. Confirm it is not the prober:

```shell
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus-blackbox-exporter
kubectl port-forward -n robot-shop svc/web 8080:8080
```

If the blackbox exporter is down, or `curl http://localhost:8080` works while the probe
fails, the problem is in monitoring, not in the shop.

2. Look at the workloads:

```shell
kubectl get pods -n robot-shop
kubectl get events -n robot-shop --sort-by=.lastTimestamp | tail -20
```

3. Check the frontend itself:

```shell
kubectl describe pod -n robot-shop -l service=web
kubectl logs -n robot-shop deploy/web --tail=30
```

Common causes, in the order they have actually happened here:

| Symptom | Cause | Action |
|---|---|---|
| `web` pod `ImagePullBackOff` | bad image tag in a rollout | `helm rollback robot-shop -n robot-shop` |
| `web` pod `Running` but not `Ready` | readiness probe failing, app not listening | check `web` logs, usually a dependency it cannot reach |
| all pods fine, probe still failing | Service selector or port changed | `kubectl get endpoints -n robot-shop web` |
| pods `Pending` | node capacity or node NotReady | `kubectl get nodes`, `kubectl describe pod` for scheduling reason |
| `cart` failures cascading | `redis-0` restarting | `kubectl get pod -n robot-shop redis-0`, check PVC |

4. Verify recovery: the alert resolves when probes succeed again; that is what
`send_resolved` in Alertmanager reports.

## RobotShopAvailabilityBurnCritical / Warning

The service is answering but failing often enough to exhaust the budget ahead of schedule.

1. Quantify it before acting:

```shell
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Then in Prometheus, `avg_over_time(probe_success{job="robot-shop-web"}[5m])`. A value of
0.97 means 3% of probes failed in the last five minutes.

2. Separate frontend from backend. The probe only touches `web`, so a healthy `web` with
failing probes means `web` is returning errors from something downstream:

```shell
kubectl logs -n robot-shop deploy/web --tail=50
kubectl logs -n robot-shop deploy/cart --tail=50
kubectl logs -n robot-shop deploy/catalogue --tail=50
```

3. Check whether anything is being restarted or OOM-killed:

```shell
kubectl get pods -n robot-shop
```

This cluster has no metrics-server, so `kubectl top` is not available. Container memory
comes from the dashboard's working-set panel, or from Prometheus directly:

```
sum by (container) (container_memory_working_set_bytes{namespace="robot-shop", container!="", container!="POD"})
```

The chart sets tight memory limits (100Mi for most services, 1000Mi for `shipping`). A
pod near its limit will be OOM-killed and recovered by the Deployment, which shows up as
intermittent failures rather than a clean outage.

4. If a recent release is the likely cause:

```shell
helm history robot-shop -n robot-shop
helm rollback robot-shop -n robot-shop <previous-revision>
```

## RobotShopLatencyBurnWarning

The probe is still succeeding, but more than 10% of requests took longer than 500ms over the
last thirty minutes. With a blackbox probe this is the only latency signal there is: it
includes the frontend, everything it calls synchronously, and the network between them.

1. Establish which end moved. In the dashboard, compare the probe latency panel with the CPU
   and memory panels. Latency rising with CPU near a limit means queueing, not a slow
   dependency:

```shell
kubectl get pods -n robot-shop
kubectl describe pod -n robot-shop -l service=web | findstr -i "cpu limits throttl"
```

2. A container whose CPU is throttled will show `CPUThrottlingHigh` in Prometheus. `shipping`
   does this permanently on the default chart values; check whether it has spread:

```
sum by (pod) (rate(container_cpu_cfs_throttled_periods_total{namespace="robot-shop"}[5m]))
```

3. Check the shared Redis, which the cart calls on every page:

```shell
kubectl logs -n robot-shop redis-0 --tail=30
kubectl get endpoints -n robot-shop redis
```

4. If latency follows a recent rollout, treat it as a bad release:

```shell
helm history robot-shop -n robot-shop
helm rollback robot-shop -n robot-shop <previous-revision>
```

## Escalation

This is a single-owner demo cluster. There is no on-call rotation; the escalation path is
the repository owner. If the budget is exhausted, stop and write the postmortem before
changing anything further, because the sequence of events is the thing worth keeping.

## Related

- `docs/slo.md`, for the definitions and the budget arithmetic
- `docs/postmortem.md`, for the write-up after an incident
