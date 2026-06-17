# The pattern — why a comment audit needs more than "find wrong comments"

## The problem it solves

A comment is a promise about what the code does. Codebases accrete broken promises: a count that was
right when written ("5 roles") and silently wrong after the sixth was added; a "TODO" that's done; a
"temporary" workaround that's now load-bearing; a docstring that says a function returns `null` when it
now throws. Each broken promise is **worse than no comment**, because the next reader trusts it.

That reader is increasingly an **AI coding agent**. An agent reads the comment, builds a mental model,
and acts on it — writes a dead `if (!x)` null-guard against a function that throws, skips a `try/catch`,
"fixes" the wrong code path, or reasons from a stale count. The comment that was meant to *help* the
agent is the thing that makes it wrong. Honesty exists to make the promises hold.

## Why the naive version fails

"Ask an agent to check the comments" goes wrong three predictable ways. The skill is built to dodge
all three:

1. **Noise.** An unconstrained audit flags every comment it could "improve" — style, wording,
   obvious-but-harmless notes. The report balloons, the real findings drown, and nobody trusts it. The
   fix: a tight 5-category rubric that flags only *inaccurate / stale / misleading / gap / unclear*, an
   explicit "do not flag accurate-and-useful comments," and "an empty findings list is a good result."
   High-signal beats exhaustive.
2. **Comment-stripping.** "Make the comments not-wrong" has a degenerate solution: delete them. A naive
   pass that removes anything it can't verify destroys the context it was meant to protect. The fix:
   *correct* stale comments, only *remove* pure noise, and treat **gap-filling** (adding a missing
   gotcha-comment) as half the job. Honesty makes the truth *present*, not just removes the false.
3. **Auto-fix that ripples.** "High confidence → just apply it" is unsafe: a confident edit to a
   contract docstring (which many callers read) or to `CLAUDE.md` (which every agent reads) can mislead
   far beyond the one line. The fix: a **multi-factor disposition gate**, below.

## The disposition gate

Whether a finding is safe to auto-apply is not one variable. It's four:

- **confidence** — is the finding + fix certainly right?
- **complexity** — is the fix a *mechanical* factual correction (a count, date, enumeration, line-ref,
  clearly-wrong clause), or a *substantive* rewrite / gap-fill that encodes judgment?
- **blast_radius** — is the comment *local* to one function, or a *wide* contract many callers depend on?
- **surface** — an inline comment, or a doc file (`CLAUDE.md`/`README`/`ARCHITECTURE`) the whole project
  trusts?

Auto-fix only when **all four** are safe: high confidence, mechanical, local, and not a doc/config file.
Everything else goes to a human-review queue with the suggested fix attached. In practice the auto-fix
set is exactly the safe mechanical class — stale counts, wrong dates, broken cross-references — and
every rewrite, gap-fill, contract docstring, and doc-file change gets a human's eye. On a fresh
codebase the whole skill defaults to **findings-first**; auto-fix is a knob you turn on after a
calibration batch proves the signal.

## What it catches (evidence from real runs)

Run on a production codebase, module by module, the audit produced **only real findings — no false
positives** across the calibration batches (verified against ground-truth knowledge of the files). The
classes that carried the value:

- **Self-contradicting / wrong API docs.** A function documented "returns `null` if X" that actually
  *throws* — its own next line even said "throws". The dangerous class: it directly causes coding
  mistakes (dead null-guards, missing error handling). Worth rating *high*.
- **Count / enumeration drift.** "5 built-in roles (a, b, c, d, e)" — hardcoded in **three** separate
  comments, all stale after a sixth was added by one change months earlier. Numbers and lists in
  comments are the highest-yield drift target; one edit invalidates them and they recur.
- **Stale scope / config notes.** A file header still describing the file as "houses only X" after it
  grew four responsibilities; a recipient list that omitted a role the SQL actually included; a config
  comment describing a migration step already done.
- **A real bug hiding behind a true comment.** A "// get cost map from D1" comment that was *technically
  accurate* but concealed that one screen computed a headline metric from a retired snapshot while the
  rest of the app used a live source — so the two screens silently disagreed. Honesty corrected the
  comment **and** filed an issue for the actual divergence. A comment audit that takes the comment's
  framing at face value misses this; one that checks the comment against *all* the code finds it.

The through-line: the value isn't cosmetic. Trustworthy comments make every future change — especially
an agent's — faster and safer, and the audit surfaces genuine correctness issues along the way.

## Scope discipline

Honesty audits **developer/agent-facing truth**: inline + block comments, docstrings, doc files
(`README`/`CLAUDE.md`/`ARCHITECTURE`/`SCHEMA`), and config-file comments. It is **not** a doc generator
(pair it with a docs-generation skill — honesty flags a stale `ARCHITECTURE.md`, the generator
regenerates it), **not** a style linter, and it leaves user-facing strings alone (those are product
behaviour, not developer truth). One lane: are the claims the code makes about itself true, and present
where they matter.
