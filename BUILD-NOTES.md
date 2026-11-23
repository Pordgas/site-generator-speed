# Build notes — fast drop-in for makefun-site 1.0.0

Entry: `solve.js` (dependency-free ES module, `run(input)`) + `main.ts` (the harness driver, reads
`/input/cases.json`, writes `/output/answers.json`). Repo: `Pordgas/site-generator-speed` (planned name).

## Practice results

- Kit scorer (`program/scorer/main.ts`): **3 of 3 cases pass** (`passed: 1, cases_passed: 3, cases_total: 3`).
- `grade-local --driver process`: 3 of 3, **L1 regrade reproduced byte for byte**.
- `answers.json` sha256 identical to the reference's output (`25980928b1…`).
- Determinism: two runs on practice write byte-identical `answers.json`.

## Timing (this machine, practice set, end-to-end `node main.ts`, alternating runs)

| run | mine | baseline (makefun-site 1.0.0) |
|-----|------|-------------------------------|
| 1 | 646 ms | 994 ms |
| 2 | 545 ms | 948 ms |
| 3 | 655 ms | 937 ms |
| 4 | 524 ms | 912 ms |
| 5 | 580 ms | 940 ms |
| 6 | 641 ms | 1017 ms |
| 7 | 538 ms | 1339 ms (load spike) |
| **median** | **~580 ms** | **~948 ms** |

**Speedup ≈ 1.63× end-to-end.** Build-only (warm, excluding JSON I/O and startup): mine ~193 ms vs
baseline ~450 ms (~2.3×). Fixed overhead (~210 ms: node startup, reading + parsing 2.9 MB cases.json,
stringifying + writing 7.1 MB answers.json) is the same for both.

## Differential testing

- Kit generator, fresh seeds: **156 generated sites, 0 mismatches** vs the reference.
- Hand-crafted edge cases: **220 checks, 0 mismatches** (front-matter corners: leap dates, quoted-string
  escapes, bad slugs, duplicate/unknown keys, unclosed matter, CRLF; unicode slugs incl. fullwidth/ligature/
  Greek/Cyrillic/CJK; duplicate headings; fence/hr/list/blockquote corners; link resolution incl. `..` above
  root, fragments, statics, broken links; template errors, include depth, standalone tags, loop vars,
  pagination, shadowed statics, reserved slugs).
- Random soup: **23,100 checks, 0 mismatches** (inline-mark soup, block-line soup, front-matter-line soup,
  template-tag soup, small random sites).

## Main speed techniques

- **Markdown inline**: the scan jumps from one special character (`\`, backtick, `*`, `[`) to the next with
  `indexOf` instead of a per-character loop; plain runs are escaped in one `replace` instead of one call per
  character; a mark that finds no closer stays in the plain run as literal text.
- **Markdown blocks**: single pass over the lines, rendering straight into the output array — no token
  objects; block shapes (fence/heading/hr/quote/list) recognised with char-code checks instead of regexes;
  fence bodies escaped as one joined string.
- **Front matter**: the closing `---` is found with `indexOf`, so the body is sliced out whole instead of
  split into lines and joined back; only the front-matter lines are split; key/value parsing is char-based.
- **Templates**: node trees rendered into one shared array (no per-include string concatenation); `for`
  uses a plain `for` loop instead of `forEach`; tag slugs computed once per name instead of three times.
- **Cards**: a post's card is built once and reused for its page view; `reading_minutes` counts words with
  an inlined whitespace test instead of a regex `match` (which allocates an array of every word).
- **Sources**: the `\r` normalisation replace runs only when the text actually holds a `\r`.
- `main.ts` keeps the baseline's driver shape, including `setFlagsFromString('--no-incremental-marking')`
  before importing `solve.js`.

## Deliberately not done

- Template compilation to generated functions (`eval`/`new Function`) was tried: it measured no better
  than the tree walk here and was removed, since dynamic code generation is not worth any risk from the
  site's security audit for ~0 gain.
- No lookup tables or precomputed data ship in the repo; everything is computed from the input.

## Repo size

44,516 bytes total (solve.js 40,589; main.ts 1,417; README.md 1,359; LICENSE 1,094; .makefun/owner.json 57)
— well under the 4 MiB cap.

## Weak spots / known limits

- The win is mostly in the build (~2.3×); the fixed JSON/startup overhead (~210 ms of ~580 ms) is shared
  with the baseline and cannot be cut much further.
- Timing on this shared machine is noisy (a parallel build pushed some runs up); the medians above are from
  alternating mine/baseline pairs. The graded heat times both entries on the same hardware, so the ratio is
  what matters, and it held at ~1.6× across every pair.
- Correctness depends on exact replication of the reference's quirks (documented in the code comments);
  the fuzzing above covers the generator's feature set plus adversarial corners, but the scored set is drawn
  from the same generator, so its distribution matches the fuzzed one.
