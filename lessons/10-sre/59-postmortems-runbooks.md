# Lesson 59. Postmortems and runbooks

## Goal
Learn from every significant incident without blame, and turn hard-won knowledge into runbooks anyone on call can follow.

## Blameless postmortems

A **postmortem** is a written review of an incident: what happened, why, and what we change so it does not happen again (or hurts less).

**Blameless** means we assume people acted reasonably with the information and tools they had. *"Anna ran the wrong command"* is not a root cause; *"the tool allowed running a destructive command against prod without confirmation"* is. Blame makes people hide information, and hidden information means repeated incidents.

### When to write one

- Every SEV1 and SEV2.
- Any incident that used a big part of the error budget (lesson 57).
- Data loss, security incident, or a near miss that could have been serious.
- When someone asks for one.

Within about 5 working days, while memory is fresh.

### Template

```markdown
# Postmortem: Checkout failures after payments-api 2.3.0 deploy

Date: 2026-01-10    Severity: SEV2    Status: action items in progress
Authors: ...        Reviewers: ...

## Summary
For 47 minutes about 8% of checkout requests failed because payments-api 2.3.0
lowered the HTTP client timeout to the payment provider from 10 s to 1 s.

## Impact
- 13:52–14:39 UTC, ~3 100 failed checkouts, ~1 900 users affected
- 31% of the monthly error budget for checkout availability used
- Support received 140 tickets

## Timeline (UTC)
- 13:45 payments-api 2.3.0 deployed (canary skipped: config-only change)
- 13:52 error rate starts rising
- 14:04 SLO fast-burn alert pages on-call
- 14:12 incident declared, IC assigned
- 14:25 deploy identified as the trigger
- 14:31 rollback started
- 14:39 error rate back to normal; monitoring
- 15:10 incident resolved

## Root cause and contributing factors
- Timeout default changed in a shared HTTP client library upgrade.
- Provider p99 latency is ~1.4 s; 1 s timeout failed slow requests.
- Config-only changes bypassed the canary stage.
- No alert on payment provider timeout rate; detection relied on the SLO alert (12 min).

## What went well
- SLO alert fired within the expected time.
- Rollback was one command and took 8 minutes.

## What went wrong
- 20 minutes from page to identifying the deploy: deploy markers missing on the dashboard.

## Where we got lucky
- The incident happened outside the peak sales hour.

## Action items
| Action | Type | Owner | Due |
|---|---|---|---|
| Set explicit timeouts for provider calls, add test | prevent | team-payments | 2026-01-17 |
| Canary for all changes, including config | prevent | platform | 2026-01-31 |
| Deploy annotations on service dashboards | detect | sre | 2026-01-14 |
| Alert on provider timeout ratio | detect | team-payments | 2026-01-17 |
| Runbook: "checkout errors" with rollback steps | mitigate | sre | 2026-01-14 |
```

### Root cause analysis techniques

- **5 Whys**: ask "why?" repeatedly until you reach something you can change in the system. Stop at systems and processes, not people.
- **Contributing factors**: complex outages rarely have one cause. List all conditions that had to be true.
- **Timeline first**: reconstruct facts from logs, chat and dashboards before discussing causes.

### Action items that actually happen

- Specific, with an owner and a due date, tracked in the normal backlog.
- Classified: prevent / detect / mitigate / process.
- Prefer fixes that remove a whole class of failure (guardrails, automation) over "be more careful".
- Review open action items in a regular meeting. A postmortem with no completed actions is theatre.

Share postmortems widely. Reading other teams' postmortems is one of the fastest ways to learn production engineering.

## Runbooks

A **runbook** tells the on-call engineer what to do for a specific alert or situation. It is linked from the alert (`runbook_url`).

### Template

```markdown
# Runbook: HighErrorRate (checkout-api)

## What this alert means
More than 5% of checkout requests return 5xx for 10 minutes. Users cannot pay.

## Impact
Direct revenue loss. Usually SEV2; SEV1 if error rate > 50%.

## Dashboards and logs
- Service overview: <link>
- Logs: service="checkout-api" level="error" (last 30 min)
- Traces: filter status=error, route=/checkout

## Diagnosis
1. Was there a deploy or config change in the last hour?
   `kubectl -n shop rollout history deploy/checkout-api`, deploy annotations on the dashboard.
2. Which status codes and routes? (dashboard panel "errors by route")
3. Dependencies: payments-api latency/errors, database connections, Redis.
4. Pods healthy? `kubectl -n shop get pods -l app=checkout-api` (restarts, OOMKilled).

## Mitigation
- Bad deploy → `kubectl -n shop rollout undo deploy/checkout-api`, or revert the commit in the config repo (Argo CD syncs in ~1 min).
- Payment provider down → enable feature flag `payments_fallback_provider`.
- DB connections exhausted → see runbook "postgres-connections".

## Escalation
Team payments on-call via pager after 30 min without mitigation, or immediately for SEV1.

## History
- 2026-01-10: client timeout misconfiguration (postmortem link)
```

### Good runbooks

- Written for an on-call engineer who did not build the service.
- Copy-pasteable commands with real names, links to the exact dashboard.
- Updated after every incident where they were wrong or missing.
- Progressively automated: a runbook step done the same way every time is a candidate for a script, then for automatic remediation.

## Practice

1. Write a full postmortem for the game-day incident from lesson 58 using the template.
2. Do a 5 Whys analysis and find at least three contributing factors.
3. Write runbooks for two alerts from lesson 56 (high error rate, disk will fill) and link them in the alert annotations.
4. Ask someone else to follow one runbook without your help. Fix what confused them.

## Check yourself

- Why is "human error" never the root cause in a blameless postmortem?
- What makes a postmortem action item likely to be completed?
- What must a runbook contain to be useful during an incident?

---

<p align="left">
  <a href="media/59-postmortems-runbooks.mp4"><img width="640" src="media/59-postmortems-runbooks.gif" alt="Lesson 59 animation: Postmortems and runbooks"></a>
</p>

## Next
[Lesson 60. Reliability patterns and capacity planning](60-reliability-patterns.md)
