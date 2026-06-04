---
name: cdk-three-source-drift-check
description: >-
  Before a CDK deploy that touches an already-deployed stack, cross-check three sources, not one:
  cdk synth (what your code says should exist), cdk diff (synth vs the last-deployed template), and
  cdk drift (the deployed template vs live AWS reality). cdk diff is BLIND to manual console/CLI
  changes — it only compares templates, never live state; cdk drift (CDK CLI 2.1017.0+, May 2025)
  calls CloudFormation drift detection to catch them. Note that not all resource types support
  drift detection (status NOT_CHECKED), so keep an aws cli describe fallback for those. Use when
  planning a CDK deploy onto live stacks (IAM, Lambda env, EventBridge rules, API Gateway, Cognito
  triggers).
version: "1.0.0"
freshness:
  verified_against:
    - source: "AWS — cdk drift command reference (CDK v2)"
      url: "https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-drift.html"
      version: "CDK CLI 2.1017.0+ (May 2025)"
    - source: "AWS — cdk diff command reference (CDK v2)"
      url: "https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-diff.html"
      version: "CDK CLI v2"
    - source: "AWS CloudFormation — Resource type support for drift detection"
      url: "https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-supported-resources.html"
      version: "CloudFormation User Guide"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "AWS SDK Go v2 major + CDK CLI major"
    or_date: "2026-09-03"
  decay_risk: high
  status: current
---

# CDK three-source drift check

`cdk diff` does **not** see manual console/CLI changes. It compares two *templates* — the one your
code synthesizes and the one CloudFormation last deployed — and nothing else. If the deployed template
says "Lambda timeout 30s" and someone bumped it to 60s in the console, `cdk diff` shows **nothing**;
the change is real but invisible to template-vs-template comparison. To catch it you need a third
source that queries **live AWS reality**.

CDK now ships that source: **`cdk drift`** (CDK CLI **2.1017.0**, released **May 2025**). It calls
CloudFormation's drift-detection operation. The
[official reference](https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-drift.html) draws the line
verbatim:

> "`cdk drift` calls CloudFormation's drift detection operation to compare the actual state of
> resources in AWS ('reality') against their expected configuration in CloudFormation. … `cdk diff`
> compares the CloudFormation template synthesized from your local CDK code against the template of
> the deployed CloudFormation stack."

> **Version note (correction):** use **`cdk drift` from CDK CLI 2.1017.0 (May 2025)**, the release
> that introduced the command — `npm install -g aws-cdk@2.1017.0` or newer. (Do not cite a later
> phantom version; 2.1017.0 is where `cdk drift` shipped.)

## When to invoke

- Planning a `cdk deploy` that **modifies an already-deployed stack** — IAM roles/policies, Lambda env
  or timeout/memory, EventBridge rules, API Gateway, Cognito triggers.
- Any time you suspect a hotfix was applied in the console/CLI outside CDK.
- Before declaring "infra matches code."

**Announce on invoke:** "Using `cdk-three-source-drift-check` to cross-check cdk synth + cdk diff + cdk drift before touching a live stack."

## The three sources

| # | Command | Compares | Catches |
|---|---|---|---|
| 1 | `cdk synth` | (produces the template from **local code**) | what the code says should exist |
| 2 | `cdk diff` | synth output vs **last-deployed template** | what *your* pending change will alter |
| 3 | `cdk drift` | deployed template vs **live AWS state** | what *someone else* changed in console/CLI |

Source 2 catches *your* drift-to-be; source 3 catches *their* already-landed drift. You need both, and
synth as the anchor.

## Caveats (verified)

- **Not all resources support drift detection.** Unsupported types report `NOT_CHECKED`; consult the
  CloudFormation
  [resource-type support list](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-supported-resources.html).
  Known gaps: Lambda `PackageType=Image`, Lambda's `Code` property (can't map to source), and IAM
  `User.LoginProfile.Password` (intentionally excluded as sensitive). For those, an
  `aws ... describe-*` call is the manual fallback — which is why "three sources" still holds.
- **Drift detection is asynchronous.** Under the hood CloudFormation returns a `DriftDetectionId` you
  poll; `cdk drift` does this for you, but budget ~30–60s per stack.
- **False positives exist.** `AWS::ApiGateway::Authorizer` and `AWS::CloudFront::Distribution` have
  shown null tag/`identitySource` values without real drift — verify before reverting.
- **`--fail` for CI.** `cdk drift --fail` returns exit code 1 when drift is detected — wire it into a
  pipeline gate.
- **Pre-existing drift masks new drift.** If old drift was never reverted, a new manual change blends
  into the same report. Keep stacks drift-clean so new drift stands out.

## Canonical example

```bash
# (1) synth — what your code says should exist
cdk synth MyServiceStack > /tmp/synth.template.json

# (2) diff — what your pending change will alter vs the last-deployed template
cdk diff MyServiceStack

# (3) drift — deployed template vs LIVE AWS; --fail for a CI gate
cdk drift MyServiceStack --fail

# Manual fallback for resource types that don't support drift detection (NOT_CHECKED)
aws cloudformation list-stack-resources --stack-name MyServiceStack \
  --query 'StackResourceSummaries[?DriftInformation.StackResourceDriftStatus==`NOT_CHECKED`].LogicalResourceId'

# For each NOT_CHECKED resource, describe it directly and eyeball-compare to synth output
aws lambda get-function --function-name MyServiceFunction \
  | jq '.Configuration | {Timeout, MemorySize, Environment}'
```

## Anti-pattern to detect (greppable)

- A deploy plan that runs **only** `cdk diff` and calls the stack "in sync."
- Treating a clean `cdk diff` as proof that no console change happened (it cannot prove that).
- Reverting an `Authorizer`/`CloudFront` "drift" without confirming it isn't the known null-value
  false positive.

## Decision aid

- **Touching a live stack and want both your change and others' changes?** → run all three.
- **Resource shows `NOT_CHECKED`?** → `aws ... describe-*` it manually and compare to synth.
- **CI gate?** → `cdk drift --fail`.
- **Only validating a brand-new, never-deployed stack?** → `cdk synth` + `cdk diff` suffice (nothing
  live to drift yet).

## Related skills

- `global-skills/aws-go/lambda-go-refactor-purge-audit/SKILL.md` — after tightening an IAM role in
  CDK, drift-check that the deployed role matches reality before declaring the cleanup done.
- `global-skills/aws-go/aws-adversarial-audit-before-new-pattern/SKILL.md` — for a *first-time*
  resource (first EventBridge Pipes, etc.), audit drift-detection support up front.

## Sources

- [cdk drift](https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-drift.html) (CDK CLI 2.1017.0+, `--fail`) · [cdk diff](https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-diff.html) · [cdk synth](https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-synth.html)
- [CloudFormation drift detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html) · [Resource type support](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-supported-resources.html)
- [aws cloudformation detect-stack-drift](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/detect-stack-drift.html)

---

**Last verified:** 2026-06-03 against the CDK v2 `cdk drift` reference (live — calls CloudFormation
drift detection, `--fail` flag) and confirmed `cdk drift` shipped in **CDK CLI 2.1017.0 (May 2025)**,
the version correction applied over an earlier phantom-version claim. `cdk diff` confirmed as
template-vs-template (blind to console changes).
**Re-check after:** CDK CLI major / AWS SDK Go v2 major, or by 2026-09-03. **Decay risk:** high
(CDK CLI verbs and drift-support coverage churn release-to-release).
**Found a drift?** Run `/skill-pattern-freshness-audit aws-go`.
