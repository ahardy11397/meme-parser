# meme-parser

Self-hosting Steps 5 & 6: A structural recursive-descent parser and canonical tree-to-text emitter for Memelang source code, written entirely in 100% Memelang.

`meme-parser` ingests token streams from the vendored lexical analyzer and produces a typed AST (Abstract Syntax Tree) representing statements and expressions, rendering the resulting tree as indented text, structured JSON records, or back into canonical Memelang source code.

## CLI Usage

Build the standalone native binary using `meme build`:

```bash
uv run --project /mnt/ssd/projects/meme meme build main.meme -o parser_bin --provider heuristic
```

### Commands

#### 1. Print parsed AST as indented text:
```bash
./parser_bin parse <file.meme>
```

Example:
```bash
./parser_bin parse main.meme | head -n 25
```

#### 2. Write parsed AST as JSON:
```bash
./parser_bin parse <file.meme> -o <out.json>
```

Example:
```bash
./parser_bin parse main.meme -o ast.json
```

#### 3. Emit canonical Memelang text to stdout:
```bash
./parser_bin emit <file.meme>
```

Example:
```bash
./parser_bin emit main.meme | head -n 25
```

#### 4. Emit canonical Memelang text to file:
```bash
./parser_bin emit <file.meme> -o <out.meme>
```

Example:
```bash
./parser_bin emit stmt.meme -o canonical_stmt.meme
```

## Canonical Forms

The emitter renders AST nodes into canonical Memelang source text according to strict formatting rules:

- **record**: `record <Name>:` followed by 4-space-indented `    <field>: <Type>` lines.
- **function**: `function <name>(<p: T, ...>) -> <Ret>:` followed by 4-space-indented body.
- **test**: `test <name>:` followed by 4-space-indented body.
- **define**: `define <name> as <expr>` (or typed form `define <name>: <Type> as <expr>` when typed).
- **set**: `set <target> to <expr>`.
- **print / print-err**: `print <expr1>, <expr2>, ...` and `print error <expr1>, <expr2>, ...`.
- **if / else if / else**: `if <cond>:`, `else if <cond>:`, `else:` with 4-space-indented branch bodies.
- **for-each**: `for each <name> in <expr>:` with 4-space-indented body.
- **for-range**: `for <name> from <start> to <end>:` with 4-space-indented body.
- **while**: `while <cond>:` with 4-space-indented body.
- **return**: `return <expr>` or `return`.
- **exit**: `exit with code <expr>`.
- **use**: `use <module>`.
- **call**: `call <name>(<args>)`.
- **add**: `add <expr> to <target>`.
- **remove**: `remove <expr> from <target>`.
- **assert**: `assert <expr>`.
- **break / continue**: `break` and `continue`.
- **write**: `write <expr> as json to file <path>` and `write file <path> with <expr>`.
- **comments**: Emitted verbatim on their own lines where they appeared, with block indentation.
- **opaque nodes**: Emitted verbatim (their raw text span), untouched.
- **minimal parens**: Expressions are emitted in minimal-paren canonical form — parentheses are inserted only where operator precedence or associativity requires (e.g. `(a + b) * c` emits with parens, while `a + (b * c)` emits without redundant parens as `a + b * c`). Binary operators have single spaces around them; calls and indexing have no space before `(` or `[`.
- **top-level spacing**: Single blank lines between top-level statements; no blank lines introduced inside blocks.
- **empty program**: Empty input produces empty output (`""`).

## The Core Invariant and Idempotence

For every input $x$ that the parser accepts:

$$\text{parse}(\text{emit}(\text{parse}(x))) == \text{parse}(x)$$

as kind/value trees (source positions shift to canonical line offsets).

Furthermore, the emitter is byte-for-byte idempotent:

$$\text{emit}(\text{emit}(x)) == \text{emit}(x)$$

Tested across all test fixtures, the compiler's own source files (`main.meme`, `stmt.meme`, `ast.meme`, `expr.meme`, `keywords.meme`, `lexer.meme`, `emit.meme`), and advanced real examples (`512`, `538`, `539`, `541`).

## Grammar Subset (Core)

The parser recognizes the canonical Memelang core grammar and maps recognized structures into typed AST nodes:

### Statements
- `record <Name>:` + indented `field: Type` lines -> `record` node with `field` children
- `function <name>(<params>) -> <Type>:` -> `function` node with `signature` child and body `children`
- `test <name>:` -> `test` node with body `children`
- `define <name> as <expr>` / `define <name>: <Type> as <expr>` -> `define` node with `child` containing the expression
- `set <target> to <expr>` -> `assign` node with `child` containing the expression
- `print <expr...>` -> `print` node with argument expression `children`
- `print error <expr...>` -> `print-err` node with argument expression `children`
- `if <cond>:` / `else if <cond>:` / `else:` -> `if` node containing `branch` children
- `for each <name> in <expr>:` -> `for-each` node with collection expression `child` and body `children`
- `for <name> from <expr> to <expr>:` -> `for-range` node with `range` child and body `children`
- `while <cond>:` -> `while` node with condition `child` and body `children`
- `return <expr>` / `return` -> `return` node
- `exit with code <expr>` -> `exit` node with code expression `child`
- `use <module>` -> `use` node
- `call <name>(<args>)` -> `call` node with argument `children`
- `add <expr> to <target>` / `append <expr> to <target>` -> `add` node
- `remove <expr> from <target>` -> `remove` node
- `write <expr> as json to file <path>` / `write file <path> with <expr>` -> `write` node
- `assert <expr>` -> `assert` node
- `break` / `continue` -> `break` / `continue` node

### Expressions (Precedence Hierarchy)
Expressions are parsed via recursive descent / shunting-yard, strictly respecting standard precedence:
1. `or`: logical OR (`a or b`)
2. `and`: logical AND (`a and b`)
3. Comparisons: `==`, `!=`, `<`, `>`, `<=`, `>=`, `contains`, `is nothing`, `is not nothing`, `or else`
4. Additive: `+`, `-`
5. Multiplicative: `*`, `/`, `%`
6. Unary: `-`, `+`, `not`, `optional`, `value of`, `text of`, `is digit`, `is letter`, `is upper`, `is lower`
7. Postfix:
   - Indexing: `x[i]`, `m["k"][0]`
   - Field access: `x.field`, `user.profile.name`
   - Function call: `f(args)`
   - Record constructor: `RecordName(field1=val1, field2=val2)`
8. Primary:
   - Literals: `Int`, `Float`, `String`, `nothing`, `true`, `false`
   - Parenthesized expressions: `(expr)`
   - List literals: `[a, b, c]`
   - Empty collections: `empty List<T>`, `empty Set<T>`, `empty Map<K, V>`, `empty set of <T>`
   - Builtin forms:
     - `length of <expr>` / `length of string <expr>`
     - `char at <i> of <s>`
     - `slice <s> from <a> to <b>`
     - `index of <x> in <s>`
     - `argument <n>` / `argument count`
     - `sha256 of <expr>`
     - `read string from file <p>`
     - `read json record <R> from file <p> or else <d>`
     - `read json records <R> from file <p> or else <d>`
     - `read json file <p> as List<T> or else <d>`
     - `read <Type> with prompt <p>` / `read input`
     - `file exists <p>`

## Opaque-Node Rule

Any construct that is validly tokenized by the lexical analyzer but falls outside the core grammar is wrapped into an `opaque` AST node containing the raw text span of the line or expression.

The parser never fails on a tokenizable file: it parses the full file and reports how much it understood. Lexical errors (such as unterminated string literals or illegal characters) exit immediately with status code `1` and a single diagnostic message.

## AST Node Model

The entire AST is built upon a single recursive record:

```meme
record Node:
    kind: String
    value: String
    line: Int
    child: Optional<Node>
    children: List<Node>
```

- `kind`: The node type (e.g., `"program"`, `"record"`, `"function"`, `"define"`, `"binary_op"`, `"call"`, `"opaque"`, `"comment"`).
- `value`: Leaf text, identifier name, operator symbol, or raw text for opaque spans.
- `line`: 1-based source line number.
- `child`: Single-child wrapper (e.g., expression in a `define`, condition in an `if_branch`, target in a postfix chain).
- `children`: Multi-child collections (statement bodies in functions/loops, branch lists in `if`, arguments in calls).

## Testing

Run the test suite across individual backends via `meme test`:

```bash
uv run --project /mnt/ssd/projects/meme meme test --target python
uv run --project /mnt/ssd/projects/meme meme test --target rust
uv run --project /mnt/ssd/projects/meme meme test --target cpp
```

The test suite covers:
- Statement parsing and canonical emission for all core grammar kinds
- Expression precedence and minimal-paren formatting
- Opaque fallback and verbatim passthrough
- Comment preservation and verbatim emission
- Self-parsing and self-emitting roundtrip invariant and byte-for-byte idempotence
- File output (`-o`) and error handling
