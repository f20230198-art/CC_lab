# Mini Compiler — Code Guide

## Project Structure

```
CC_Lab/
├── lexer.h          Phase 1  — Lexical Analysis
├── parser.h         Phase 2  — Recursive-Descent Parser + AST
├── ll_slr.h         Phase 3  — LL(1) + SLR Table-Driven Parsers
├── symbol_table.h   Phase 4  — Symbol Table & Scope Handling   ← Compre Q2
├── semantic.h       Phase 5  — Semantic Analysis               ← Compre Q3
├── main.cpp         Runs the full pipeline on eval_program.txt
├── eval_program.txt Mandatory evaluation source program
└── old/             Previous monolithic cclab1–5 files
```

## Compile & Run

```bash
g++ -std=c++17 -o compiler main.cpp
./compiler
```

Run on a different file:
```bash
./compiler mytest.txt
```

## Dependency Chain

```
lexer.h  →  parser.h  →  symbol_table.h  →  semantic.h
                                                ↑
                                           main.cpp also includes ll_slr.h
```

## Adding a New Phase (next lab)

1. Create `tac.h` starting with `#include "semantic.h"`
2. Write your new class/functions in it
3. Add `#include "tac.h"` to `main.cpp` and a new demo section
4. Compile the same way — nothing else changes

---

## Assignment Coverage

### Midsem Q1 — Lexical Analysis (`lexer.h`)
Tokenizes the source program into a stream of tokens.
Each token has a type (INT, ID, NUMBER etc.), its text, and line number.
Detects and reports any unrecognized characters as lexical errors.

### Midsem Q2 — Syntax Analysis (`parser.h`)
Recursive-descent parser — one function per grammar rule.
Builds a parse tree (AST) from the token stream.
Grammar handles: declarations, assignments, if-else, while, print, expressions with full precedence.
Error recovery: on a bad token it skips to the next `;` or `}` and keeps going.

### Midsem Q3 — Syntax Error Detection (`main.cpp` Phase 2)
The parser reports syntax errors with line numbers.
Any syntactically broken program passed to it will produce SYNTAX ERROR messages and a FAILED result.

### Compre Q3 — LL(1) + SLR Parsers (`ll_slr.h`)
Two table-driven parsers built on top of the lexer.
Both use a simpler expression-only grammar (not the full language grammar — LL(1) cannot handle the full grammar without heavy transformation).

**LL(1):**
- Left-recursion eliminated using primed non-terminals (E', F', S', T')
- FIRST and FOLLOW sets computed automatically
- Parsing table built from FIRST/FOLLOW — no conflicts means grammar is LL(1)
- Parses top-down using a stack: expand non-terminals via table, match terminals

**SLR:**
- Uses the original left-recursive grammar (bottom-up parsers handle left-recursion natively)
- LR(0) item sets and automaton built automatically
- SLR table: ACTION (shift/reduce/accept) + GOTO
- Parses bottom-up: shift tokens onto stack, reduce when a handle is found

### Compre Q2 — Symbol Table (`symbol_table.h`)

**Data structure:** stack of scope frames. Each frame is a `map<name, Symbol>`.
Each `Symbol` stores: name, type (int/float), scope level (0=global), offset (bytes from frame start).

**Scope handling:**
- `enterScope()` called on every `{` — pushes a new empty frame, resets offset to 0
- `exitScope()` called on every `}` — pops the frame, all variables in it are gone
- Scope level 0 = global, level 1 = first nested block, level 2 = deeper, etc.

**Memory offset:**
- Both int and float are 4 bytes
- First variable in a scope gets offset 0, next gets 4, next gets 8, etc.
- Each scope frame has its own independent offset counter

**Insert:** adds to current (innermost) scope frame. Returns false if name already exists in that frame.

**Lookup:** searches from innermost frame outward. This is how inner blocks can see outer variables.

**Step-by-step log:** every insert, lookup, and scope transition is recorded and printed — this is the "display symbol table updates step-by-step" the assignment asks for.

### Compre Q3 — Semantic Analysis (`semantic.h`)

Walks the AST node by node. Uses the symbol table for all variable information.

**How it's integrated with parser and symbol table:**
- Parser builds the AST
- SemanticAnalyzer takes that AST + a SymbolTable reference
- `analyze(root)` walks every node and calls the symbol table for every variable it sees

**What it checks:**

| Error | How it's detected |
|---|---|
| Undeclared variable | `lookup()` returns nullptr — variable never inserted |
| Duplicate declaration | `insert()` returns false — name already in current scope frame |
| Type mismatch (assign) | LHS symbol type is "int", RHS `typeOf()` returns "float" |
| Invalid bool condition | condition `typeOf()` returns "int" instead of "bool" |

**`typeOf()` function:**
- `NUMBER` → "int", `FLOAT_LIT` → "float"
- `ID` → looks up symbol table, returns stored type
- `add_expr` / `mul_expr` → both operands same type returns that type; int+float widens to float
- `rel_expr` → always returns "bool" (it's a comparison)
- `or_expr` / `and_expr` / `not_expr` → always "bool"
- `paren_expr` → recurses into the expression inside

**Scope during semantic analysis:**
- Global declarations go into scope 0
- Every `block` node triggers `enterScope()` before checking its contents and `exitScope()` after
- This means inner variables shadow outer ones correctly, and are gone after their block

**How to demo semantic errors during evaluation:**
The compiler runs on whatever is in `eval_program.txt` (or any file you pass).
To show error cases, just edit the file and rerun — the compiler detects and reports everything automatically.

Examples of changes that trigger errors:
- Remove `int b;` → every use of `b` becomes an undeclared variable error
- Add `int a;` twice → duplicate declaration error in scope 0
- Change `sum = 0;` to `sum = 3.14;` → type mismatch (float assigned to int)
- Change `while (a < b ...)` to `while (a ...)` → invalid boolean condition (plain int)

No code changes needed — just edit the `.txt` file and run `./compiler`.
