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
string match. No exact names needed. Any time you're about to `grep`/`rg` to answer "where does …
live" or "how is … implemented", run perseus first — it's faster and lands you on the right file
before you've read the tree.

## The command (agent mode)

```bash
perseus query --no-summary --json "auth enforcement on the login route" | jq '.hits[]'
```

- `--no-summary` → ranked locations only (skips prose). `--json` → clean JSON on stdout (status on
  stderr), so piping to `jq` works. Each hit in `.hits[]` has `path`, `line_start`, `line_end`,
  `score`, `snippet`. Read the top few, then act.
- `--files-only` → bare `path:line` per hit, to pipe straight into your next tool.
- Exit code is **1 when zero hits** (branch on it), 0 otherwise.
- `-k 15` widens the candidate set if the first pass misses.
- `score` is relative within one query — read down to where it drops off; a big gap below the top
  hit means confidence, a tight cluster means several plausible spots (check more than one).
  Negative scores are normal — the signal is the gap, not the sign.

## `perseus research` — one wider query across several linked questions

When one query is too narrow — you're mapping a whole flow or subsystem, not one spot — ask 2-8
related questions at once and get a single cross-referenced answer: per-question hits plus the
load-bearing files and symbol-graph links that tie them together, in a dependency-ordered reading
list.

```bash
perseus research "where is auth enforced" "where are sessions created" "how logout clears state"
```

- `--files-only` → just the reading list as `path:line`; `--json` → the full packet; `-k` → hits
  per question. Use `research` for "explain how X works end to end"; plain `query` for a single
  "where is Y".

## Phrase the query as a terse noun-phrase of *discriminating* words

1. **Lead with the rarest, most structural noun** (`registry`, `reducer`, `tokenizer`,
   `entrypoint`). Cut words that saturate the repo (`handler`, `data`, `component`, `service`,
   `config`) — they give the ranker nothing. Word choice dominates ranking, more than length.
2. **Drop "where/how/what" and the question mark.** Name the thing; don't ask about it.
3. **Use the real symbol name when you know it** (`useLeadsQuery`, `build_search_index`) — exact
   identifiers are the most reliable phrasing there is. But **never fabricate** one you don't know:
   a wrong concept word steers ranking into the wrong subsystem. Omitting beats guessing.
4. **UI/product flows need product-surface anchors.** User language names the job ("invite
   teammate"); code names the surface ("Collaborators", `org-members`). If a job-phrase misses,
   rewrite once toward a nav label, route stem, or component name.

| weak (generic / vague)                   | strong (distinctive noun)            |
| ---------------------------------------- | ------------------------------------ |
| `how does auth work on the login route?` | `auth enforcement on the login route`|
| `react query fetch leads list`           | `useLeadsQuery`                       |
| `team member invite form`                | `collaborators panel`                |

## On a miss, recover deliberately — don't synonym-crawl

A reasonable first phrasing lands the right file in the top-5 most of the time. If it misses, do
**one** rewrite using the best real anchor you have — an exact function/class name, CLI command,
filename stem, route path, table name, env var, or error string from the user's text or traceback.
If that misses too, assume the index is stale or the code is absent: re-index, then move on.

## Keep the index fresh

perseus indexes your working tree including uncommitted edits; re-indexing is incremental and takes
seconds. A stale index is the #1 cause of a bad result.

```bash
perseus index            # re-index the current checkout
perseus watch            # auto-reindex as you edit (preferred during active work)
```

## perseus vs grep

- **perseus** — target described by intent: a behavior, responsibility, a file's role; "X
  implemented / wired up / enforced / configured", "tests for Y", "entrypoint for <flow>". Pick the
  file to edit *before* reading the tree.
- **grep/rg** — you already know the exact symbol/string and want every occurrence (rename, count,
  mechanical sweep), or perseus errored (fall through to rg).

## `perseus impact` — understand a thing once you know its name

`query` finds *where* code lives; **impact** tells you what a named thing *does to the rest of the
codebase*. Give it a symbol (or file) and it walks the symbol graph for callers, the tests that
cover it, obligations, risk, and a verification order — so you understand something before you
touch it.

```bash
perseus impact build_search_index          # one symbol
perseus impact src/planner.py              # or a whole file
perseus impact submit_query resolve_index  # several at once
```

Static — keyed on the name you give, not on any change.

## `perseus eval` — see how YOUR changes ripple through the chain

**eval** is impact aimed at your git **diff** instead of a name. It reads what you changed, maps
those edited lines to the symbols they touch, and reports what the changes put at risk: callers,
the tests you should run, and a verification order. The pre-commit / pre-PR "blast radius of my
branch" check.

```bash
perseus eval                     # your uncommitted changes (worktree)
perseus eval --diff main...HEAD  # everything on the branch vs main
perseus eval --run-tests         # derive the affected tests AND run them (fails loud)
```

- No `--diff` = worktree edits on top of HEAD; `--diff <range>` scopes to a committed range.
- Callers/tests come from the index, so re-index (or keep `perseus watch` on) before trusting an
  eval after a signature change.

Shared flags (impact + eval): `--print-tests` prints the derived test commands; `--run-tests` runs
them and fails on a break; `--files-only` / `--json` for machine output.
