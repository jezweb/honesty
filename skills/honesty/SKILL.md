---
name: honesty
description: Audit a codebase so every comment, docstring, and doc file is a TRUE and useful reflection of the code — for the next AI agent (or human) who reads it before changing anything. Finds stale, inaccurate, misleading, and missing comments (and stale README/CLAUDE.md/ARCHITECTURE/config notes), produces structured findings, and splits them into a confidence/complexity/blast-radius-gated auto-fix set vs a human-review queue. Module-by-module, high-signal, never a comment-stripper. Use when the user wants to verify comments match the code, clean up stale/misleading comments or docs, make a codebase trustworthy for AI agents, or asks to "run honesty". Triggers: honesty, comment audit, do the comments match the code, are the comments accurate, stale comments, trustworthy comments, audit the docs against the code, comment truth.
---

# Honesty — make the codebase tell the truth

> Part of the **shipwright** method — see the `shipwright` skill for the work-loop this move fits into and when to invoke it.

A comment is a promise about what the code does. When it drifts, it becomes worse than no comment:
the next reader — increasingly an **AI agent** — *trusts it*, builds a wrong mental model, and ships a
mistake. Honesty audits every claim the codebase makes about itself and either corrects it or flags
it, so the promises hold.

The organising principle:

> **Every claim the codebase makes about itself should be true — and present where it matters — for the next agent or human who reads it before changing the code.**

This is not a doc *generator* (see project-docs for that) and not a style linter. It verifies the
**truth** of what already exists, and adds a comment only where its absence would trip a future reader.

## The surfaces it audits

Comments live in more than `//` lines. Honesty covers all the places the code makes claims:

1. **Inline + block comments** — `//`, `/* */`, the per-function "why" notes.
2. **Docstrings / JSDoc** — the API contracts. Highest-value class: a wrong contract (`@returns null`
   when the function throws) causes *coding* mistakes, not just confusion.
3. **Doc files** — `README`, **`CLAUDE.md`/`AGENTS.md`** (the first thing an agent reads — a stale one
   misleads on every task), `ARCHITECTURE.md`, `DATABASE_SCHEMA.md`, API docs.
4. **Config-file comments** — `wrangler.jsonc`, CI YAML, Dockerfiles, etc. They drift silently.

Out of scope: user-facing strings (that's product behaviour, not developer-facing truth), commit
messages, and pure style/wording.

## The loop

```
scope → bundle → audit (one agent per bundle) → structured findings →
  disposition (auto-fix vs review) → apply + verify → review queue → roll
```

1. **Scope.** Map the repo: file count, LOC, comment-line density per area. The load-bearing "why"
   comments cluster in core/server/shared (gotchas, auth, data access, integration quirks); UI is
   usually comment-light + self-evident. Prioritise the dense, high-traffic areas. Always include the
   doc files (CLAUDE.md/README/ARCHITECTURE) — they're the highest-leverage agent-facing surface.
2. **Bundle.** Group related files into module bundles sized for one agent to read carefully (a route
   + its helpers; a middleware dir; the shared contracts; one doc file). Don't fire one giant fleet —
   roll bundle by bundle so each batch can be reviewed.
3. **Audit.** One agent per bundle reads each file IN FULL and cross-checks every comment against the
   code, using the rubric below. Conservative + high-signal: an all-sound bundle returns zero findings,
   and that is a good result. The failure mode is a noisy report nobody trusts.
4. **Findings.** Structured per finding (see `templates/findings-schema.json`): file, line, category,
   severity, confidence, complexity, blast_radius, surface, the exact comment excerpt, the precise
   problem (what the code actually does), and a concrete suggested fix.
5. **Disposition** (the gate — below): each finding is routed to **auto-fix** or **review**.
6. **Apply + verify.** Auto-fix findings are applied as comment-only edits (a subagent can do this
   mechanically); then run the build/typecheck — comment-only edits must not change it.
7. **Review queue.** Everything else is handed to the human/lead with the suggested fix, applied after
   they read it. Roll to the next bundle.

## The rubric — flag only real problems

- **inaccurate** — the comment contradicts what the code does (wrong condition, default, return, or
  **API contract**). The dangerous class.
- **stale** — removed/renamed code, superseded behaviour, wrong **counts / dates / enumerations**,
  done-but-still-"TODO", "temporary" that's now permanent, a fixed bug still described as live, a stale
  line/file cross-reference.
- **misleading** — not strictly false, but builds a wrong mental model or sends a reader down the wrong path.
- **gap** — non-obvious, surprising, or load-bearing code (a gotcha, ordering constraint, security/auth
  subtlety, query/driver quirk, concurrency/cache/timezone trap) with **no** comment.
- **unclear** — too vague, cryptic, or abbreviated to actually help.

Two cues that earn their keep (from real runs):
- **Scrutinise counts, dates, and enumerations** in comments ("5 built-in roles", "as of <date>",
  "(a, b, c)", "X of Y mapped"). They are the single highest-yield drift target — one code change
  silently invalidates them, and they often appear in several comments at once. Verify each against
  the current code.
- **Rate severity *high* when a wrong comment would cause a *coding* mistake** — dead code from a wrong
  null-guard, a missing `try/catch` from a wrong "throws/returns" doc, wrong API usage. Reserve
  medium/low for wrong-mental-model and clarity issues.

Do **not** flag: accurate + useful comments, harmless-obvious comments, style/wording nitpicks, or
speculation. Every finding must be verifiable against the code.

## The disposition gate — auto-fix vs review

Confidence alone is not enough to auto-apply: a confident fix to a contract or a doc file can still
ripple. Route a finding to **auto-fix** only when **all four** hold:

| Factor | Auto-fix requires |
|---|---|
| **confidence** | `high` — certain the comment is wrong AND the fix is right |
| **complexity** | `mechanical` — a localised factual correction (count/date/enum/line-ref/name/clearly-wrong clause). NOT a rewrite that encodes judgment, NOT a gap-fill (adding new explanatory text), and NOT a correction to a **code or usage example** in a comment/docstring — those are `substantive` even when they look like a factual fix, because the replacement encodes API knowledge that must be verified against the real exports/signatures first (a "fixed" example can itself be subtly wrong) |
| **blast_radius** | `local` — an inline/block comment scoped to one function/section. NOT a contract-defining docstring many callers depend on, NOT cross-cutting |
| **surface** | inline/block/jsdoc-local only. **Doc files (CLAUDE.md/README/ARCHITECTURE/SCHEMA) and config files are NEVER auto-fixed** — high blast radius, often human-owned |

Anything failing any factor → **review queue**. In practice the auto-fix set is the safe mechanical
class — stale counts/dates/enumerations, wrong line-refs, an obviously-wrong factual clause on a local
comment, at high confidence. Gap-fills, rewrites, contract docstrings, and every doc-file change get a
human's eye. Default the whole skill to **findings-first** on a new codebase; enable auto-fix once a
calibration batch has shown the signal is clean. The auto-fix threshold is a knob, not a default-on.

## The discipline (what makes it *honesty*, not vandalism)

- **Gap-filling, not just lie-deleting.** Adding the missing gotcha-comment is half the value. Making
  the truth *present* matters as much as removing the false.
- **Never a comment-stripper.** Deleting all comments would "pass" a naive audit and destroy context.
  Removing a comment is only correct when the comment is pure noise; a stale comment gets *corrected*.
- **Comment-only edits.** A fix changes comment characters only — never code, logic, identifiers, or
  control flow. Verify with a diff (no non-comment line changed) + a clean build.
- **Conservative, high-signal.** Trust is the product. One bogus finding and the report gets ignored.
- **A finding can surface a real bug — a false comment is a lead, not just noise.** Two shapes recur:
  (1) a comment that is *technically true but hides a divergence* (two code paths that should agree but
  don't); and (2) a comment that describes a **guard or behaviour that doesn't exist** — e.g. "only runs
  when X and not when already-picked" where no such guard is in the code, so the missing guard is itself
  the bug. In both, correct the comment to match reality AND file an issue for the real fix — don't
  silently bury it, and don't "fix" the comment by inventing the behaviour it claims.

## Modes / depth

- **`quick`** — the high-leverage surfaces only: `CLAUDE.md`/`README` + docstrings + file headers.
- **`standard`** — + all inline comments, bundle by bundle, over the chosen area.
- **`thorough`** — + doc files + config comments, whole codebase, rolled one bundle per pass.

## Running it

1. Scope the repo (`docs/pattern.md` has the heuristics). Pick the area + depth.
2. Bundle the files. For anything beyond a few files, fan out **one agent per bundle** (a workflow or
   parallel subagents) with the prompt in `templates/audit-prompt.md` and the schema in
   `templates/findings-schema.json`.
3. Collect findings; split by the disposition gate.
4. Apply the auto-fix set (comment-only) + verify the build. Hand the review queue to the human.
5. Roll to the next bundle. Track coverage so you know what's been audited.

See [`docs/pattern.md`](../../docs/pattern.md) for why this resists the usual failure modes (noisy
reports, comment-stripping, auto-fix that ripples) and the evidence behind the rubric.
