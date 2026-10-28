# Error Budget Policy — Catalogue Service

## SLO Definition

| Property | Value |
|---|---|
| Service | catalogue (robot-shop) |
| SLI | Availability: successful HTTP requests / total HTTP requests |
| Success criteria | HTTP response code is not 5xx |
| SLO target | 99% over a 30-day rolling window |
| Error budget | 1% of requests (~7.2 hours per 30 days) |
| Measured by | Prometheus + kube-state-metrics |

## Error Budget Calculation
## Policy

### When error budget is > 50% remaining
- Normal operations. Ship features freely.
- Review SLO weekly in team standup.

### When error budget is 25–50% remaining
- Slow down feature releases.
- Prioritise reliability improvements in next sprint.
- Engineering lead notified.

### When error budget is < 25% remaining
- Feature freeze. Reliability work only.
- Incident review required for all 5xx errors.
- Daily error budget review until budget recovers.

### When error budget is fully exhausted (0%)
- Immediate incident declared.
- No feature releases until budget recovers above 25%.
- Post-mortem required within 48 hours.
- Escalate to engineering manager.

## Responsibilities

| Role | Responsibility |
|---|---|
| On-call engineer | Respond to alerts within 15 minutes |
| Engineering lead | Review budget weekly, approve freeze decisions |
| Product manager | Accept feature freeze when budget < 25% |

## Review cadence
This policy is reviewed quarterly or after any incident that
exhausts more than 20% of the error budget in a single event.

## Slack alerts
Alerts fire to #robot-shop-alerts via Alertmanager when:
- Fast burn: >5% budget consumed in 1 hour
- Slow burn: >10% budget consumed in 6 hours
- Critical: error budget exhausted
