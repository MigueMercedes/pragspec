# Runbooks

Operational playbooks for things that go wrong in production. Each runbook is a single Markdown file describing **how to detect, diagnose, and recover** from a specific failure mode.

Runbooks are tied to **alerts and operational extensions**. A spec with the `operational` extension enabled should reference (or create) the relevant runbook. New alerts without a runbook are a smell — by the time a human is paged, the answer should be one search away.

## Suggested filename

```
docs/runbooks/<area>-<failure-mode>.md
```

Examples:
- `docs/runbooks/billing-stripe-webhook-stuck.md`
- `docs/runbooks/auth-rate-limit-exhausted.md`
- `docs/runbooks/db-replica-lag-spike.md`

## Suggested structure

```markdown
# <Area>: <failure mode in 3-6 words>

> **Last updated**: 2026-04-28 · **Owner**: <team or person> · **Severity**: SEV1 | SEV2 | SEV3

## Detect

How does this manifest? What alert fires? What dashboard shows it? Concrete signals — not symptoms a user might describe.

## Diagnose

The decision tree. What logs / queries / commands narrow down the cause? Order them from cheapest to most expensive.

## Recover

The steps to restore service. Prefer reversible mitigation first (kill switch, scale up, rollback) over root-cause fixes during the incident.

## Postmortem

After the incident: link the postmortem doc, the related ADR if a pattern emerged, and any spec changes the incident triggered.
```

Keep runbooks short — 50-150 lines. If a runbook grows past that, the failure mode is probably two failure modes pretending to be one; split it.
