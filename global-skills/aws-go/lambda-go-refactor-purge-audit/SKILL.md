---
name: lambda-go-refactor-purge-audit
description: >-
  After a refactor that REMOVES functionality from a Go Lambda (or any module with a go.mod), run
  go mod tidy and audit the dependency purge. If the purge drops an AWS SDK module (service/s3,
  service/cognitoidentityprovider, service/ses, ...), the lambda's IAM execution role is now
  over-provisioned — those actions are dead permissions. The go.mod delta is the objective,
  tool-detectable signal that the lambda's blast radius shrank, so it's the trigger to tighten IAM
  in the same PR or file tracked cleanup debt. Use whenever a PR deletes a feature/branch from a Go
  Lambda, or any time go mod tidy removes a github.com/aws/aws-sdk-go-v2/service/* direct dependency.
version: "1.0.0"
freshness:
  verified_against:
    - source: "Go Modules Reference — go mod tidy / indirect"
      url: "https://go.dev/ref/mod#go-mod-tidy"
      version: "Go 1.x"
    - source: "AWS — Lambda best practices (Security: most-restrictive IAM)"
      url: "https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html"
      version: "AWS Lambda Developer Guide"
    - source: "AWS — IAM Access Analyzer (unused access findings)"
      url: "https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html"
      version: "IAM User Guide"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "AWS SDK Go v2 major + CDK CLI major"
    or_date: "2026-09-03"
  decay_risk: low
  status: current
---

# Refactor purge → IAM blast-radius audit

Two well-established practices, fused into one signal: (a) `go mod tidy` removes modules no longer
reachable in the import graph and adds missing ones
([Go Modules Reference](https://go.dev/ref/mod#go-mod-tidy)); (b) the AWS Lambda
[best practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html) say verbatim:
"Use most-restrictive permissions when setting IAM policies. Understand the resources and operations
your Lambda function needs, and limit the execution role to these permissions." The fusion: **the
`go mod tidy` diff after a refactor is a proxy metric for IAM over-provisioning.** When the refactor
removes the last call site for `service/cognitoidentityprovider`, `go mod tidy` purges that module —
and the `cognito-idp:*` actions on the execution role just became dead permissions nobody usually
revokes.

## When to invoke

- A PR **removes a feature or branch** from a Go Lambda (or any module with a `go.mod`).
- `go mod tidy` removes a **direct** `github.com/aws/aws-sdk-go-v2/service/*` dependency.
- You're cutting a Lambda's scope and want the IAM role to track the new, smaller blast radius.

**Announce on invoke:** "Using `lambda-go-refactor-purge-audit` to treat the go mod tidy purge as the signal to tighten the over-provisioned IAM role."

## The loop (close it — most teams don't)

1. PR removes feature X from the Lambda (and its only use of `service/myservice`).
2. Run `go mod tidy`; review `git diff go.mod`.
3. For each **direct** AWS SDK module removed, audit that lambda's execution-role policy for the
   matching actions (`myservice:*`).
4. Remove those actions **in the same PR**, or file a tracked cleanup ticket. The purge is the
   quantitative proof the blast radius shrank.

The trigger is objective — "`go mod tidy` diff includes `service/<aws-name>`" — not a vibes check.

## Watch-outs (verified)

- **`go mod tidy` is conservative.** It won't drop a module still reachable via a blank import
  `_ "..."` or a `//go:build`-guarded file you forgot. A "surviving" module masks the dead
  permission — grep for blank imports.
- **`// indirect` removals don't count.** Only **direct** deps map to IAM actions your code calls.
  An `// indirect` line (a transitive dep) reflects the dependency graph, not your call sites —
  per the Go ref, `// indirect` means "no package from this module is directly imported by the main
  module."
- **`go.sum` may keep entries** after a `go.mod` purge — benign for IAM auditing (you care about
  `go.mod` direct deps), but don't read `go.sum` as the signal.
- **One execution role per Lambda, or this breaks.** If multiple Lambdas share a role (anti-pattern),
  stripping Cognito actions can break a sibling. The audit is clean only with per-Lambda roles.
- **CDK-managed grants complicate it.** If the policy comes from `table.grantReadWriteData(fn)` /
  `bucket.grantRead(fn)` rather than a hand-written `PolicyStatement`, removing the SDK import won't
  shrink IAM until you also delete the `grant*()` call — add that to the same PR.

## Canonical example

```bash
# Baseline before the refactor.
go mod tidy

# ... refactor: delete feature X, which was the only user of service/myservice ...

go mod tidy
git diff go.mod
# Blast radius shrank if you see a DIRECT removal, e.g.:
#   - github.com/aws/aws-sdk-go-v2/service/myservice v1.x.y
# (Ignore lines ending in `// indirect`.)

# Audit the lambda's execution-role policy for the now-dead actions.
aws iam list-attached-role-policies --role-name MyServiceLambdaExecRole
aws iam get-role-policy --role-name MyServiceLambdaExecRole --policy-name MyServiceInline

# Empirical cross-check: which actions were actually used recently?
aws iam generate-service-last-accessed-details \
    --arn arn:aws:iam::123456789012:role/MyServiceLambdaExecRole
# then, with the returned JobId:
aws iam get-service-last-accessed-details --job-id <JobId>

# If "myservice" shows no recent access AND you just purged its SDK,
# drop myservice:* from the policy in THIS PR, or open a tracked cleanup ticket.
```

`IAM Access Analyzer` (unused-access findings) is the standing complement: even before the refactor,
an SDK imported but never *called* shows zero invocations and is already dead.

## Anti-pattern to detect (greppable)

- A PR that deletes a feature but leaves `go.mod` (and the IAM policy) untouched — run `go mod tidy`.
- A surviving `_ "github.com/aws/aws-sdk-go-v2/service/..."` blank import after the feature is gone.
- A CDK stack still calling `grant*()` for a service the Lambda no longer imports.

## Decision aid

- **Direct AWS SDK module purged?** → audit + tighten IAM (this PR or a ticket).
- **Only `// indirect` lines moved?** → no IAM action; transitive graph churn only.
- **Module survived an obvious feature removal?** → hunt a blank import or build-tag file before
  concluding "still needed."

## Related skills

- `global-skills/aws-go/lambda-go-lazy-init-segregated/SKILL.md` — removing a branch also removes its
  `sync.Once`/client; the two cleanups travel together.
- `global-skills/aws-go/cdk-three-source-drift-check/SKILL.md` — after tightening IAM in CDK, confirm
  the deployed role matches reality (no console-side drift) before declaring done.

## Sources

- [go mod tidy / `// indirect`](https://go.dev/ref/mod#go-mod-tidy)
- [Lambda best practices — most-restrictive IAM](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [IAM Access Analyzer — unused access](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html) · [generate-service-last-accessed-details](https://docs.aws.amazon.com/cli/latest/reference/iam/generate-service-last-accessed-details.html)

---

**Last verified:** 2026-06-03 against the Go Modules Reference (`go mod tidy` removes
no-longer-needed modules; `// indirect` = not directly imported) and the AWS Lambda best-practices
guide (live — "Use most-restrictive permissions when setting IAM policies"). IAM Access Analyzer
unused-access findings confirmed as the empirical complement.
**Re-check after:** AWS SDK Go v2 major / CDK CLI major, or by 2026-09-03. **Decay risk:** low
(`go mod tidy` and IAM least-privilege are stable).
**Found a drift?** Run `/skill-pattern-freshness-audit aws-go`.
