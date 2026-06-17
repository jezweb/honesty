# Honesty

**Audit a codebase so every comment, docstring, and doc file tells the truth about the code — for the next AI agent (or human) who reads it before changing anything.**

A comment is a promise about what the code does. When it drifts, it becomes *worse* than no comment:
the next reader trusts it, builds a wrong mental model, and ships a mistake. And the next reader is
increasingly an AI coding agent — it reads a docstring that says a function returns `null`, writes a
dead null-guard against a function that actually throws, and skips the `try/catch`. The comment meant
to help is the thing that makes it wrong.

Honesty finds those broken promises and either corrects them or flags them. It's a Claude Code plugin:
one skill that audits a codebase module by module and tells you exactly which comments lie, which are
missing, and which can be fixed automatically versus which need your eye.

## How it works

```
scope → bundle → audit (one agent per module) → structured findings →
  disposition (auto-fix vs review) → apply + verify → review queue → roll
```

It checks every surface the code makes claims on — inline + block comments, docstrings/JSDoc, doc
files (`README`, `CLAUDE.md`/`AGENTS.md`, `ARCHITECTURE.md`, `DATABASE_SCHEMA.md`), and config-file
comments — against a tight rubric:

- **inaccurate** — contradicts the code (wrong condition, default, return, or API contract)
- **stale** — wrong counts/dates/enumerations, done-but-still-"TODO", "temporary" that's now permanent,
  superseded behaviour, broken cross-references
- **misleading** — builds a wrong mental model
- **gap** — non-obvious / load-bearing code with no comment
- **unclear** — too vague to help

It's deliberately **high-signal** — an all-sound module returns nothing, and that's a good result.

## Auto-fix is gated, not eager

Confidence alone doesn't make a fix safe to apply — a confident edit to a contract docstring or to
`CLAUDE.md` can mislead far beyond one line. Honesty auto-applies a finding only when it's **high
confidence AND a mechanical factual correction AND local in blast radius AND not a doc/config file**.
Everything else — rewrites, gap-fills, contract docstrings, every doc-file change — goes to a review
queue with the suggested fix attached. New codebases default to findings-first; auto-fix is a knob you
turn on once a calibration batch proves the signal.

## What it is not

- Not a doc **generator** (pair it with one — honesty flags a stale `ARCHITECTURE.md`, the generator
  regenerates it).
- Not a comment-**stripper** — stale comments get *corrected*, not deleted; missing ones get *added*.
- Not a style linter, and it leaves user-facing strings alone (that's product behaviour, not developer
  truth).

One lane: **are the claims the code makes about itself true, and present where they matter.**

See [`docs/pattern.md`](docs/pattern.md) for the methodology, the disposition gate, and the evidence
from real runs (including a real bug found hiding behind a technically-true comment). The skill itself
is in [`skills/honesty/SKILL.md`](skills/honesty/SKILL.md); reusable audit prompt + findings schema are
in [`templates/`](templates/).

---

Part of a family of Claude Code plugins: [fixer](https://github.com/jezweb/fixer),
[perfection](https://github.com/jezweb/perfection), [walkabout](https://github.com/jezweb/walkabout).

MIT © Jeremy Dawes / Jezweb
