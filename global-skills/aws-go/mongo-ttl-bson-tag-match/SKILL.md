---
name: mongo-ttl-bson-tag-match
description: >-
  When creating or modifying a MongoDB TTL index from Go, the index Keys field MUST match the Go
  struct's bson tag exactly (byte-for-byte, case-sensitive) and the field MUST serialize to BSON
  Date. MongoDB silently skips documents that lack the indexed field or whose field isn't a date —
  no error, no warning, no log. A snake_case-vs-camelCase typo between bson:"expiresAt" and a
  Keys:{"expires_at"} index leaks storage forever and you find out weeks later via the cost graph.
  Use before any migration that calls SetExpireAfterSeconds, or when writing the first timestamped
  document of a new collection.
version: "1.0.0"
freshness:
  verified_against:
    - source: "MongoDB Manual — TTL Indexes (silent-skip behavior)"
      url: "https://www.mongodb.com/docs/manual/core/index-ttl/"
      version: "MongoDB Manual (current)"
    - source: "pkg.go.dev — mongo-driver/v2/mongo (IndexModel, IndexView.CreateOne)"
      url: "https://pkg.go.dev/go.mongodb.org/mongo-driver/v2/mongo"
      version: "v2.6.0 (Apr 27, 2026)"
    - source: "MongoDB Go Driver — Struct Tagging"
      url: "https://www.mongodb.com/docs/drivers/go/current/usage-examples/struct-tagging/"
      version: "Go driver v2"
  verified_on: "2026-06-03"
  recheck_after:
    trigger: "MongoDB Go driver v2 major + MongoDB server major"
    or_date: "2026-09-03"
  decay_risk: low
  status: current
---

# MongoDB TTL index ↔ bson tag match

A MongoDB TTL index deletes documents whose indexed **Date** field is older than the threshold. The
[MongoDB Manual](https://www.mongodb.com/docs/manual/core/index-ttl/) is explicit about the failure
mode, verbatim:

> "If a document does not contain the indexed field, the document will not expire."
>
> "If the indexed field in a document doesn't contain one or more date values, the document will not
> expire."

MongoDB does **not** error, warn, or log when it skips those documents. The TTL monitor runs every
**60 seconds**, deletes what matches, and silently ignores everything else. There is no schema
enforcement unless you explicitly add Schema Validation, and no observability surface for "documents
that should expire but don't." So a single tag mismatch is a **silent storage leak**.

## When to invoke

- Any migration or code path that creates/modifies a TTL index — i.e. calls
  `options.Index().SetExpireAfterSeconds(...)`.
- Writing the **first** timestamped document of a new collection.
- Reviewing a struct whose field feeds a TTL index, or renaming such a field/tag.

**Announce on invoke:** "Using `mongo-ttl-bson-tag-match` to verify the TTL index Keys equals the bson tag exactly and the field is a BSON Date."

## The two invariants

1. **`Keys` key string == the struct's `bson:"..."` tag, byte-for-byte.** MongoDB is case-sensitive
   and does not normalize: `expiresAt` ≠ `expires_at` ≠ `expiresat`. The Go driver writes the field
   under the **bson tag** name; the index points at the **Keys** name; if they differ, the index sees
   `null` on every document and nothing expires.
2. **The field must serialize to BSON `Date`.** Go's `time.Time` marshals to BSON `Date` by default —
   good. `string` (ISO-8601) and `int64` (Unix epoch) marshal to BSON `String`/`Int64`, which the TTL
   monitor treats as "not a date value" and skips. Use `time.Time`.

Both failures are silent. Neither throws. The cost graph is your only late alarm — so catch it early.

## Canonical example

```go
import (
	"context"
	"time"

	"go.mongodb.org/mongo-driver/v2/bson"
	"go.mongodb.org/mongo-driver/v2/mongo"
	"go.mongodb.org/mongo-driver/v2/mongo/options"
)

// CRITICAL: the bson tag below MUST equal the index Keys key, and the type MUST be time.Time.
type SessionDoc struct {
	ID        bson.ObjectID `bson:"_id,omitempty"`
	Token     string        `bson:"token"`
	ExpiresAt time.Time     `bson:"expiresAt"` // BSON Date — NOT string, NOT int64
}

func EnsureSessionTTL(ctx context.Context, coll *mongo.Collection) error {
	_, err := coll.Indexes().CreateOne(ctx, mongo.IndexModel{
		Keys:    bson.D{{Key: "expiresAt", Value: 1}}, // MUST match `bson:"expiresAt"` exactly
		Options: options.Index().SetExpireAfterSeconds(0),
	})
	return err
}
```

`SetExpireAfterSeconds(0)` means "expire at the absolute timestamp stored in the field" (the doc dies
0–60s after `ExpiresAt`). A non-zero N means "expire N seconds after the field's Date." Both are
valid — choose deliberately; the value is `int32`.

### Cheap insurance: a reflection test

```go
// A unit test that asserts every TTL IndexModel's Keys matches a real bson tag on the struct
// fails in CI in milliseconds — far cheaper than a silent leak discovered weeks later in billing.
func assertTTLKeyMatchesTag[T any](keyName string) bool {
	// walk reflect.TypeOf(T).Field(i).Tag.Get("bson"), strip ",omitempty", compare to keyName
	// return true iff some field's bson tag == keyName
	return true // implement per your structs
}
```

## Anti-pattern to detect (greppable)

- A `Keys: bson.D{{Key: "...", ...}}` whose key has **no** exactly-matching `bson:"..."` tag on the
  persisting struct.
- A TTL-indexed field typed `string` or `int64` instead of `time.Time`.
- A field/tag **rename** that doesn't update the index `Keys` (or migration) in the same change.
- Relying on "MongoDB will error if it's wrong" — it won't.

## Decision aid

- **Absolute per-document expiry timestamp?** → store `time.Time`, index that field,
  `SetExpireAfterSeconds(0)`.
- **Fixed lifetime after a creation time?** → `SetExpireAfterSeconds(N)`, N>0, on the creation-time
  `time.Time`.
- **Tempted to store the timestamp as a string/epoch int?** → don't; TTL silently ignores non-Date.
- **Defense-in-depth?** → add MongoDB Schema Validation to reject writes missing the field (catches
  it at write time, not TTL time).

## Related skills

- `global-skills/aws-go/lambda-go-lazy-init-segregated/SKILL.md` — the Mongo client this index lives
  on should be initialized lazily behind its own `sync.Once`, off the cold-start happy path.

## Sources

- [MongoDB TTL Indexes](https://www.mongodb.com/docs/manual/core/index-ttl/) ("the document will not expire"; 60s monitor) · [Expire Data from Collections by Setting TTL](https://www.mongodb.com/docs/manual/tutorial/expire-data/)
- [mongo-driver/v2 — IndexModel / IndexView.CreateOne](https://pkg.go.dev/go.mongodb.org/mongo-driver/v2/mongo) · [options.Index().SetExpireAfterSeconds](https://pkg.go.dev/go.mongodb.org/mongo-driver/v2/mongo/options)
- [MongoDB Go Driver — Struct Tagging](https://www.mongodb.com/docs/drivers/go/current/usage-examples/struct-tagging/)

---

**Last verified:** 2026-06-03 against the MongoDB Manual (live — "If a document does not contain the
indexed field, the document will not expire"; TTL monitor every 60 seconds; no error path) and
`mongo-driver/v2` v2.6.0 (`mongo.IndexModel{Keys bson.D}`, `IndexView.CreateOne`,
`options.Index().SetExpireAfterSeconds(int32)`).
**Re-check after:** MongoDB Go driver v2 major / MongoDB server major, or by 2026-09-03.
**Decay risk:** low (TTL silent-skip semantics are long-stable).
**Found a drift?** Run `/skill-pattern-freshness-audit aws-go`.
