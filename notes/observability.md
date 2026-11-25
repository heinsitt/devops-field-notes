# Observability

## PromQL worth memorising

Request rate, over a real window:

```promql
sum by (job) (rate(http_requests_total[5m]))
```

Error ratio as a fraction, not a count, so it compares across services:

```promql
sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
  /
sum by (job) (rate(http_requests_total[5m]))
```

Latency from a histogram, because averages hide the people who are suffering:

```promql
histogram_quantile(0.95,
  sum by (le, job) (rate(http_request_duration_seconds_bucket[5m])))
```

`rate` needs a counter and at least two samples in the window. If a graph is empty
after a restart, that is why: give it one full window.

## Alert on symptoms, not causes

Page on what a user would notice: error ratio above a threshold for five minutes,
or p95 latency past the budget. Do not page on CPU at 80 percent; a busy machine
that serves fast requests is a working machine.

```yaml
- alert: ApiErrorRateHigh
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m]))
      / sum(rate(http_requests_total[5m])) > 0.05
  for: 10m
  labels:
    severity: page
  annotations:
    summary: "More than 5% of API requests are failing"
```

`for` is the difference between an alert and a notification: it swallows the single
bad minute that would otherwise wake someone up.

## Dashboards that answer a question

Four panels, in this order: is it up, how much traffic, how fast, how many errors.
Anything else goes below the fold. A dashboard nobody can read at 3am is
decoration.

## The three signals to keep

- **Metrics** for trends and alerting: cheap, aggregated, no identities.
- **Logs** for the one request that failed: expensive, keep them short and structured.
- **Traces** for where the time went across services: sample hard, keep the errors.

If a system has only logs, every incident starts with a grep and a guess.
