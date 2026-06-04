---
name: aws-sdk-error-handling-canonical
description: >-
  Handle AWS SDK Go v2 errors the canonical way — errors.As against typed/modeled errors and the
  smithy.APIError interface — and NEVER strings.Contains(err.Error(), "...") to silence or branch on
  an SDK error. String-matching error text breaks silently across SDK upgrades and, worse, masks a
  deeper bug: if a "silenced" error fires on every normal invocation, the API is being called in a
  context AWS does not support (e.g. AdminLinkProviderForUser in PreSignUp). Use when writing or
  reviewing any error-handling branch around an AWS SDK Go v2 call, especially one that treats an
  error as "idempotent / already exists / edge case OK."
version: "1.0.0"
freshness:
  verified_against:
    - source: "AWS — Handling Errors in the AWS SDK for Go V2 (Developer Guide)"
      url: "https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/handle-errors.html"
      version: "SDK Go v2"
    - source: "pkg.go.dev — smithy-go (APIError, OperationError, ErrorFault)"
      url: "https://pkg.go.dev/github.com/aws/smithy-go"
      version: "smithy-go (Jun 2026)"
    - source: "Go standard library — errors.As"
      url: "https://pkg.go.dev/errors#As"
      version: "Go 1.x"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "AWS SDK Go v2 major + CDK CLI major"
    or_date: "2026-09-03"
  decay_risk: low
  status: current
---

# Canonical AWS SDK Go v2 error handling

The AWS SDK for Go v2 contracts on **error type identity**, not error **message text**. The
[official guide](https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/handle-errors.html) is
explicit: errors implement `Unwrap`, and you use `errors.As` to reach the typed layers. Error
*messages* are layered (a wrap chain) and AWS does **not** guarantee their stability across SDK or
service versions — so `strings.Contains(err.Error(), "...")` is undefined behavior that compiles
today and breaks on the next release with no signal.

## When to invoke

- Writing any `if err != nil` branch around an AWS SDK Go v2 call that **does something other than
  return the error** — silences it, treats it as "already exists / idempotent," or routes on it.
- Reviewing code that contains `strings.Contains(err.Error(), ...)`, `strings.HasPrefix(err.Error(),
  ...)`, or a `switch` over `err.Error()` against an AWS call.
- Cross-service handlers that need generic error-code routing.

**Announce on invoke:** "Using `aws-sdk-error-handling-canonical` to branch on typed errors via `errors.As` + `smithy.APIError`, not on error text."

## The three typed layers (most specific first)

1. **Operation-specific modeled error** — e.g. `*types.BucketAlreadyExists` for S3 `CreateBucket`,
   `*types.UsernameExistsException` / `*types.AliasExistsException` for Cognito. Most precise; use
   for business logic. Lives in each service's `.../service/<name>/types` package.
2. **`smithy.APIError`** (interface) — covers both modeled and un-modeled responses. Exposes
   `ErrorCode() string`, `ErrorMessage() string`, `ErrorFault() ErrorFault` (and `ErrorFault.String()`).
   Use for generic, cross-service code-based routing.
3. **`smithy.OperationError`** — wraps everything with service + operation context (`Service()`,
   `Operation()`, `Unwrap()`). Use for centralized logging.

## The hard rule: never string-match SDK error text to silence

The anti-pattern isn't just fragile — it's a **misdiagnosis detector**. If you find yourself writing:

```go
// WRONG — fragile AND a symptom of a deeper bug
if err != nil && strings.Contains(err.Error(), "Already found an entry for username") {
	return nil // "idempotent, the link already exists" — NO.
}
```

…and that branch fires on **every normal invocation**, the silence is hiding the real fact: the API
is being invoked in a context AWS does not support. The canonical instance —
`cognito-idp:AdminLinkProviderForUser` called inside the **PreSignUp** trigger — returns
`AliasExistsException` ("Already found an entry for username") on the *first* federated sign-up
because the user does not exist yet. It is **not** idempotency; it is a documented Cognito
limitation. The fix is not a stronger string match — it's reading the doc and moving the call to
**PostConfirmation** (see related skill). String-silencing would have buried that signal.

## Canonical example

```go
import (
	"context"
	"errors"
	"log"

	"github.com/aws/aws-sdk-go-v2/service/myservice"
	mytypes "github.com/aws/aws-sdk-go-v2/service/myservice/types"
	"github.com/aws/smithy-go"
)

func DoThing(ctx context.Context, c *myservice.Client, id string) error {
	_, err := c.DescribeThing(ctx, &myservice.DescribeThingInput{Id: &id})
	if err == nil {
		return nil
	}

	// 1) Preferred: typed modeled error — "absence is legitimately OK" handled by TYPE.
	var nf *mytypes.ResourceNotFoundException
	if errors.As(err, &nf) {
		log.Printf("thing %s not found", id)
		return nil
	}

	// 2) Generic fallback: code-based routing for any AWS error.
	var ae smithy.APIError
	if errors.As(err, &ae) {
		log.Printf("aws error code=%s msg=%s fault=%s",
			ae.ErrorCode(), ae.ErrorMessage(), ae.ErrorFault().String())
	}
	return err
}
```

For batch/cross-service logging, add the operation context:

```go
var oe *smithy.OperationError
if errors.As(err, &oe) {
	log.Printf("failed service=%s operation=%s: %v", oe.Service(), oe.Operation(), oe.Unwrap())
}
```

## Anti-pattern to detect (greppable)

- `strings.Contains(err.Error(), ...)`, `strings.HasPrefix(err.Error(), ...)`, or `err.Error() ==
  "..."` anywhere near an AWS SDK call — reject in review.
- Any branch that returns `nil` (success) on an AWS error matched by **text**.
- A silenced error whose message you copied from a stack trace — that's the tell that you're matching
  text, not type.

## Decision aid

- **Need "this specific error means X"** (already exists, not found, conflict)? → `errors.As` with the
  service's modeled `*types.XxxException`.
- **Need "any AWS error with code C"** across N services? → `errors.As(err, &ae)` then compare
  `ae.ErrorCode()`.
- **A silence fires on every happy-path call?** → stop. Read the AWS doc for that API in that context.
  The error is telling you the API isn't supported there.

## Related skills

- `global-skills/aws-go/verify-provider-api-supports-property/SKILL.md` — the
  `AdminLinkProviderForUser`-in-PreSignUp case is the textbook "promised property the provider API
  doesn't support" — verify the method before promising atomic account-linking.
- `global-skills/aws-go/aws-sdk-go-v2-version-policy/SKILL.md` — modeled `types` packages live under
  the service package you chose; pick that deliberately first.

## Sources

- [Handling Errors in the AWS SDK for Go V2](https://docs.aws.amazon.com/sdk-for-go/v2/developer-guide/handle-errors.html) — uses `errors.As` with `*types.BucketAlreadyExists`, `smithy.APIError`, `smithy.OperationError`
- [smithy.APIError / OperationError / ErrorFault](https://pkg.go.dev/github.com/aws/smithy-go) · [errors.As](https://pkg.go.dev/errors#As)
- [Cognito AdminLinkProviderForUser API ref](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AdminLinkProviderForUser.html) · [re:Post — "Already found an entry for username"](https://repost.aws/questions/QUgWVkIodQS1W3Yj8MYjInbA/cognito-auth-flow-fails-with-already-found-an-entry-for-username-username)

---

**Last verified:** 2026-06-03 against the AWS SDK Go v2 error-handling guide (live), which models
the canonical pattern as `errors.As` against `*types.BucketAlreadyExists` and the `smithy.APIError`
interface (`ErrorCode()`/`ErrorMessage()`/`ErrorFault().String()`); the
`AdminLinkProviderForUser`/PreSignUp limitation re-confirmed on AWS re:Post.
**Re-check after:** AWS SDK Go v2 major / CDK CLI major, or by 2026-09-03. **Decay risk:** low
(typed-error contract is stable; the message text it replaces is what churns).
**Found a drift?** Run `/skill-pattern-freshness-audit aws-go`.
