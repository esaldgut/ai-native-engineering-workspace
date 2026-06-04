# aws-go — Lambda Go, AWS SDK v2, CDK skills

8 skills for backend engineering on AWS with Go. SDK conventions, Lambda cold-start discipline,
infrastructure drift, and two meta-disciplines for not promising what the provider can't deliver.
Verified against `pkg.go.dev`, `docs.aws.amazon.com`, and the MongoDB docs.

## SDK & error handling

| Skill | What it does |
|-------|--------------|
| [`aws-sdk-go-v2-version-policy`](aws-sdk-go-v2-version-policy/SKILL.md) | Which AWS SDK Go package to import — default `aws-sdk-go-v2` + latest stable service-API generation; distinguishes SDK version from service-API generation (`ses` vs `sesv2`) |
| [`aws-sdk-error-handling-canonical`](aws-sdk-error-handling-canonical/SKILL.md) | The canonical way — `errors.As` + `smithy.APIError` typed errors; never `strings.Contains(err.Error(), …)` to silence errors |

## Lambda discipline

| Skill | What it does |
|-------|--------------|
| [`lambda-go-lazy-init-segregated`](lambda-go-lazy-init-segregated/SKILL.md) | Segregate `sync.Once` by usage path so a happy path never pays cold-start for clients it doesn't use |
| [`lambda-go-refactor-purge-audit`](lambda-go-refactor-purge-audit/SKILL.md) | After a refactor removes functionality, `go mod tidy` + audit the dependency purge → flag over-provisioned IAM |

## Infrastructure & data

| Skill | What it does |
|-------|--------------|
| [`cdk-three-source-drift-check`](cdk-three-source-drift-check/SKILL.md) | Before a CDK deploy on a live stack, cross-check three sources: `cdk synth`, `cdk diff`, and `cdk drift` (CDK 2.1017.0+) — `cdk diff` is blind to console changes |
| [`mongo-ttl-bson-tag-match`](mongo-ttl-bson-tag-match/SKILL.md) | A MongoDB TTL index `Keys` field MUST match the Go struct's `bson` tag exactly — MongoDB silently skips docs missing the field (no error) |

## Meta-disciplines (verify before you promise)

| Skill | What it does |
|-------|--------------|
| [`aws-adversarial-audit-before-new-pattern`](aws-adversarial-audit-before-new-pattern/SKILL.md) | Before implementing a *new* AWS pattern (first Step Functions, first Bedrock Guardrails…), run an adversarial doc-research pass first |
| [`verify-provider-api-supports-property`](verify-provider-api-supports-property/SKILL.md) | Before a design doc promises a property (idempotency, exactly-once, cross-service atomicity), confirm the concrete AWS SDK method that guarantees it exists |

---

**Freshness:** AWS moves fast — re-check after an **AWS SDK Go v2 major or CDK CLI major**, or by
**2026-09-03** (quarterly). `cdk-three-source-drift-check` is `decay_risk: high` (CDK CLI verbs
change often). Run `/skill-pattern-freshness-audit aws-go`.
