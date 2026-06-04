---
name: verify-provider-api-supports-property
description: >-
  Before a design doc promises an architectural property — reversibility, idempotency, exactly-once,
  cross-service atomicity, mechanical compensation, a sub-Xms SLA — verify the concrete AWS SDK method
  that mechanically guarantees it actually exists and supports the claim. "The Saga from the book" is
  not the same as "a Saga implementable on Cognito + Step Functions." AWS's own Saga docs admit the
  pattern gives eventual consistency, not ACID. For each promised property, demand the specific AWS
  API that backs it; if none exists, the property is NOT the system's and must be documented as "not
  mechanically guaranteed" or the design must change. Use at design review, whenever a doc claims a
  strong distributed-systems guarantee.
version: "1.0.0"
freshness:
  verified_against:
    - source: "AWS Prescriptive Guidance — Saga orchestration (eventual consistency, not ACID)"
      url: "https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html"
      version: "Prescriptive Guidance (current)"
    - source: "AWS — DynamoDB TransactWriteItems (intra-DynamoDB atomicity only)"
      url: "https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_TransactWriteItems.html"
      version: "DynamoDB API (current)"
    - source: "AWS — Cognito Lambda trigger limits (hard 5s synchronous timeout)"
      url: "https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-working-with-lambda-triggers.html"
      version: "Cognito Developer Guide (current)"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "AWS SDK Go v2 major + CDK CLI major"
    or_date: "2026-09-03"
  decay_risk: low
  status: current
---

# Verify the provider API supports the promised property

A design doc may *claim* exactly-once, atomicity, or a tight SLA. A claim is only real if a concrete
provider API **mechanically** delivers it. The discipline: for every promised property, name the
specific AWS SDK method (or service guarantee) that backs it. If you can't, the property is **not** a
property of the system — downgrade it in the doc to "best-effort, not mechanically guaranteed," or
change the design.

AWS itself draws this line. Its
[Saga orchestration guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html)
states, verbatim, in "Issues and considerations":

> "**Eventual consistency**: The sequential processing of local transactions results in eventual
> consistency …"
> "**Idempotency**: Saga participants need to be idempotent …"
> "**Transaction isolation**: Saga lacks transaction isolation. Concurrent orchestration of
> transactions can lead to stale data."

So "Saga from the book" gives **eventual** consistency. If a doc claims "exactly-once cross-service
atomicity" and the implementation is Step Functions + Lambda, the doc is wrong.

## When to invoke

- At **design review** of any doc that asserts a strong distributed-systems guarantee:
  reversibility, idempotency, exactly-once, cross-service atomicity, mechanical compensation, SLA < X ms.
- When an implementer reaches for an SDK method to satisfy a promised property — check the method
  actually supports it *in that context*.

**Announce on invoke:** "Using `verify-provider-api-supports-property` to bind each promised guarantee to the concrete AWS method that backs it — or downgrade the claim."

## The verification table (the artifact)

For each promised property, fill a row. A property with no "YES" row is not the system's.

| Property promised | Backing AWS API / service | Mechanically guaranteed? | Note |
|---|---|---|---|
| Atomic write across 2 DynamoDB **tables** | `dynamodb:TransactWriteItems` | **YES** — all-or-nothing within DynamoDB | tables must be **same account + Region**; ≤100 actions, ≤4 MB |
| Atomic write across DynamoDB **+ S3** | *(none)* | **NO** — must use Saga + idempotency | no native cross-service atomic primitive |
| Idempotent SQS processing | `sqs:ReceiveMessage` + a DynamoDB conditional `PutItem` | **YES** — you implement it | the conditional write *is* the guarantee |
| Idempotent transactional write | `TransactWriteItems` + `ClientRequestToken` | **YES** — token-scoped, 10-min window | identical calls = one effect |
| Link accounts on first federated sign-up | `cognito-idp:AdminLinkProviderForUser` in **PostConfirmation** | **YES — only in PostConfirmation, NOT PreSignUp** | in PreSignUp it throws `AliasExistsException` |
| Lambda response < 500 ms from a Cognito trigger | Lambda timeout, **5 s hard cap** | **PARTIAL** — Cognito enforces a 5 s ceiling you can't raise | budget within 5 s |
| Fan-out from one source | EventBridge **Pipes** | **NO** — Pipes is single-source→single-target | fan-out is an **event bus**, not Pipes |

## Worked instances (verified live)

- **Cross-service atomicity → DynamoDB only.** `TransactWriteItems` is all-or-nothing but its actions
  "can target items in different tables, but **not in different AWS accounts or Regions**." Across
  DynamoDB + S3 + SQS there is **no** atomic primitive — that's Saga territory, hence eventual
  consistency.
- **Atomic account-link in PreSignUp → unsupported.** `AdminLinkProviderForUser` in the **PreSignUp**
  trigger throws `AliasExistsException` ("Already found an entry for username") on the first federated
  sign-up — the user doesn't exist yet. It's a documented limitation; the supported path is
  **PostConfirmation**. So "atomically link in PreSignUp" is a property AWS does **not** provide.
- **SLA from a Cognito trigger → capped at 5s.** "Amazon Cognito invokes Lambda functions
  synchronously … it must respond within 5 seconds … You can't change this five-second timeout
  value." Any promised trigger SLA lives under that ceiling.
- **Compensation must itself be idempotent.** Step Functions runs compensating tasks on failure, but
  AWS notes compensations can fail and must be retried to success — so even the rollback needs its own
  backing guarantee (an idempotent API).

## Anti-pattern to detect

- A design doc asserting "exactly-once / atomic across services / fully reversible" with **no named
  AWS method** beside the claim.
- Reaching for an SDK method that's valid generally but **unsupported in the specific context** (e.g.
  `AdminLinkProviderForUser` in PreSignUp).
- Promising a sub-5s trigger SLA without accounting for Cognito's hard 5s cap.
- Calling an EventBridge Pipes design "fan-out."

## Decision aid

- **Property has a named backing method that supports it in-context?** → keep the claim; cite the
  method.
- **No backing method, or method unsupported in this context?** → downgrade to "best-effort, not
  mechanically guaranteed," or change the design (e.g. Saga + idempotency instead of cross-service
  ACID).

## Related skills

- `global-skills/aws-go/aws-adversarial-audit-before-new-pattern/SKILL.md` — its sibling: that skill
  finds issues *before* implementation; this one verifies the *claims* a design makes at review. Run
  that first, this at design-review.
- `global-skills/aws-go/aws-sdk-error-handling-canonical/SKILL.md` — the
  `AdminLinkProviderForUser`/PreSignUp error is the type-vs-text case; never silence it by string match.

## Sources

- [AWS Saga orchestration](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html) (eventual consistency, idempotency, no isolation) · [Serverless Saga with Step Functions](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/implement-the-serverless-saga-pattern-by-using-aws-step-functions.html)
- [DynamoDB TransactWriteItems](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_TransactWriteItems.html) (same account+Region only; `ClientRequestToken` idempotency)
- [Cognito Lambda trigger limits (5s cap)](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-working-with-lambda-triggers.html) · [AdminLinkProviderForUser](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AdminLinkProviderForUser.html) · [EventBridge Pipes (point-to-point)](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-pipes.html)

---

**Last verified:** 2026-06-03 (live): AWS Saga guidance admits eventual consistency / no transaction
isolation; `TransactWriteItems` is atomic but "not in different AWS accounts or Regions"; Cognito
triggers have a hard, unchangeable 5s timeout; `AdminLinkProviderForUser` is unsupported in PreSignUp
(use PostConfirmation); EventBridge Pipes routes a single source to a single target.
**Re-check after:** AWS SDK Go v2 major / CDK CLI major, or by 2026-09-03. **Decay risk:** low (these
are durable service guarantees; re-confirm the specific method facts each major).
**Found a drift?** Run `/skill-pattern-freshness-audit aws-go`.
