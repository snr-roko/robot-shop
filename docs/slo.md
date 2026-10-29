# SLOs and error budget: Stan's Robot Shop

## Service and user journey

The service under measurement is the storefront: `web` on port 8080, reached by users
through a browser. A user session is browse catalogue, add to cart, check out. Everything
behind `web` (catalogue, cart, user, payment, shipping, and the databases) matters only
because a failure there surfaces as a failed or slow page.

Applications in the cluster that no user touches are out of scope: `dispatch`, `ratings`
and the load generator are not part of any SLO.

## SLIs

robot-shop ships no `/metrics` endpoint. There is no Prometheus client library in any of
the twelve services, so request counts and error counts do not exist inside the app. The
SLIs are therefore measured from outside, with Prometheus's blackbox exporter probing the
`web` Service every 15 seconds.

| SLI | Definition | Source |
|---|---|---|
| Availability | proportion of probes that return an HTTP 2xx within the timeout | `probe_success{job="robot-shop-web"}` |
| Latency | proportion of successful probes that complete in under 500ms | `probe_duration_seconds{job="robot-shop-web"}` |

The probe checks `http://web.robot-shop.svc.cluster.local:8080/`, the same path a user
hits. It is a synthetic check: it exercises the frontend and whatever that request touches,
but it cannot prove that checkout works for a real user with a real cart. That limitation is
the reason the latency SLO exists alongside it.

## Objectives

| SLO | Target | Window |
|---|---|---|
| Availability | 99.0% of probes succeed | 28 days |
| Latency | 95% of successful probes complete under 500ms | 28 days |

99% over 28 days is roughly seven hours of allowed downtime per month. That is a
deliberately modest target for a demo cluster on three `t3.medium` nodes, where a node
roll or an eviction can take the storefront away for a minute without anything being
wrong with the application.

## Error budget

| SLO | Budget | In minutes (28 days) |
|---|---|---|
| Availability 99% | 1% of probes may fail | 403 minutes, about 6h43m |
| Latency 95% | 5% of probes may exceed 500ms | 2016 minutes, about 33h36m |

The budget is arithmetic, not storage. Prometheus keeps 6 hours of samples, so the
28-day budget cannot be recomputed from raw data. What the system actually evaluates is
the burn rate over short windows, which is what the alerts below do.

## Alerting

A static threshold ("page me when availability drops below 99%") is wrong for a 28-day
objective: at 99.9% measured over five minutes, a single failed probe looks like a page,
and at the start of the window you have spent almost none of the budget. The alerts are
therefore burn-rate based, using the standard multi-window pattern.

| Alert | Condition | Window | Meaning |
|---|---|---|---|
| `RobotShopAvailabilityBurnCritical` | failure ratio > 14.4x budget | 1h and 5m | the whole month's budget goes in about two days |
| `RobotShopAvailabilityBurnWarning` | failure ratio > 6x budget | 6h and 30m | budget is going faster than the window allows |
| `RobotShopWebDown` | all probes failing | 2m | the storefront is down, no ambiguity needed |

14.4 and 6 come from the SRE workbook: with a 1% budget, burning 2% in an hour is 14.4x,
and burning 5% in six hours is 6x. The `for:` duration keeps a single dropped probe from
paging anyone.

Each alert carries `runbook_url` pointing at `docs/runbook.md`, so the person receiving it
has a next step rather than a graph.

## What this does not cover

- **Per-service SLOs.** catalogue, cart and payment have no individual objectives. A slow
  `catalogue` shows up only as slow `web` probes.
- **Real user traffic.** The probe is synthetic. The load generator produces realistic
  traffic, but it is not measured per user and has no notion of success.
- **Satutation.** No queue depth, no connection pool, no database latency.
- **Long-term trends.** 6 hours of retention, so the dashboard shows hours, not weeks.

## Review

The objectives and the probe path are reviewed whenever the entry point changes (an
Ingress, a new frontend, a different port). The burn-rate thresholds do not change;
they are derived from the budget, not from observed traffic.
