# Alerting Strategy

## Context & Problem

Metrics and logs are only useful if someone acts on them. Alerting bridges monitoring and response — notifying the right people when something needs attention. The challenge: too many alerts and people ignore them (alert fatigue). Too few and problems go undetected.

## Design Decisions

### Alert on Symptoms, Not Causes

**Symptom (good):** "API error rate > 5% for 5 minutes"
**Cause (bad):** "Database CPU > 80%"

Users experience symptoms. High database CPU might not cause any user-visible impact. High error rate always matters. Alert on what users feel, then use dashboards to diagnose the cause.

### Severity Levels

| Severity | Criteria | Response | Example |
|---|---|---|---|
| **P1 — Critical** | Revenue or data integrity at risk | Immediate page, wake people up | Trade execution failing, data loss |
| **P2 — High** | Degraded but functional | Respond within 1 hour | Elevated error rate, slow response times |
| **P3 — Warning** | Potential future problem | Investigate during business hours | Consumer lag growing, disk 80% full |
| **P4 — Info** | Notable but not actionable now | Review in weekly ops meeting | Dependency deprecated, certificate expiring in 30 days |

### Alert Design Rules

1. **Every alert must have a runbook** — what to check first, common causes, remediation steps
2. **Every alert must be actionable** — if there is nothing to do, it is not an alert, it is a log
3. **Alerts should fire rarely** — if an alert fires daily and is always ignored, delete it
4. **Group related alerts** — one incident, one notification (not 50 alerts for the same root cause)
5. **Include context in the alert** — metric value, threshold, affected service, link to dashboard

### Alert Examples

```yaml
# Prometheus alerting rules

groups:
  - name: api
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: p2
        annotations:
          summary: "API error rate above 5%"
          runbook: "https://wiki.internal/runbooks/high-error-rate"

      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2.0
        for: 5m
        labels:
          severity: p2
        annotations:
          summary: "P99 latency above 2 seconds"

  - name: kafka
    rules:
      - alert: ConsumerLagGrowing
        expr: |
          kafka_consumer_lag > 10000
        for: 10m
        labels:
          severity: p3
        annotations:
          summary: "Kafka consumer lag above 10K for {{ $labels.consumer_group }}"

  - name: data
    rules:
      - alert: MarketDataStale
        expr: |
          time() - market_data_last_update_timestamp > 300
        for: 5m
        labels:
          severity: p1
        annotations:
          summary: "Market data feed stale for 5+ minutes"
          runbook: "https://wiki.internal/runbooks/market-data-stale"
```

### SLO-Based Alerting

Instead of arbitrary thresholds, derive alerts from Service Level Objectives:

```
SLO: 99.9% of requests complete successfully within 500ms
Error budget: 0.1% of requests can fail per month

Alert: "Error budget burn rate > 10x normal"
→ At this rate, the monthly error budget will be exhausted in <3 days
```

This connects alerts directly to business impact rather than arbitrary numbers.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Alert fatigue | Too many noisy alerts | Review alert frequency monthly, delete unused alerts |
| Missing alert | New failure mode not covered | Post-incident review adds alerts for new failure modes |
| Flapping alert | Threshold at the edge of normal | Add hysteresis (`for: 5m`), widen threshold |
| Wrong recipient | Alert routed to wrong team | Alert routing by service label, review ownership quarterly |

## Related Documents

- [Metrics Design](metrics-design.md) — the metrics that alerts evaluate
- [Structured Logging](structured-logging.md) — diagnostic detail when alerts fire
- [Graceful Degradation](../resilience/graceful-degradation.md) — automated response to failures
