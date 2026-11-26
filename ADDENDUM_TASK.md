# ADDENDUM_TASK: `emit` command — canonical re-rendering from the parse tree

Extend this repo (self-hosting step 6): a tree-to-text canonical emitter.
The parser (step 5) already builds the Node tree; this adds the missing
half: render the tree back to canonical Memelang text.

## Ground rules (unchanged from PLAN.md)

- Work ONLY in this repo. NO git commits.
- 100% Memelang; every construct parses with `--provider heuristic`.
- FOREGROUND runs for individual targets; coordinator runs the full matrix.
- NO new language features. Record gaps, route around, report.
- PERF CONSTRAINT (learned in step 5): list args deep-clone per call site
  on rust/cpp. The emitter walks a tree and builds text — design it
  clone-light: accumulate text into a List<String> of lines and join ONCE
  at the end; never build text with repeated `out = out + piece` in a loop
  (quadratic string concat via clones), never re-parse subtrees.

## New command

    emit <file.meme>                 # canonical text to stdout
    emit <file.meme> -o <out.meme>   # canonical text to file

Render rules per node kind (canonical forms — this DEFINES the tool's
canonical text, one form per construct):
- record: `record <Name>:` then 4-space-indented `    <field>: <Type>` lines.
- function: `function <name>(<p: T, ...>) -> <Ret>:` then body indented 4.
- test: `test <name>:` + body, same indentation.
- define: `define <name> as <expr>` (typed form if the original had a type:
  `define <name>: <Type> as <expr>` — preserve which, via a node marker if
  needed).
- set/print/if-else/for-each/for-from/while/return/exit/use/call/
  add/remove: canonical phrasing per the kinds you already parse.
- Expressions: minimal-paren canonical form — parenthesize only where
  precedence requires (the tree already encodes structure; emit from the
  tree, never from source text). Binary ops: single spaces around the
  operator. Calls/indexing: no space before `(` / `[`.
- Opaque nodes: emitted verbatim (their raw text), untouched.
- Blank lines between top-level statements; none introduced inside blocks.
- Empty program -> empty output.

## THE INVARIANT (test it hard — this is the acceptance core)

For every input this parser accepts:
    parse(emit(parse(x))) == parse(x)
as kind/value trees (positions may shift). Test on: every fixture in the
existing suites, the repo's own .meme files (self-emit), and the real
examples 538/539/541/512 copied to tests/tmp/. Idempotence too:
emit(emit(x)) == emit(x) byte-for-byte.

## Tests — add to existing suites or new files, `meme test`

- The invariant above on all listed fixtures (comment-filtered? NO —
  comments are tokens and opaque-ish: emit them verbatim on their own
  line where they appeared; the invariant holds with comments included
  as raw lines).
- Minimal parens: `(a + b) * c` emits WITH parens; `a + (b * c)` emits
  WITHOUT the redundant parens; both re-parse to the SAME tree.
- Canonical phrasing per statement kind (one test per kind).
- Opaque passthrough unchanged.
- Idempotence on all fixtures.
- -o writes the file; stdout form otherwise; missing file -> clean error.
- Existing 28 tests stay green, unchanged.

## Acceptance ("done means")

1. `meme test` python target — 0 failed (all existing + new).
2. rust/cpp/go/c/ts — coordinator verifies; you run python + rust + cpp
   individually, and TIME the self-parse test (must stay < 5s rust).
3. `meme build main.meme -o parser_bin` — demo transcript:
   emit the repo's own stmt.meme (self-emit), re-parse the emitted text,
   show the invariant holds (0 diff via a second parse + compare), show
   idempotence (second emit byte-identical), and emit 539 with its opaque
   line passed through verbatim.
4. README updated: emit command, canonical forms, the invariant.
5. Final message: files, test counts, timings, demo transcript,
   deviations (if any) with reasons.
