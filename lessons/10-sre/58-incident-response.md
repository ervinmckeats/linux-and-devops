# Lesson 58. Incident response and on-call

## Goal
Handle a production incident calmly with clear roles and communication, and run an on-call rotation people can live with.

## What is an incident

An unplanned event that hurts users or puts them at risk: outage, degraded performance, data loss or exposure, security breach. Declare early — downgrading a false alarm is cheap, a late response is not.

### Severity levels (example)

| Sev | Impact | Response |
|---|---|---|
| SEV1 | core service down or data loss/breach for many users | page now, all hands, status page, leadership informed |
| SEV2 | major feature broken or severe degradation | page on-call, status page |
| SEV3 | minor feature broken, workaround exists | handle in working hours |
| SEV4 | cosmetic, no real user impact | ticket |

Write your own table with concrete examples for your product, so nobody debates severity during an outage.

## Roles

For anything above a small incident, split the work:

- **Incident Commander (IC)** — coordinates, decides, keeps the timeline. **Does not debug.** Asks "what do we know, what are we trying, who is doing it, when do we check back?"
- **Operations / subject-matter experts** — investigate and fix.
- **Communications lead** — status page, support, stakeholders, regular updates.
- **Scribe** — writes the timeline in the incident channel (often the IC early on).

One person can hold several roles in a small team, but the IC role must be explicit.

## The response loop

1. **Detect** — an alert, a customer, a colleague.
2. **Acknowledge** the page (so it does not escalate) and **declare** the incident: severity, IC, a dedicated channel (`#inc-2026-01-10-checkout`), a video bridge if needed.
3. **Assess impact** — who is affected, since when, how badly? Dashboards, SLOs, error logs.
4. **Mitigate first, root-cause later.** Stop the bleeding:
   - roll back the last deploy or config change (most incidents follow a change);
   - fail over to another region/replica;
   - scale up, shed load, rate-limit, disable a feature flag;
   - block a bad client or traffic source.
5. **Communicate** on a fixed cadence (every 30 min for SEV1, even if "no news").
6. **Resolve** — users are fine again; keep monitoring for a while.
7. **Follow up** — postmortem (lesson 59), action items with owners.

## Communication templates

Internal update:

```
[SEV2] Checkout errors — update 14:30 UTC
Impact: ~8% of checkout requests fail since 13:52. Other pages OK.
Current theory: payment provider timeouts after the 13:45 deploy of payments-api 2.3.0.
Actions: rolling back payments-api to 2.2.4 (Anna), checking provider status (Ben).
Next update: 15:00 UTC or sooner.
IC: Chris
```

Public status page:

```
Investigating — Some customers cannot complete checkout. We are working on a fix.
Next update in 30 minutes.
```

No blame, no guesses as facts, no internal jargon in public updates.

## Debugging under pressure

- Start from **what changed**: deploys, config, feature flags, infrastructure, traffic, dependencies, certificates, DNS, time (DST, leap events, month-end jobs).
- Use the methods you know: golden signals (lesson 55), USE (lesson 39), Kubernetes routine (lesson 54).
- Form a hypothesis, test it, write the result in the channel. Avoid five people changing things in parallel without telling each other.
- Prefer **reversible** actions. Record every action with a timestamp.
- If you are stuck for 15–30 minutes, escalate: bring in more people or the owning team.

## On-call

### Setup that works

- A rotation with at least 5–6 people (one week each is common), primary + secondary.
- Follow-the-sun across time zones when possible: nobody is paged at night routinely.
- Clear escalation policy: primary → secondary → team lead → service owner, with timeouts (for example 5 / 15 / 30 minutes).
- Handover at the end of each shift: open incidents, noisy alerts, risky changes coming.
- Compensation or time off for on-call. Burned-out on-call engineers make outages longer.

### Readiness checklist

- [ ] laptop, VPN, 2FA, access to prod tools tested **before** the shift
- [ ] pager app installed, notifications loud, tested
- [ ] runbooks and dashboards bookmarked
- [ ] knows how to declare an incident and who to escalate to
- [ ] knows how to roll back each service they own

### Healthy on-call metrics

- Pages per shift (target: a few, not dozens).
- Share of pages that were actionable (goal: almost all).
- Night-time pages.
- Time to acknowledge, time to mitigate (MTTA, MTTR).

Every non-actionable page gets a ticket: fix the alert or delete it.

## Practice

1. Write a severity table with three concrete examples per level for a product you know.
2. Run a **game day** in your lab: one person breaks something (stop the DB, fill a disk, deploy a bad image), another person is IC, another debugs. Use a chat channel and keep a timeline.
3. Write internal and public update messages for that incident.
4. Draft an on-call policy for a 6-person team: rotation, escalation, handover, compensation.

## Check yourself

- Why does the Incident Commander not debug?
- Mitigation or root cause first? Why?
- What is the first question to ask when something suddenly breaks?

## Next
[Lesson 59. Postmortems and runbooks](59-postmortems-runbooks.md)
