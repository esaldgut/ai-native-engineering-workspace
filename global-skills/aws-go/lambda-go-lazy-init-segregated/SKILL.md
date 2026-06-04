---
name: lambda-go-lazy-init-segregated
description: >-
  In a Go Lambda with multiple clients (AWS SDK, MongoDB, Cognito, EventBridge), segregate the
  sync.Once values by real usage path — never one initAll() that ties every client together. Each
  dependency you instantiate during INIT (init() or eager globals) is paid at every cold start, on
  every code path; the happy path must not pay the cold-start tax for clients it never touches. Use
  when a Go Lambda has 2+ clients across branches, when you see init() opening a Mongo/DB connection
  unconditionally, or when adding a new client to an existing handler. (Go has no SnapStart, so this
  matters more than in Java/Python.)
version: "1.0.0"
freshness:
  verified_against:
    - source: "AWS — Best practices for working with Lambda functions (Function code)"
      url: "https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html"
      version: "AWS Lambda Developer Guide"
    - source: "pkg.go.dev — sync.Once"
      url: "https://pkg.go.dev/sync#Once"
      version: "Go 1.x"
    - source: "pkg.go.dev — aws-lambda-go/lambda (lambda.Start)"
      url: "https://pkg.go.dev/github.com/aws/aws-lambda-go/lambda"
      version: "v1.54.0 (Mar 20, 2026)"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "AWS SDK Go v2 major + CDK CLI major"
    or_date: "2026-09-03"
  decay_risk: low
  status: current
---

# Segregated lazy init for Go Lambdas

AWS Lambda's
[best practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html) say verbatim:
"Initialize SDK clients and database connections outside of the function handler … Subsequent
invocations processed by the same instance of your function can reuse these resources." True — but
the language-neutral doc stops short of the Go-specific refinement: **in Go, anything you initialize
during INIT (an `init()` block or an eager package-level global) runs unconditionally on every cold
start, on every code path.** `init()` cascades across all imported packages. So a Mongo connection
opened in `init()` taxes 100% of cold starts even if 70% of invocations never touch Mongo.

The fix: defer each expensive client behind its **own** `sync.Once`, grouped by *temporal locality of
use* — not one `initAll()`.

## When to invoke

- A Go Lambda has **2+ clients** (AWS SDK service clients, Mongo, an HTTP client to a slow upstream)
  used on **different branches**.
- You see `init()` (or an eager global) opening a database/Mongo connection or constructing a client
  the common path doesn't use.
- Adding a new client to an existing handler — wire it to the branch that needs it, not to `init()`.

**Announce on invoke:** "Using `lambda-go-lazy-init-segregated` to split sync.Once by usage path so the happy path doesn't pay cold-start for unused clients."

## The mental model

- The AWS `aws.Config` itself is **cheap** — load it once (globally or behind one `sync.Once`).
- Each `serviceX.NewFromConfig(cfg)` is cheap-ish; a **Mongo dial** or a TLS handshake to a slow
  upstream is the expensive part.
- Group `sync.Once` by *which branch needs the client together*, not "all infra clients in one Once."
- `sync.Once.Do` guarantees exactly-once, goroutine-safe, lazy execution — the idiomatic deferral
  primitive.

Example shape: an inbound-federation handler with 3 clients (an identity provider, EventBridge,
Mongo) across **2** segregated `Once` values. The most common branch — a brand-new federated
sign-up with no candidate to reconcile — initializes **only** the identity-provider client, never
Mongo or EventBridge.

## Why Go specifically

Java/Python/.NET have **SnapStart**, which snapshots a pre-initialized environment so eager init is
effectively free. **Go has no SnapStart.** Every cold start re-runs INIT from scratch, so segregated
lazy init is the primary lever for Go cold-start cost. (If you *do* use Provisioned Concurrency, INIT
runs during pre-warm and the value diminishes — this pattern matters most for **on-demand** Lambdas
with bursty traffic and seldom-used branches.)

## Canonical example

```go
package main

import (
	"context"
	"sync"

	"github.com/aws/aws-lambda-go/lambda"
	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/eventbridge"
	"github.com/aws/aws-sdk-go-v2/service/cognitoidentityprovider"
	"go.mongodb.org/mongo-driver/v2/mongo"
)

// aws.Config: loaded once, cheap. Shared by all service clients.
var (
	cfgOnce sync.Once
	awsCfg  aws.Config
)

func loadAWSConfig(ctx context.Context) aws.Config {
	cfgOnce.Do(func() {
		c, err := config.LoadDefaultConfig(ctx)
		if err != nil {
			panic(err) // init failure: fail fast, structured error to Lambda
		}
		awsCfg = c
	})
	return awsCfg
}

// Once #1 — identity path. Every invocation needs the IdP client (cheap).
var (
	idpOnce   sync.Once
	idpClient *cognitoidentityprovider.Client
)

func initIdentity(ctx context.Context) {
	idpOnce.Do(func() { idpClient = cognitoidentityprovider.NewFromConfig(loadAWSConfig(ctx)) })
}

// Once #2 — data-plane path. Mongo dial is EXPENSIVE; only the reconcile branch pays it.
var (
	dataOnce    sync.Once
	ebClient    *eventbridge.Client
	mongoClient *mongo.Client
)

func initDataPlane(ctx context.Context) {
	dataOnce.Do(func() {
		ebClient = eventbridge.NewFromConfig(loadAWSConfig(ctx))
		mongoClient = connectMongo(ctx) // the costly dial, deferred off the common path
	})
}

func handler(ctx context.Context, e MyEvent) (MyResp, error) {
	initIdentity(ctx) // common: only the cheap IdP client
	if needsReconcile(e) {
		initDataPlane(ctx) // rare: only this branch pays Mongo + EventBridge cold start
	}
	return doWork(ctx, e)
}

func main() { lambda.Start(handler) }
```

## Anti-pattern to detect (greppable)

- A single `func initAll()` or one `sync.Once` that constructs **every** client.
- `init()` calling `connectMongo(...)`, `sql.Open(...)`, or any dial unconditionally.
- An eager package-level `var mongoClient = mustConnect()` outside any `Once`/branch.
- A `sync.Once` shared across two branches that never co-occur.

## Decision aid

- **Client used on (almost) every invocation, cheap to build?** → fine to init eagerly or behind one
  `Once`.
- **Client expensive (Mongo dial, slow-upstream TLS) and used on a minority branch?** → its own
  `sync.Once`, called *inside* that branch only.
- **On Provisioned Concurrency?** → the asymmetry shrinks; still prefer segregation for the
  on-demand overflow path.

## Related skills

- `global-skills/aws-go/lambda-go-refactor-purge-audit/SKILL.md` — when a branch (and its client) is
  removed, `go mod tidy` purges the module; audit the now-dead IAM actions.
- `global-skills/aws-go/aws-sdk-go-v2-version-policy/SKILL.md` — share one `aws.Config`; construct
  each chosen service client with `NewFromConfig`.

## Sources

- [Lambda best practices — Function code (init outside the handler, reuse the environment)](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [sync.Once](https://pkg.go.dev/sync#Once) · [aws-lambda-go/lambda (lambda.Start)](https://pkg.go.dev/github.com/aws/aws-lambda-go/lambda)
- [Understanding and Remediating Cold Starts (AWS Compute Blog)](https://aws.amazon.com/blogs/compute/understanding-and-remediating-cold-starts-an-aws-lambda-perspective/)

---

**Last verified:** 2026-06-03 against the AWS Lambda best-practices guide (live — "Initialize SDK
clients and database connections outside of the function handler"), `sync.Once` semantics, and
`aws-lambda-go/lambda` v1.54.0 (`lambda.Start`). Go has no SnapStart, confirmed against the
SnapStart docs (Java/Python/.NET only).
**Re-check after:** AWS SDK Go v2 major / CDK CLI major, or by 2026-09-03. **Decay risk:** low (the
INIT-phase cost model and `sync.Once` are stable primitives).
**Found a drift?** Run `/skill-pattern-freshness-audit aws-go`.
