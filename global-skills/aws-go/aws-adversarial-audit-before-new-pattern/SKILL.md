---
name: aws-adversarial-audit-before-new-pattern
description: >-
  Before implementing a NEW AWS pattern for the first time in a repo (first Step Functions, first
  Bedrock Guardrails, first EventBridge Pipes, first SES account, first service X), pause the
  implementation and run a time-boxed adversarial doc-research pass against current official docs —
  ~8-10 sharp questions covering quotas, IAM action naming, regional availability, eventual
  consistency, error/retry semantics, cost at scale, observability gaps, reversibility, and what
  re:Post reports as currently broken. AWS docs move quarterly and re:Post surfaces issues the main
  docs don't; ~45 minutes of research saves hours of mid-implementation rework. Triggers ONLY on
  "first" instances, not on refactors of patterns the repo already uses.
version: "1.0.0"
freshness:
  verified_against:
    - source: "AWS Well-Architected Framework (decision lens for AWS work)"
      url: "https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html"
      version: "Well-Architected (current)"
    - source: "AWS Service Authorization Reference (IAM actions per service)"
      url: "https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html"
      version: "Service Authorization Reference (current)"
    - source: "Klein — Performing a Project Premortem (HBR)"
      url: "https://hbr.org/2007/09/performing-a-project-premortem"
      version: "HBR 2007"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "AWS SDK Go v2 major + CDK CLI major"
    or_date: "2026-09-03"
  decay_risk: low
  status: current
---

# Adversarial audit before a new AWS pattern

The first time a repo adopts an AWS pattern, the failure modes are unknown and the docs are scattered.
AWS services carry non-obvious surprises — service quotas, regional GA gaps, IAM action-naming
inconsistencies, eventual-consistency windows, hard timeouts — documented across the main docs, the
[Service Authorization Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html),
and re:Post threads. This skill is a **pre-mortem** ([Klein, HBR 2007](https://hbr.org/2007/09/performing-a-project-premortem))
applied to AWS adoption: before writing code, spend ~45 minutes adversarially interrogating current
docs. The asymmetry is steep — discovering a blocker mid-implementation costs 10–100× discovering it
up front.

## When to invoke

Invoke **only** on a genuinely *first* instance in the repo:

- First Step Functions state machine. First Bedrock Guardrails config. First EventBridge Pipes. First
  SES account / sending domain. First use of any **service X** the codebase has never touched.

Do **not** invoke for: another Lambda that uses an integration the repo already has 50 of; a refactor
of a canonical, well-worn pattern. The marginal value collapses once the team has muscle memory.

**Announce on invoke:** "Using `aws-adversarial-audit-before-new-pattern` to run a time-boxed adversarial doc pass before standing up this first-of-its-kind AWS pattern."

## Discipline (not a brainstorm)

- **Time-box it** (~45 min). Without a box, the audit becomes the project.
- **Questions must be specific.** "What could go wrong?" yields mush. "List 3 service quotas this hits
  at 10× P99 traffic" yields action.
- **Verify against *current* docs**, not memory — AWS changes quarterly. A research-capable agent
  (WebFetch/WebSearch or AWS docs tooling) is effectively required.
- **Produce a written artifact** (an ADR). An audit nobody wrote down is forgotten by sprint+2.
- **Define "first" sharply.** "First state machine ever" → yes. "Another Lambda invoked by an existing
  state machine" → no.

## The audit checklist (a starting frame, not gospel)

Adapt per pattern; the value is in *answering against live docs*, not the exact list:

1. **Quotas/limits** — what service quotas could this hit at 10× current traffic?
   (`docs.aws.amazon.com/general/latest/gr/<service>.html`)
2. **IAM actions** — which actions does each call require? Any naming oddities (e.g. `s3:ListBucket`
   vs `s3:ListAllMyBuckets`)? (Service Authorization Reference)
3. **Regional availability** — is the feature GA in the deployment Region?
4. **Eventual consistency** — any eventually-consistent read on the happy path?
5. **Error/retry semantics** — for each call: typed error type, default retry, idempotency guarantee,
   any hard timeout (e.g. Cognito triggers' 5s synchronous cap).
6. **Cost at scale** — marginal cost per 1M invocations / per state transition / per GB.
7. **Observability** — which CloudWatch metrics ship by default; gap to the SLO dashboard.
8. **Reversibility** — how to roll back; any destructive op on the happy path.
9. **Cross-account/region** — what breaks if a second account/region is added later.
10. **Known issues** — search re:Post for the last ~90 days: what's actively broken right now.

End with a decision: **Proceed / Defer / Choose alternative**, recorded in `docs/adr/<NNNN>-<slug>.md`.

## Worked instance

A real audit return on EventBridge Pipes: question 9 ("fan-out?") surfaces that **a pipe routes events
from a single source to a single target** — point-to-point, not many-to-many. If the design assumed
fan-out, that's an *event bus*, not Pipes. Catching that in the 45-minute audit is the entire ROI;
discovering it after wiring Pipes is the rework this skill prevents.

## Anti-pattern to detect

- Standing up a first-of-its-kind AWS service straight from training-data memory, no current-doc pass.
- An unbounded "research spike" with no time-box and no written decision.
- Running this audit on a routine refactor (over-applying it dilutes the signal).

## Decision aid

- **Pattern is genuinely first in the repo?** → run the time-boxed audit, write the ADR.
- **Pattern already canonical here?** → skip; trust muscle memory.
- **Audit surfaces a blocker?** → Defer or pick the alternative *before* writing code.

## Related skills

- `global-skills/aws-go/verify-provider-api-supports-property/SKILL.md` — its sibling: this skill
  finds issues *before* you start; that one verifies the *claims* a finished design makes. Run this
  first, that one at design-review.
- `global-skills/aws-go/cdk-three-source-drift-check/SKILL.md` — audit whether the new resource type
  even supports drift detection as part of question 7 (observability).

## Sources

- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) · [Service Authorization Reference (IAM actions per service)](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html)
- [Klein — Performing a Project Premortem (HBR)](https://hbr.org/2007/09/performing-a-project-premortem) · [ADR community](https://adr.github.io/)
- [EventBridge Pipes (point-to-point, single source → single target)](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-pipes.html)

---

**Last verified:** 2026-06-03. The process is grounded in the AWS Well-Architected Framework and the
pre-mortem method (HBR 2007); the worked instance (EventBridge Pipes is point-to-point, not fan-out)
re-confirmed live against the EventBridge Pipes user guide.
**Re-check after:** AWS SDK Go v2 major / CDK CLI major, or by 2026-09-03. **Decay risk:** low (a
process pattern; only the cited example facts can drift).
**Found a drift?** Run `/skill-pattern-freshness-audit aws-go`.
