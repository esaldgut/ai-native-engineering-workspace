# Contributing

This repository follows **Git Flow** with branch protection and a Conventional Commits history.
The same discipline that governs the engineering documented here governs the repository itself.

---

## Branching model (Git Flow)

Two permanent branches, three kinds of supporting branch.

```
main ─────●──────────────●──▶   release tags (v1.0, v1.1, …)
           \            /
develop ─●───●───●───●──●──▶     integration branch (default)
           \   /
 feature/x  ●─●                  short-lived, merged via PR
```

| Branch | Lives | Base | Merges into | Purpose |
|--------|-------|------|-------------|---------|
| `main` | permanent | — | — | Released, tagged state. Every commit is a release. |
| `develop` | permanent | — | `main` (via release) | Integration. The repository's **default branch**; day-to-day work lands here. |
| `feature/<slug>` | short-lived | `develop` | `develop` | A new skill, doc, or change. Deleted after merge. |
| `release/<version>` | short-lived | `develop` | `main` **and** `develop` | Version stabilization (final polish, version bump) before a tag. |
| `hotfix/<version>` | short-lived | `main` | `main` **and** `develop` | An urgent fix to a released state. |

### Branch naming

- `feature/extract-android-paging-skill`
- `release/v1.1`
- `hotfix/v1.0.1`

Lowercase, hyphen-separated, prefixed by type. The slug names *what changes*, not who changes it.

---

## Branch protection

Both permanent branches are protected. The asymmetry is intentional.

| Rule | `main` | `develop` |
|------|:------:|:---------:|
| Pull request required to merge | ✅ | — |
| Linear history (no merge commits) | ✅ | — |
| Force-push blocked | ✅ | ✅ |
| Deletion blocked | ✅ | ✅ |

`main` represents tagged releases, so everything reaching it goes through a pull request and keeps
a linear history — clean to tag and to read. `develop` is the integration buffer: protected against
rewrites and deletion, but it accepts direct pushes so the daily `feature/* → develop` loop is not
slowed down.

---

## Workflow

### A feature

```bash
git checkout develop && git pull
git checkout -b feature/<slug>
# … work, commit …
git push -u origin feature/<slug>
gh pr create --base develop --fill
```

After the PR merges, delete the branch (`gh pr merge --squash --delete-branch` does both).

### A release

When `develop` is ready to become a new version:

```bash
git checkout -b release/v1.1 develop
# … version bump, final polish …
git push -u origin release/v1.1
gh pr create --base main --fill            # release/v1.1 → main (PR required)
# after merge:
git tag -a v1.1 -m "Release v1.1" && git push origin v1.1
git checkout develop && git merge main     # back-merge so develop has the release commit
git push origin develop
```

### A hotfix

```bash
git checkout -b hotfix/v1.0.1 main
# … fix …
gh pr create --base main --fill            # hotfix → main
# after merge: tag, then back-merge into develop (as above)
```

Both `release/*` and `hotfix/*` merge into **both** `main` and `develop`, so the two permanent
branches never diverge.

---

## Commit messages — Conventional Commits

The history follows [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): subject`.

```
feat(skills): extract spanish-technical-tone-canon (skill #42)
fix(nda): purge pre-existing leaks surfaced by widened NDA gate
docs: tone pass on root README — remove AI-smell, tighten claims
chore: bootstrap workspace
```

| Type | Use for |
|------|---------|
| `feat` | A new skill, doc, or capability |
| `fix` | A correction to existing content (including NDA-safety fixes) |
| `docs` | README / navigation / narrative changes |
| `chore` | Tooling, structure, repo housekeeping |
| `refactor` | Restructuring without changing meaning |

Scope is the area touched: `skills`, `reference`, `docs`, `nda`, a domain name. Subject is
imperative and lowercase. The body (optional) explains *why*, not *what* — the diff already shows
what.

---

## Two repository-specific rules

This repository carries two constraints that ordinary repos don't. Both are non-negotiable.

### 1. NDA-safety

Everything here is a **generic extraction** from production engineering — never an extract of any
private codebase. No client or employer identifiers, real account IDs, bundle IDs, team IDs,
internal endpoints, or personal paths. Any contribution runs through a pre-merge scan for those
classes of identifier; a hit blocks the merge. Examples use placeholders (`MyApp`,
`com.example.app`, `auth.example.com`, `<region>`, `<AWS_ACCOUNT_ID>`).

When in doubt, genericize. A pattern is publishable only when removing every proper noun still
leaves something useful — and where it doesn't, it stays private. See the coverage matrix in
[`global-skills/README.md`](global-skills/README.md) for how that line is drawn.

### 2. Freshness

Every skill carries a `freshness` block in its frontmatter: cited sources with versions, a
verification date, a re-check trigger, and a `status`. A new or edited skill must:

- cite **at least two** sources, one of them a primary vendor or standards doc, each with a
  resolving URL and a version;
- name **at least one** concrete public API in its body (the anti-sterilization rule — it anchors
  the pattern and gives the freshness audit a literal symbol to re-check);
- set `verified_on` to the date it was checked and `status` accordingly.

The contract is defined in [`global-skills/FRESHNESS_SPEC.md`](global-skills/FRESHNESS_SPEC.md).
Run `/skill-pattern-freshness-audit <domain>` before relying on a skill, and
`dossier-driven-skill-update` to repair what the audit flags.

---

## Before you open a pull request

- [ ] Branch is `feature/*`, `release/*`, or `hotfix/*` off the right base.
- [ ] Commits follow `type(scope): subject`.
- [ ] No private identifiers anywhere in the diff (NDA-safety).
- [ ] If a skill changed: two cited sources, one concrete public API, `freshness` updated.
- [ ] PR base is `develop` (features) or `main` (releases / hotfixes).
