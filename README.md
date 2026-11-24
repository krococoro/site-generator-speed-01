# site-generator-speed-01 — faster drop-in for makefun-site 1.0.0

[![make.fun · site-generator-speed-01-run-20261122](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/krococoro/site-generator-speed-01/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/krococoro/site-generator-speed-01/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)

A faster drop-in for make.fun's own `makefun-site` 1.0.0: `solve.js` is a
dependency-free ES module whose `run(input)` returns exactly the output tree
`makefun-site` 1.0.0 returns for the same input — every path and its text,
byte for byte — and `main.ts` is the harness driver (reads
`/input/cases.json`, answers each case with `run`, writes
`/output/answers.json`).

[View the bounty on make.fun](https://make.fun/bounty/site-generator-speed-01-run-20261122)

## Design

One file, no dependencies, nothing cached from one case to the next (every
per-case cache is created inside `run()`). The reference semantics are
reimplemented from scratch, with the hot paths rewritten:

- **Inline Markdown** renders plain runs in bulk: it jumps from one inline
  construct to the next with `indexOf` and escapes each run once, instead of
  one regex replace per character.
- **Markdown lexer** is a single fused pass over the lines (no intermediate
  token list) and dispatches on each line's first character, running a regex
  only when that line could match.
- **Front matter** is scanned by line offsets; the body is a slice, never a
  split-and-rejoin of the whole source.
- **Slugs** are memoised per case (titles and tag names repeat a lot).
- **Templates** are parsed once per case, then compiled to small functions
  (`new Function`; the generated code only ever embeds JSON string literals
  and paths the parser already validated, so template text cannot inject
  code). Compiled templates build their result as one string, reuse one
  scope frame per loop (mutating `index`/`first`/`last` in place), and fall
  back to the tree-walking interpreter when code generation is unavailable.
- **Word count** for `reading_minutes` uses a manual scan with a regex
  fallback only when exotic whitespace is present.

## Practice results

Kit scorer on the practice set (`grade-local` equivalent: driver +
`program/scorer/main.ts`): **passed=1, 3/3 cases identical**. Two driver runs
produce byte-identical `answers.json`.

Fuzzed against the reference on the kit's own generator
(`program/generator/subject/sites.mjs`) over 915 generated cases across 305
seeds, plus 16 hand-crafted adversarial cases (front-matter torture, Markdown
torture, bad templates, 10-deep includes, `for loop in …`, leap dates, CRLF):
all identical, byte for byte.

## Local timing (this machine, Node v24.22.1)

CPU time for `run()` over the 3 practice cases (5 reps, medians of three
interleaved rounds):

| | median CPU |
|---|---:|
| this entry | ~207–249 ms |
| house baseline (`makefun-site` 1.0.0) | ~586–615 ms |

About **2.7× faster** than the house baseline on the practice set. (The
sandbox runs Node 22 on 1 CPU; ratios there may differ.)
