# Postmortem: storefront unavailable during SLO drill

## Summary

On 29 October 2026 the `web` deployment in `robot-shop` was scaled to zero as a planned test
of the alerting path. The storefront was unreachable for about ten minutes. The blackbox
probe noticed the failure immediately, `RobotShopWebDown` reached Slack three minutes later,
and the alert resolved itself once the deployment was restored. There were no customers and
no data loss; this is a single-owner demo cluster.

## Impact

- Duration: roughly ten minutes, from the scale-down to the new pod passing readiness.
- Scope: the `web` Service had no endpoints, so every page of the storefront failed. Cart,
  catalogue and the other backends stayed healthy but were unreachable through the UI.
- Budget: ten minutes out of the 40,320 in a 28-day window is 0.025% of the period. Against
  a 1% availability budget that is about 2.5% of the month spent in one drill.

## Timeline (UTC, 29 October 2026)

| Time | Event |
|---|---|
| 18:11 | `kubectl scale deployment/web --replicas=0` |
| 18:11 | First failed probe within one scrape interval (30s) |
| 18:11 - 18:14 | Prometheus evaluates `avg_over_time(probe_success[2m]) == 0`, then waits for the `for: 1m` clause. Nothing is sent during this window. |
| 18:14:19 | Alert active in Prometheus (`activeAt` on `/api/v1/alerts`) |
| 18:15 | Slack receives `[FIRING] RobotShopWebDown`, including severity, summary and runbook link |
| ~18:20 | Deployment restored to one replica, new pod ready |
| ~18:24 | Slack receives `[RESOLVED]`, after the two-minute average recovers and `group_wait` elapses |

## Detection

What worked: blackbox probe to Prometheus, recording rules to alert, Alertmanager to Slack.
The full path took three minutes from failure to notification, and the message carried the
same `runbook_url` that is defined in `observability/slo-rules.yaml`.

What did not: nothing monitored the alerting path itself. Earlier the same day the operator
rejected the Alertmanager configuration (`undefined receiver "null" used in route`) and for
about twenty minutes every notification was dropped, while Prometheus, Grafana and the
dashboard all looked healthy. That failure was found by posting a test alert by hand, not by
an alert. A monitoring stack that cannot report its own failure is the gap this incident
exposed.

## Root cause

The outage was deliberate. The three-minute delay is not a fault, it is threshold design: two
minutes of averaging to avoid flapping, plus one minute of `for:` to avoid paging on a single
bad scrape. That design costs three minutes of blind time on a total outage, which is a
reasonable price for a 99% target and a poor one for a service with a tighter SLO.

Contributing factors found during the same session:

- `shipping` has been `0/1` for two hours. Its container is throttled at about 100% of a 200m
  CPU limit, so the Java process never finishes starting within the readiness window.
- `KubeSchedulerDown` and `KubeControllerManagerDown` fire permanently on EKS. Those
  components are managed by AWS and are not scrapeable from inside the cluster, so the alerts
  are always true and always useless.

## What went well

- The probe caught the failure at the first scrape, with no special-case code.
- The Slack message was actionable on its own: severity, what broke, what it means, runbook.
- The resolved notification arrived without manual intervention, which is what makes the
  channel trustworthy rather than something to mute.

## What did not

- Three minutes of detection latency on a complete outage, for a fault that is unambiguous.
- No signal on Alertmanager's own health, so a silent notification outage lasted forty minutes.
- A failing service (`shipping`) sat unnoticed behind a 15-minute `KubePodNotReady` threshold
  in a channel that also carries false criticals, which is how real alerts get ignored.

## Action items

| # | Action | Reason | Status |
|---|---|---|---|
| 1 | Disable `kubeScheduler`, `kubeControllerManager` and `kubeEtcd` monitoring in `observability/values-prometheus.yaml` | permanent false criticals on EKS | open |
| 2 | Alert on `Alertmanager` reconciling to `False`, or on `alertmanager_build_info` going stale | the twenty-minute silent gap | open |
| 3 | Lower `for:` on `RobotShopWebDown` from 1m to 30s | three-minute detection on total outage | open |
| 4 | Raise the `shipping` CPU limit above 200m in `K8s/helm/templates/shipping-deployment.yaml` | container throttled, never Ready | open |
| 5 | Add log aggregation (Loki or Fluent Bit) and request tracing | the SLO is blackbox only, with no view inside the request | deferred |

## Blameless note

Neither failure was an individual mistake. The stack was assembled without a health signal
for its own alerting path, and the upstream chart's 200m CPU limit for a Java service was
accepted without checking it against the startup profile. Both are structural: they are
addressed by the action items above, not by paying closer attention next time.

## Evidence

- `docs/drill.png`: Slack channel showing the synthetic test, the `[FIRING]` alert at 18:15
  and the `[RESOLVED]` message.
- `docs/grafana-dashboard.png`: the dashboard during normal operation.
- `observability/slo-rules.yaml`: the alert definitions and thresholds referenced above.
