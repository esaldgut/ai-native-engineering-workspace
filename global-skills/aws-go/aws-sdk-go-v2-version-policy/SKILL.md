---
name: aws-sdk-go-v2-version-policy
description: >-
  Decide which AWS SDK Go package to import in a Lambda or Go service the canonical way:
  default to aws-sdk-go-v2 (the modular v2 SDK) plus the latest stable v1-generation service
  package (service/ses, service/cognitoidentityprovider, service/s3), and add a serviceXv2
  package (sesv2, cloudwatchlogsv2) only on a whitelist, when you actually need a v2-API-only
  feature. Distinguishes the SDK major version (v1 vs v2) from the service-API generation
  (ses vs sesv2) — the two axes that engineers conflate. Use before adding any
  github.com/aws/aws-sdk-go-v2/service/* import, before a go get -u that crosses a service
  package major, or when someone proposes "migrate service X to its v2 API."
version: "1.0.0"
freshness:
  verified_against:
    - source: "AWS SDK for Go v2 — root module (pkg.go.dev)"
      url: "https://pkg.go.dev/github.com/aws/aws-sdk-go-v2"
      version: "aws-sdk-go-v2 (Jun 2026)"
    - source: "AWS — Migrating to the AWS SDK for Go V2 (Developer Guide)"
      url: "https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/migrate-gosdk.html"
      version: "SDK Go v2"
    - source: "pkg.go.dev — service/cognitoidentityprovider (single generation)"
      url: "https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/cognitoidentityprovider"
      version: "v1.61.2 (Jun 3, 2026)"
    - source: "pkg.go.dev — service/sesv2 (distinct SES API v2 generation)"
      url: "https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/sesv2"
      version: "v1.62.2 (Jun 3, 2026)"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "AWS SDK Go v2 major + CDK CLI major"
    or_date: "2026-09-03"
  decay_risk: medium
  status: current
---

# AWS SDK Go v2 version policy

The canonical Go SDK for AWS is **`aws-sdk-go-v2`** (`github.com/aws/aws-sdk-go-v2`). The v1 SDK
(`github.com/aws/aws-sdk-go`) is in maintenance mode and AWS publishes a dedicated
[migration guide](https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/migrate-gosdk.html). v2
is **modular**: every service is its own independently-versioned Go module under
`github.com/aws/aws-sdk-go-v2/service/<name>`. Importing `service/s3` does not drag in unrelated
clients, which keeps the Lambda zip small and the cold start cheap.

This skill exists because two unrelated "version" decisions get conflated. Keep them separate.

## When to invoke

- Before adding **any** new `github.com/aws/aws-sdk-go-v2/service/*` import.
- Before a `go get -u` that would cross a **major** of a service package.
- When someone says "let's migrate service X to its v2 API" (e.g., `ses` → `sesv2`).
- When reviewing a PR that imports a `serviceXv2` package — confirm the v2-only feature is real.

**Announce on invoke:** "Using `aws-sdk-go-v2-version-policy` to pick the SDK + service-API generation per the default-v2/whitelist-v2-API rule."

## The two axes (do not conflate them)

| Axis | What it is | Default | The suffix that signals it |
|---|---|---|---|
| **A — SDK major** | `aws-sdk-go` (v1) vs `aws-sdk-go-v2` (v2) | **v2** | the `-v2` in the *module root* `github.com/aws/aws-sdk-go-v2` |
| **B — Service-API generation** | e.g. `service/ses` vs `service/sesv2` | **latest stable v1-gen** | the `v2` *inside the service path* `service/sesv2` |

The trap: an engineer hears "use v2" (axis A) and imports `service/sesv2` (axis B), assuming they're
the same decision. They are not. `service/ses` and `service/sesv2` **both ship inside the v2 SDK** and
co-exist. The `v2` in `sesv2` is the **SES API generation**, not the SDK version.

## The policy

1. **Default to the v2 SDK.** Stay on v1 only if a third-party library locks you in.
2. **Default to the latest stable v1-generation service package** — `service/ses`,
   `service/cognitoidentityprovider`, `service/s3`, `service/dynamodb`, `service/eventbridge`,
   `service/sfn`, `service/sqs`, `service/sns`. For a vanilla `SendEmail`, `service/ses` is the path
   of least surprise.
3. **Add a `serviceXv2` package only on a whitelist** — when you need a feature that generation
   introduced. SESv2 adds subscription/list management, dedicated IP pools, and the v2 unsubscribe
   surface; reach for `service/sesv2` **only** when one of those is in scope, not by reflex. AWS's
   own messaging is that new SES capabilities land in the SESv2 API, so SES is on a long arc toward
   `sesv2` — but migrate deliberately, per feature.

### Not every service has a v2 API

Verified on pkg.go.dev (Jun 2026): **Cognito User Pools, S3, DynamoDB, and Lambda have no
`serviceXv2` package** — `service/cognitoidentityprovider` is the single generation. "Migrate Cognito
to its v2 API" is a non-task; the package does not exist. Don't invent a migration that has no target.

## Canonical example

Default posture: v2 SDK, v1-generation service packages, one shared `aws.Config`.

```go
package main

import (
	"context"

	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/cognitoidentityprovider"
	"github.com/aws/aws-sdk-go-v2/service/ses" // v1-gen SES API — correct for SendEmail
	// "github.com/aws/aws-sdk-go-v2/service/sesv2" // whitelist: only for list/IP-pool/v2 features
)

var (
	cognitoClient *cognitoidentityprovider.Client
	sesClient     *ses.Client
)

func init() {
	cfg, err := config.LoadDefaultConfig(context.TODO())
	if err != nil {
		panic(err)
	}
	// NewFromConfig is the canonical constructor on every v2 service client.
	cognitoClient = cognitoidentityprovider.NewFromConfig(cfg)
	sesClient = ses.NewFromConfig(cfg)
}
```

## Anti-pattern to detect (greppable)

- An import of `service/sesv2` (or any `serviceXv2`) with **no** v2-only feature in the diff — the
  reviewer should ask "which SESv2 capability?" and bounce it if the answer is "none."
- A PR that pins `github.com/aws/aws-sdk-go` (v1) for a *new* service when v2 covers it.
- A go.mod that mixes v1 and v2 of the **same** service generation — pick one.

## Decision aid

- **Need only the basic operation** (SendEmail, PutObject, AdminGetUser)? → v1-generation
  `service/<name>`.
- **Need a capability the service's v2 API added** (SESv2 subscription management, dedicated IP
  pools)? → whitelist `service/<name>v2`, note the reason in the PR.
- **Unsure whether a v2 generation even exists** for the service? → check pkg.go.dev first;
  Cognito/S3/DynamoDB/Lambda don't have one.

## Related skills

- `global-skills/aws-go/aws-sdk-error-handling-canonical/SKILL.md` — once you've picked the package,
  handle its errors with `errors.As` + `smithy.APIError`, never string-matching.
- `global-skills/aws-go/lambda-go-refactor-purge-audit/SKILL.md` — when a refactor *removes* a
  service import, `go mod tidy` purges the module; audit the IAM role it implied.

## Sources

- [aws-sdk-go-v2 root module](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2) · [Migrating to SDK Go V2](https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/migrate-gosdk.html)
- [service/cognitoidentityprovider](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/cognitoidentityprovider) (single generation) · [service/ses](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/ses) · [service/sesv2](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/sesv2) (distinct SES API v2)
- [config.LoadDefaultConfig](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/config#LoadDefaultConfig)

---

**Last verified:** 2026-06-03 against pkg.go.dev (live): `service/cognitoidentityprovider` v1.61.2
is single-generation (no `cognitoidentityproviderv2`); `service/ses` and `service/sesv2` co-exist
inside the v2 SDK, `sesv2` exposing `SendEmail` as the SES **API v2** generation — confirming the
SDK-version vs service-API-generation distinction.
**Re-check after:** AWS SDK Go v2 major / CDK CLI major, or by 2026-09-03. **Decay risk:** medium.
**Found a drift?** Run `/skill-pattern-freshness-audit aws-go`.
