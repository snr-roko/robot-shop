# Capstone 2: observability for Stan's Robot Shop on EKS

This document is the entry point for the observability work. It describes what is deployed,
how each piece is configured, what was verified, and what is missing.

## What is running

- EKS 1.37, three `t3.medium` nodes, `robot-shop` namespace.
- Stan's Robot Shop installed from the chart in `K8s/helm`, with the Instana instrumentation
  removed. Redis uses a `gp2` volume so the cart state survives a pod restart.
- A load generator (`K8s/load-deployment.yaml`) sending 15 concurrent clients at the
  storefront, so the probe and the dashboards have something to measure.

## Monitoring stack

`kube-prometheus-stack` 92.16.0 installed as release `monitoring` in namespace `monitoring`:
Prometheus, Alertmanager, Grafana, kube-state-metrics and node-exporter on every node.

The stack does not ship a blackbox exporter, so it is a separate release (`blackbox`) from
`prometheus-community/prometheus-blackbox-exporter`. Robot Shop's services expose no
`/metrics` endpoint, which is why the SLO is measured from outside the application through
an HTTP probe against the storefront Service rather than from application counters.

Configuration lives in three files:

| File | Purpose |
|---|---|
| `observability/values-prometheus.yaml` | Prometheus retention and selectors, Alertmanager receivers, Grafana resources and dashboard sidecar |
| `observability/values-blackbox.yaml` | Blackbox exporter release, pinned service name |
| `observability/probe-web.yaml` | `Probe` resource: 15s interval, `http_2xx` module, targets the `web` Service |

Two non-default settings matter:

- `probeSelectorNilUsesHelmValues: false` and `ruleSelectorNilUsesHelmValues: false` let the
  Prometheus instance pick up the hand-written `Probe` and `PrometheusRule` objects. Without
  them the operator only selects objects carrying the chart's release label, and the probe
  and SLO rules are silently ignored.
- Grafana's admin password comes from the `grafana-admin` secret. No credential is stored in
  this repository.

## SLI, SLO and alerts

`docs/slo.md` defines the targets and the arithmetic: 99% availability over 28 days, and 95%
of requests faster than 500ms, also over 28 days.

`observability/slo-rules.yaml` implements them:

- Recording rules for the 1h and 6h availability ratios and the 30m latency ratio.
- `RobotShopWebDown`: every probe failed for two minutes, `for: 1m`.
- `RobotShopAvailabilityBurnCritical`: 14.4x burn over both the 5m and 1h windows.
- `RobotShopAvailabilityBurnWarning`: 6x burn over the 30m and 6h windows.
- `RobotShopLatencyBurnWarning`: more than 10% of probes slower than 500ms over 30m.

Every alert carries a `runbook_url` pointing at the matching section of `docs/runbook.md`.

## Alerting to Slack

Alertmanager reads the webhook from a mounted Secret rather than from the values file:
`api_url_file: /etc/alertmanager/secrets/alertmanager-slack/webhook-url`. The Secret is
created out of band and is not in this repository.

Routing sends the two meta alerts (`Watchdog`, `InfoInhibitor`) to a receiver with no
integration configs. They exist to prove the pipeline is alive; sending them to a human
channel every four hours is how alert channels lose their audience.

## Dashboard

`observability/dashboard-robot-shop.json` is the dashboard source of truth, and
`observability/dashboard-configmap.yaml` wraps it in a ConfigMap labelled
`grafana_dashboard: "1"` so the Grafana sidecar provisions it on load. Nothing was built by
hand in the UI.

Panels: probe success over 5m, budget consumed in the last hour, pods ready, probe p95
latency, probe success against the 99% line, probe p50/p95 over time, CPU and memory by
container, container restarts, and not-ready pods. Screenshot: `docs/grafana-dashboard.png`.

The budget panel is worth explaining: dividing the failure ratio by the 1% budget converts
it into "percent of the monthly budget this hour cost you". A short outage reads near zero;
a full hour down reads 100.

## What was verified

| Check | Result |
|---|---|
| `probe_success{job="robot-shop-web"}` = 1 | Confirmed in Prometheus |
| `Probe` and `PrometheusRule` loaded without label selectors | Confirmed |
| Dashboard provisioned from ConfigMap | Sidecar log: `Writing /tmp/dashboards/robot-shop.json` |
| Alertmanager accepts the rendered config | `Reconciled: True` |
| Alert reaches Slack | Synthetic alert, then a real one during the drill |
| Alert includes the runbook link | Visible in `docs/drill.png` |

## Incident drill

`web` was scaled to zero and left down for roughly ten minutes. Prometheus marked
`RobotShopWebDown` active at 18:14:19, Slack received the firing message at 18:15, and the
alert resolved after the deployment was restored. The full write-up is in
`docs/postmortem.md`; the channel is captured in `docs/drill.png`.

Two real failures happened during the build and are documented in the same place: Grafana was
OOM-killed by a 256Mi limit that was too small for a 13.x container with provisioning
sidecars, and the operator rejected the Alertmanager config for about twenty minutes because a
Helm merge left a route pointing at a receiver that no longer existed.

## Requirements

| Requirement | Status |
|---|---|
| Deploy the application and generate load | Done |
| Metrics with kube-prometheus-stack | Done |
| Dashboard exported as code | Done |
| SLO document with error budget | Done |
| Alert that fires and reaches Slack | Done, verified in the drill |
| Runbook linked from the alert | Done |
| Centralised logs searchable by service and level | Not implemented |
| Distributed tracing of a single request | Not implemented |

## Known limitations

- The SLO measures the storefront over HTTP from outside. It says nothing about a slow
  backend that `web` masks, and nothing about a request that fails after the first byte.
- No log aggregation or tracing, so an incident cannot be followed into a specific request.
  This is the largest gap, and it is why the runbook sends the reader to `kubectl logs`
  rather than to a query.
- Prometheus keeps six hours of data in an `emptyDir`. Restarting the pod loses history, and
  the 28-day SLO is only computable while the process stays up. A PVC and a longer retention
  would fix it, at the cost of storage.
- Images are `robotshop/rs-*:latest` with no digest pinning. The same chart can deploy
  different code a week apart.
- `shipping` never becomes Ready: it is CPU-throttled against a 200m limit inherited from the
  upstream chart. It does not affect the storefront SLO but it is visible in the dashboards.
- `KubeSchedulerDown` and `KubeControllerManagerDown` fire permanently on EKS, because those
  components are managed by AWS. They are false criticals and should be disabled.
- Alertmanager and Prometheus run as single replicas without anti-affinity. A node failure
  takes monitoring down with it.

## Reproducing

```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring \
  --create-namespace -f observability/values-prometheus.yaml
helm install blackbox prometheus-community/prometheus-blackbox-exporter -n monitoring \
  -f observability/values-blackbox.yaml
kubectl apply -f observability/probe-web.yaml
kubectl apply -f observability/slo-rules.yaml
kubectl apply -f observability/dashboard-configmap.yaml
```

The Alertmanager receiver needs the Slack Secret first, or the pod will not start:

```shell
kubectl create secret generic alertmanager-slack -n monitoring \
  --from-literal=webhook-url=<incoming webhook>
```
