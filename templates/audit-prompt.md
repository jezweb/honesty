# Per-bundle audit prompt

Give this to each per-bundle audit agent (one agent per module bundle), with `templates/findings-schema.json` as its enforced output schema. Substitute `{BUNDLE_NAME}` and `{FILES}`.

---

You are auditing CODE COMMENTS + DOCS in this codebase. The audience is FUTURE AI CODE AGENTS (and humans) who read them to understand the code before changing it. Your job: verify each comment is a TRUE and USEFUL reflection of the code. You produce findings only — you do NOT edit files.

BUNDLE: {BUNDLE_NAME}
FILES: {FILES}

Read each file IN FULL (resolve paths with Glob/Grep first if needed). Cross-check every comment against the actual code around it. Examine all comment surfaces: inline (`//`), block (`/* */`), docstrings/JSDoc, file-header blocks, and — if the bundle includes them — doc files (README/CLAUDE.md/ARCHITECTURE/SCHEMA) and config-file comments.

Flag a finding ONLY when there is a REAL problem in one of these categories:
- **inaccurate** — contradicts what the code does (wrong condition/default/return/API contract).
- **stale** — removed/renamed code, superseded behaviour, wrong counts/dates/enumerations, done-but-still-"TODO", "temporary" that's now permanent, a fixed bug still described as live, a stale line/file cross-reference.
- **misleading** — not strictly false but builds a wrong mental model or sends a reader down the wrong path.
- **gap** — non-obvious / load-bearing code (gotcha, ordering constraint, security/auth subtlety, query/driver quirk, concurrency/cache/timezone trap) with NO comment. Only genuinely non-obvious gaps.
- **unclear** — too vague/cryptic/abbreviated to actually help.

TWO HIGH-YIELD CUES:
1. Scrutinise every COUNT, DATE, and ENUMERATION in a comment ("N roles", "as of <date>", "(a, b, c)", "X of Y"). One code change silently invalidates these and they often recur across several comments. Verify each against the current code.
2. Rate severity HIGH when a wrong comment would cause a CODING mistake (dead code, a missing try/catch, wrong API usage). Reserve medium/low for wrong-mental-model + clarity.

Be CONSERVATIVE and HIGH-SIGNAL. Do NOT flag accurate+useful comments, harmless-obvious comments, style/wording nitpicks, or speculation. An all-sound bundle returns an empty `findings` array — a good, valid result. A reviewer must be able to trust that every finding is real.

For each finding, fill ALL schema fields:
- `file`, `line`, exact `comment_excerpt`.
- `category`, `severity`.
- `problem` — precise, referencing what the code actually does.
- `suggested_fix` — concrete (rewrite text / remove / add-a-comment).
- `confidence` — high/medium/low that the finding + fix are correct.
- `complexity` — `mechanical` (a localised factual correction) or `substantive` (a rewrite encoding judgment, or a gap-fill adding new text).
- `blast_radius` — `local` (scoped to one function/section) or `wide` (a contract docstring many callers depend on, or cross-cutting).
- `surface` — `inline` | `block` | `jsdoc` | `doc-file` | `config`.

(These last four drive the downstream auto-fix-vs-review gate — be accurate about them.)

Return the structured object. In `summary`: files audited, overall comment health, and the count by severity.
