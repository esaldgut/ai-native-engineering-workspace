# meta — skills about skills

3 skills that operate on the skill library itself. Together they form a closed lifecycle:
**create → monitor → repair**. This is what makes the rest of the repo maintainable rather than a
static snapshot.

| Skill | Role | What it does |
|-------|------|--------------|
| [`skill-extraction-pattern`](skill-extraction-pattern/SKILL.md) | **Create** | Extract public, reusable skills from private NDA-locked ones — a 7-step pipeline (inventory → classify → research-verify → rewrite-from-scratch → re-classify → anchor with a concrete example → self-applicability check). Grounded in GoF (1994) + Anthropic skill best practices. |
| [`skill-pattern-freshness-audit`](skill-pattern-freshness-audit/SKILL.md) | **Monitor** | Detect (don't repair) drift between a skill and the current docs it cites. Reads the `freshness` block, re-verifies each API live, flags `current` / `needs-recheck` / `stale` / `superseded` with citations. Read-only and cheap (writes status only). |
| [`dossier-driven-skill-update`](dossier-driven-skill-update/SKILL.md) | **Repair** | Fix a skill the audit flagged — focused re-research of just the drifted APIs + a localized rewrite + provenance refresh. Explicit-invocation only (`disable-model-invocation: true`). |

```
skill-extraction-pattern      →  create  (private → public, generically)
        ↓
skill-pattern-freshness-audit →  monitor (drift = a queryable signal)
        ↓  flags stale / superseded
dossier-driven-skill-update   →  repair  (re-research + rewrite + re-verify)
        ↓
        └────────────────────►  back to monitor: confirm current
```

The contract these enforce is [`../FRESHNESS_SPEC.md`](../FRESHNESS_SPEC.md).

> `skill-extraction-pattern` is **self-applicable** — this entire repo's skill library was produced
> by applying it to a private corpus, and it could extract a future version of itself. That recursion
> is step 7 of its own pipeline.

---

**Freshness:** re-check when the methodology or the Claude Code skill format changes, or by
**2026-12-03**. Run `/skill-pattern-freshness-audit meta`.
