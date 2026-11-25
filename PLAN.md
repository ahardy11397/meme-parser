# PLAN — meme-parser: self-hosting step 5 (recursive-descent parser subset)

Build a structural parser for Memelang source, entirely in Memelang.
Self-hosting step 5: the front-door proof — token stream in (vendored
step-1 lexer), statement/expression tree out, rendered as text and JSON.
This is the biggest rung: read this PLAN fully before writing any code.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands run via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits — leave changes for review.
- 100% Memelang for program logic; no other languages.
- Every construct must parse with `--provider heuristic`.
- FOREGROUND test runs for individual targets; coordinator runs the full
  six-target matrix.
- lexer.meme and keywords.meme vendored from meme-tokenizer (attribution
  header only, do not refactor).
- NO new language features. Record gaps, route around, report.

## Honest scope

- The parser recognizes the CORE grammar (below). A statement or
  expression it recognizes becomes a typed tree node.
- Anything tokenizable but OUTSIDE the core grammar becomes an `opaque`
  node carrying its raw text span — NOT an error. The tool never fails on
  a tokenizable file; it tells you how much it understood.
- Lexical errors (bad char, unterminated string) are real errors:
  one clear line + exit 1.
- Output must be BYTE-IDENTICAL across all six targets for the same input.

## Grammar subset (core)

Statements:
- `record <Name>:` + indented `field: Type` lines -> record node
- `function <name>(<params>) -> <Type>:` and legacy-free signature form
  only (params: `name: Type` comma list) -> function node
- `test <name>:` -> test node (body like a function body)
- `define <name> as <expr>` / `define <name>: <Type> as <expr>` -> define
- `set <target> to <expr>` -> assign
- `print <expr>` / `print <expr>, <expr>, ...` -> print (arg list)
- `print error <expr...>` -> print-err
- `if <cond>:` / `else:` / `else if <cond>:` -> if node with branches
- `for each <name> in <expr>:` -> for-each
- `for <name> from <expr> to <expr>:` -> for-range
- `while <cond>:` -> while
- `return <expr>` / `return` -> return
- `exit with code <expr>` -> exit
- `use <module>` -> use
- `call <name>(<args>)` -> call
- `add <expr> to <target>` / `remove <expr> from <target>` -> add/remove
- anything else (whole line) -> opaque

Expressions (recursive descent, lowest to highest precedence):
- `or`, then `and`, then comparisons (`==`, `!=`, `<`, `>`, `<=`, `>=`),
  then `+`/`-`, then `*`/`/`/`%`, then unary `-`/`not`, then postfix
  (indexing `x[i]`, `x["k"]`, call `f(args)`), then primary.
- Primary: Int/Float literals, String literals with escapes, `true`/`false`
  if lexed as keywords (else treat as identifiers — mirror the lexer),
  `nothing`, parenthesized exprs, `[a, b, c]` list literals, identifiers.
- Recognized builtin shapes become typed nodes: `length of <expr>`,
  `char at <i> of <s>`, `slice <s> from <a> to <b>`, `index of <x> in <s>`,
  `read <Type>`, `read input`, `argument <n>`, `argument count`,
  `sha256 of <expr>`, `read json file <p> as List<T> or else <d>`,
  `read json record <R> from file <p> or else <d>`.
- Anything else at primary level: if it looks like `name(...)`, a call
  node; otherwise OPAQUE (consume the expression's token run — up to the
  end of line / `,` in an arg list / closing bracket — and wrap it).
  Expressions do NOT span multiple lines.

## AST model (this is where the beta's recursive records shine)

`record Node: kind: String, value: String, line: Int,
child: Optional<Node>, children: List<Node>`
- `child` for single-child wrappers (e.g. define's expr);
  `children` for statement bodies, arg lists, branches.
- Leaf nodes carry their text in `value`. Opaque nodes: kind `opaque`,
  value = raw text of the span.

## CLI (thin argv-router main + logic modules; unknown command -> usage +
exit 1; every error = one clear line + exit 1)

    parse <file.meme>                 # tree as indented text to stdout
    parse <file.meme> -o <out.json>   # tree as JSON (records) to file

Module layout: `ast.meme` (Node + emit-tree-text + constructor helpers),
`expr.meme` (expression recursive descent), `stmt.meme` (statement/block
parser: consumes the token list with an index, builds Nodes),
`main.meme` (router). Vendored lexer/keywords.

## Tests — tests/meme/*.meme, `meme test`

- Every core statement kind parses to the right node kind/value.
- Expression precedence: `1 + 2 * 3` nests `*` under `+`; `(1 + 2) * 3`
  nests `+` under `*`; comparisons bind looser than arithmetic;
  `a or b and c` nests `and` under `or`.
- Postfix chains: `m["k"][0]`, `f(1, g(2))`.
- if/else-if/else nesting; nested loops; while.
- Opaque fallback: an unrecognized construct line -> single opaque node,
  rest of file still parses.
- Lexical error propagates: unterminated string -> clean error, exit 1.
- Round-trip fidelity: parse the parser's OWN main.meme (vendored copy in
  tests/tmp/) — zero opaque nodes for its statements (the tool uses only
  core grammar; assert no opaque nodes in the tree).
- Real-fixture parity fixtures: small fixed .meme strings, asserted
  exactly (kind/value/children shape).
- Fixtures under tests/tmp/, unique per test, cleaned up.

## Acceptance ("done means")

1. `meme test` python target — 0 failed.
2. Same `--target rust cpp go c ts` — ALL SIX green (coordinator verifies).
3. `meme build main.meme -o parser_bin` — demo transcript:
   - parse this repo's own main.meme and stmt.meme (self-parse, no opaque
     nodes in either tree),
   - parse 4 real examples copied from
     /mnt/ssd/projects/meme/examples/advanced/ (538, 539, 541, 512) —
     show the tree heads and OPAQUE-NODE COUNTS per file.
4. README.md: CLI usage, grammar subset, opaque-node rule, AST node model,
   test instructions.
5. Final message: files, test count per target, demo transcript, opaque
   counts for the demoed files, deviations (if any) with reasons.
