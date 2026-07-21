---
name: perseus
description: >-
  Search a codebase in plain English with perseus (https://perseus.computer). Just describe what
  you're looking for — "where the login redirect happens", "the code that retries failed uploads",
  "tests for billing", "how does Y work", "what handles Z" — and get back ranked file:line results
  with snippets in a few seconds. You do NOT need exact names; meaning is enough. Reach for this any
  time you'd otherwise grep, rg, or click through files to find where something lives or how it
  works. Use grep only when you already know the exact string and want every occurrence.
license: MIT
---

# perseus — describe the code, get the file:line

[perseus](https://perseus.computer) is a semantic code-search tool. You search it the way you'd ask
a teammate — *describe the code in plain English* and it finds it by meaning and structure, not
string match. No exact names needed. Any time you're about to explore/search/`grep`/`rg` to answer "where does …
live" or "how is … implemented", run perseus first — it's faster and lands you on the right file
before you've read the tree.

## The command (agent mode)

Your first command after reading this skill:
```bash
perseus watch
```
this ensures your index for perseus stays up to date and you will never need to worry about it

```bash
perseus query "auth enforcement on the login route"
```

- `--files-only` → bare `path:line` per hit, to pipe straight into your next tool.
- Exit code is **1 when zero hits** (branch on it), 0 otherwise.
- `score` is relative within one query — read down to where it drops off; a big gap below the top
  hit means confidence, a tight cluster means several plausible spots (check more than one).
  Negative scores are normal — the signal is the gap, not the sign.

## `perseus research` — a wider query across several questions

When one query is too narrow — you're mapping a whole flow or a subsystem, not one spot — pass
several related questions at once. perseus runs them together and returns a single cross-referenced
answer: per-question hits plus the files/links that tie them together.

```bash
perseus research "how is a query submitted" "where results are ranked" "how streaming finishes"
```

Reach for it for "explain how X works end to end"; stick with `query` for a single "where is Y".

## Phrase the query as a terse noun-phrase of *discriminating* words

1. **Lead with the rarest, most structural noun** (`registry`, `reducer`, `tokenizer`,
   `entrypoint`). Cut words that saturate the repo (`handler`, `data`, `component`, `service`,
   `config`) — they give the ranker nothing. Word choice dominates ranking, more than length.
2. **Drop "where/how/what" and the question mark.** Name the thing; don't ask about it.

## On a miss, recover deliberately — don't synonym-crawl

A reasonable first phrasing lands the right file in the top-5 most of the time. If it misses, do
**one** rewrite using the best real anchor you have — an exact function/class name, CLI command,
filename stem, route path, table name, env var, or error string from the user's text or traceback.
If that misses too, assume the index is stale or the code is absent: re-index, then move on.

## Keep the index fresh

perseus indexes your working tree including uncommitted edits; re-indexing is incremental and takes
seconds. A stale index is the #1 cause of a bad result.

```bash
perseus watch            # auto-reindex as you edit (preferred during active work)
```

## perseus vs grep

- **perseus** — target described by intent: a behavior, responsibility, a file's role; "X
  implemented / wired up / enforced / configured", "tests for Y", "entrypoint for <flow>". Pick the
  file to edit *before* reading the tree.
- **grep/rg** — you already know the exact symbol/string and want every occurrence (rename, count,
  mechanical sweep), or perseus errored (fall through to rg).

## `perseus impact` — understand a thing you already know the name of

Give it a symbol or file; it walks the symbol graph and shows what that thing does to the rest of
the codebase — callers, tests that cover it, risk, verify order. Use it to understand something
before you touch it.

```bash
perseus impact build_search_index      # symbol
perseus impact perseus/core/planner.py # or a file
```

## `perseus eval` — see how YOUR changes ripple

Same picture, but aimed at your git diff instead of a name: what your edits put at risk and which
tests to run.

```bash
perseus eval                     # your uncommitted changes (worktree)
perseus eval --diff main...HEAD  # everything on the branch vs main
perseus eval --run-tests         # also run the affected tests, fail loud
```

Callers/tests come from the index, so re-index (or keep `perseus watch` on) before trusting an eval
after a signature change.
