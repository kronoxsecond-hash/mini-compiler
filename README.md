# MiniLang Compiler 🔧

A complete, didactic compiler for the **MiniLang** language, implemented in pure Python 3. The project covers all four classic compiler front-end phases: **Lexical Analysis**, **Syntax Analysis**, **Semantic Analysis**, and **Intermediate Code Generation**.

---

## 📋 Table of Contents

1. [Language Overview](#language-overview)
2. [Architecture](#architecture)
3. [Phase 1 – Lexer](#phase-1--lexical-analysis)
4. [Phase 2 – Parser](#phase-2--syntax-analysis--ast)
5. [Phase 3 – Semantic Analyser](#phase-3--semantic-analysis)
6. [Phase 4 – Code Generator](#phase-4--intermediate-code-generation)
7. [Error Handling](#error-handling)
8. [Usage](#usage)
9. [Example](#example)

---

## Language Overview

MiniLang is a small, statically-typed, imperative language that supports:

| Feature | Syntax |
|---|---|
| Integer variable declaration | `int x = 5;` |
| Assignment | `x = x + 1;` |
| Arithmetic | `+  -  *  /` |
| Comparison | `==  <  >` |
| Conditional | `if (cond) { … } else { … }` |
| Output | `print(expr);` |
| Line comments | `// this is ignored` |

---

## Architecture

```
Source Code
    │
    ▼
┌─────────────┐     List[Token]
│    LEXER    │ ──────────────────►
└─────────────┘
                                  ┌──────────────┐     AST
                                  │    PARSER    │ ──────────►
                                  └──────────────┘
                                                           ┌──────────────────┐    Symbol Table
                                                           │ SEMANTIC ANALYSER│ ──────────────►
                                                           └──────────────────┘
                                                                                           ┌───────────────┐    TAC
                                                                                           │ CODE GENERATOR│ ────►
                                                                                           └───────────────┘
```

---

## Phase 1 – Lexical Analysis

**File:** `compiler.py` → class `Lexer`

The lexer converts raw text into a flat sequence of **tokens**. Each token carries:
- `type` — one of the `TT` enum values (e.g. `TT.KW_INT`, `TT.IDENT`, `TT.PLUS`)
- `value` — the matched text (or integer for `INT_LIT`)
- `line` / `col` — source location for error reporting

**Token categories supported:**

| Category | Examples |
|---|---|
| Keywords | `int` `if` `else` `print` |
| Identifiers | `x`, `myVar`, `result_1` |
| Integer literals | `0`, `42`, `1000` |
| Operators | `+` `-` `*` `/` `=` `==` `<` `>` |
| Delimiters | `;` `{` `}` `(` `)` |

The lexer uses a **manual character-by-character scan** rather than regular expressions, giving full control over line/column tracking. It gracefully skips whitespace and `//` line comments.

---

## Phase 2 – Syntax Analysis / AST

**File:** `compiler.py` → class `Parser`, AST node dataclasses

The parser implements a **Recursive-Descent / LL(1)** strategy. Each grammar rule maps to a single Python method (`_stmt`, `_expr`, `_factor`, etc.). The full grammar is:

```
program     → stmt*  EOF
stmt        → var_decl | assign_stmt | if_stmt | print_stmt
var_decl    → 'int' IDENT ( '=' expr )? ';'
assign_stmt → IDENT '=' expr ';'
if_stmt     → 'if' '(' expr ')' block ( 'else' block )?
print_stmt  → 'print' '(' expr ')' ';'
block       → '{' stmt* '}'
expr        → comparison
comparison  → addition ( ( '==' | '<' | '>' ) addition )*
addition    → term ( ( '+' | '-' ) term )*
term        → factor ( ( '*' | '/' ) factor )*
factor      → NUMBER | IDENT | '(' expr ')'
```

The output is an **Abstract Syntax Tree (AST)** composed of lightweight Python dataclasses (`Program`, `VarDecl`, `Assign`, `BinOp`, `IfStmt`, `PrintStmt`, `Number`, `Identifier`).

---

## Phase 3 – Semantic Analysis

**File:** `compiler.py` → class `SemanticAnalyser`, `SymbolTable`

The semantic analyser does a single **recursive walk** of the AST and performs:

1. **Symbol table population** — every `int` declaration registers the variable name and type.
2. **Undeclared-variable detection** — any reference to an identifier not in the table raises a `SemanticError`.
3. **Duplicate declaration detection** — declaring the same variable twice is an error.
4. **Type propagation** — `BinOp` nodes return `"int"` for arithmetic operators and `"bool"` for comparison operators (`==`, `<`, `>`), enabling rudimentary type checking.

---

## Phase 4 – Intermediate Code Generation

**File:** `compiler.py` → class `CodeGenerator`

The code generator produces **Three-Address Code (TAC)**, a common compiler IR where every instruction has at most one operator and three operands (two sources, one result). Example instructions:

```
t0 = y * 2
t1 = x + t0
z = t1
PRINT z
IF_FALSE t2 GOTO L0
GOTO L1
LABEL L0
```

Temporary variables (`t0`, `t1`, …) are generated automatically. Labels (`L0`, `L1`, …) are created for control-flow instructions (`if`/`else` branches).

---

## Error Handling

All three analysis phases raise typed, descriptive exceptions:

| Exception | Phase | Example message |
|---|---|---|
| `LexerError` | Lexical | `[Lexer Error] Line 3, Col 7: Unknown character '@'` |
| `ParseError` | Syntax | `[Parse Error] Line 5, Col 1: Expected SEMI (got IDENT 'x')` |
| `SemanticError` | Semantic | `[Semantic Error] Line 8: Undeclared variable 'foo'` |

Every error message includes the **line number**, **column** (where applicable), the **error class**, and a **human-readable description**.

---

## Usage

```bash
# Run on a source file
python compiler.py example.ml

# The compiler will print:
#   === TOKENS ===
#   === AST ===
#   === SYMBOL TABLE ===
#   === THREE-ADDRESS CODE ===
```

You can also use the `compile_source(source: str) -> dict` function programmatically:

```python
from compiler import compile_source

result = compile_source("int x = 5; print(x);")
print(result["tac"])      # Three-address code
print(result["symbols"])  # Symbol table entries
print(result["errors"])   # Any errors encountered
```

---

## Example

**Input (`example.ml`):**
```c
int x = 10;
int y = 3;
int z = x + y * 2;
print(z);

if (x > y) {
    int result = x - y;
    print(result);
} else {
    print(0);
}
```

**Generated TAC:**
```
x = 10
y = 3
t0 = y * 2
t1 = x + t0
z = t1
PRINT z
t2 = x > y
IF_FALSE t2 GOTO L0
t3 = x - y
result = t3
PRINT result
GOTO L1
LABEL L0
PRINT 0
LABEL L1
```

---

## Requirements

- Python 3.8 or higher
- No external dependencies — pure standard library

---

## License

MIT
