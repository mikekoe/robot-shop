# Postmortem — [Service] Incident
**Date:** 2026-10-28
**Author:** Mike Adeyeye
**Status:** Resolved
**Severity:** [P1 / P2 / P3]

## Summary
One sentence describing what happened and its impact.

## Timeline (all times WAT)

| Time | Event |
|---|---|
| HH:MM | Incident begins / alert fires in Slack |
| HH:MM | On-call engineer acknowledges |
| HH:MM | Mitigation applied |
| HH:MM | Service restored |
| HH:MM | Root cause identified |
| HH:MM | Incident closed |

## Impact
- **Duration:** X minutes
- **Services affected:** catalogue, web, etc.
- **Users affected:** all users / subset
- **Error budget consumed:** X% of 30-day budget

## Root Cause
What actually caused the incident. Be specific.
Example: "The catalogue deployment was scaled to 0 replicas by
a misconfigured HPA reacting to a CPU spike during load testing."

## Contributing factors
- Factor 1
- Factor 2

## What went well
- Slack alert fired within 2 minutes
- Mitigation applied in under 5 minutes
- Logs in Loki helped narrow down the service quickly

## What went wrong
- No runbook for this failure mode
- Root cause took X minutes to identify

## Action items

| Action | Owner | Due |
|---|---|---|
| Add runbook for deployment scale-down | Mike | 2026-11-04 |
| Add HPA safeguards | Mike | 2026-11-04 |
| Add availability panel to dashboard | Mike | 2026-11-04 |

## Lessons learned
What would you do differently next time?
