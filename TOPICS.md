# Compiler Construction - Viva Topics Guide

> **Speed-read version:** Skim the **bold** lines and the tables/diagrams. Come back to the details only if something is unclear.

---

## Table of Contents
1. [The Big Picture: What Does a Compiler Do?](#1-the-big-picture)
2. [Lexical Analysis (Tokenizer / Scanner / Lexer)](#2-lexical-analysis)
3. [Tokens, Lexemes & Patterns](#3-tokens-lexemes--patterns)
4. [How the Lexer Decides: Keywords vs Identifiers](#4-keywords-vs-identifiers)
5. [Lexical Errors](#5-lexical-errors)
6. [Syntax Analysis (Parser)](#6-syntax-analysis)
7. [Context-Free Grammar (CFG)](#7-context-free-grammar)
8. [Parse Trees](#8-parse-trees)
9. [Operator Precedence & Associativity](#9-operator-precedence--associativity)
10. [Recursive Descent Parsing](#10-recursive-descent-parsing)
11. [FIRST and FOLLOW Sets](#11-first-and-follow-sets)
12. [Syntax Errors vs Lexical Errors](#12-syntax-errors-vs-lexical-errors)
13. [Error Recovery (Panic Mode)](#13-error-recovery-panic-mode)
14. [Previous Viva Questions — Answered](#14-previous-viva-questions--answered)
15. [LL(1) Parsing](#15-ll1-parsing)
16. [Shift-Reduce Parsing](#16-shift-reduce-parsing)
17. [LL(1) vs Shift-Reduce — When to Use Which?](#17-ll1-vs-shift-reduce--when-to-use-which)
18. [Previous Viva Questions — Lab 4 Specific](#18-previous-viva-questions--lab-4-specific)

---

## 1. The Big Picture

```
Source Code ──▶ [ LEXER ] ──▶ Token Stream ──▶ [ PARSER ] ──▶ Parse Tree
               (Phase 1)                       (Phase 2)
```

| Phase | Input | Output | Catches |
|-------|-------|--------|---------|
| **Lexer** | Raw characters (`int x = 5;`) | Tokens (`<INT,"int"> <ID,"x"> <ASSIGN,"="> <NUMBER,"5"> <SEMI,";">`) | Unrecognized characters (`@`, `$`, lone `\|`) |
| **Parser** | Token stream | Parse tree / AST | Structural mistakes (missing `;`, wrong order of tokens) |

**Think of it like language:**
- The **lexer** checks spelling (are these valid words?)
- The **parser** checks grammar (are the words in the right order?)

---

## 2. Lexical Analysis

The **lexer** (also called **scanner** or **tokenizer**) reads source code **character by character** and groups characters into meaningful chunks called **tokens**.

### How it works step-by-step:

```
Input: "int x = 5;"

Step 1: Read 'i' → letter, so keep reading → 'n' → 't' → space (stop!)
        Collected: "int" → Is it a keyword? YES → Token: <INT, "int">

Step 2: Skip whitespace

Step 3: Read 'x' → letter, keep reading → '=' is not alphanumeric (stop!)
        Collected: "x" → Is it a keyword? NO → Token: <ID, "x">

Step 4: Skip whitespace

Step 5: Read '=' → Is next char '='? NO → Token: <ASSIGN, "=">

Step 6: Skip whitespace

Step 7: Read '5' → digit, keep reading → ';' is not digit (stop!)
        Collected: "5" → Token: <NUMBER, "5">

Step 8: Read ';' → Token: <SEMI, ";">

Step 9: Read '\0' → Token: <EOF>
```

### Key rules the lexer follows:

1. **Skip whitespace** (spaces, tabs, newlines are ignored — they just separate tokens)
2. **Starts with letter or `_`** → read an identifier/keyword
3. **Starts with digit** → read a number (check for `.` to decide int vs float)
4. **Otherwise** → try to match an operator or punctuation

---

## 3. Tokens, Lexemes & Patterns

These three terms come up A LOT in vivas. Know the difference:

| Term | What it means | Example |
|------|--------------|---------|
| **Token** | The *category* + *value* pair | `<ID, "abc">`, `<NUMBER, "42">`, `<PLUS, "+">` |
| **Lexeme** | The *actual substring* from source code | `"abc"`, `"42"`, `"+"` |
| **Pattern** | The *rule* that defines what a token looks like | ID = `[a-zA-Z_][a-zA-Z0-9_]*` |

### Example breakdown:

```c
float avg = 5.0;
```

| Lexeme | Token Type | Token |
|--------|-----------|-------|
| `float` | FLOAT (keyword) | `<FLOAT, "float">` |
| `avg` | ID (identifier) | `<ID, "avg">` |
| `=` | ASSIGN | `<ASSIGN, "=">` |
| `5.0` | FLOAT_LIT | `<FLOAT_LIT, "5.0">` |
| `;` | SEMI | `<SEMI, ";">` |

**Important:** In the token `<ID, "abc">`:
- Token type = `ID`
- Lexeme = `abc`
- The whole thing `<ID, "abc">` is the token

---

## 4. Keywords vs Identifiers

**This is a very common viva question!**

### The Problem:
Both keywords (`int`, `if`, `while`) and identifiers (`x`, `myVar`, `count`) start with a letter. How does the lexer tell them apart?

### The Solution — "Check keyword table AFTER reading the word":

```
Read full word → Check: Is it in keyword list?
                    YES → return keyword token (e.g., INT, IF, WHILE)
                    NO  → return ID token
```

### What if keyword and identifier are in the same declaration?

```c
int int;     ← "int" appears twice
```

- First `int`: The lexer reads "int", checks keyword table → **match** → `<INT, "int">`
- Second `int`: The lexer reads "int", checks keyword table → **match** → `<INT, "int">`
- The **lexer** will happily tokenize both as keyword `INT`
- The **parser** will then FAIL because it expects `<ID>` after a type, not another keyword
- **Result: This is a SYNTAX ERROR, not a lexical error**

### The `intx=5` question (no spaces):

```
Input: "intx=5"

Lexer reads: 'i','n','t','x' → keeps going because 'x' is alphanumeric!
Collected: "intx" → Is it a keyword? NO → Token: <ID, "intx">

Then: '=' → Token: <ASSIGN, "=">
Then: '5' → Token: <NUMBER, "5">
```

**Result:** `<ID, "intx"> <ASSIGN, "="> <NUMBER, "5">`

The lexer does NOT see `int` as a keyword here! Because it uses **maximal munch** — it reads the longest possible token. Since `intx` is a valid identifier, it takes the whole thing.

---

## 5. Lexical Errors

A **lexical error** happens when the lexer encounters a character that **doesn't match ANY token pattern**.

### Examples of lexical errors:
| Character | Why it's an error |
|-----------|------------------|
| `@` | Not in any token definition |
| `$` | Not recognized |
| Single `\|` | Our language only has `\|\|`, not `\|` alone |
| Single `&` | Our language only has `&&`, not `&` alone |

### What is NOT a lexical error:
| Input | Why it's NOT a lex error |
|-------|------------------------|
| `%` by itself | `%` IS a valid token (MODULO) — the lexer recognizes it fine |
| `int int;` | Both are valid tokens — the error is structural (parser's job) |
| Missing `;` | The lexer doesn't care about structure |

---

## 6. Syntax Analysis

The **parser** takes the token stream and checks if it follows the **grammar rules** (CFG). If yes, it builds a **parse tree**.

### What the parser does:
1. Reads tokens one by one (like the lexer reads characters)
2. Tries to match them against grammar rules
3. Builds a tree structure
4. Reports errors when tokens don't match expected patterns

---

## 7. Context-Free Grammar (CFG)

A CFG is a set of rules that describe valid programs. It's written as:

```
non_terminal → what it can be replaced with
```

### Your lab's grammar:

```
program      → stmt_list EOF
stmt_list    → stmt stmt_list | ε (empty)
stmt         → decl_stmt | assign_stmt | if_stmt | while_stmt | print_stmt | block
decl_stmt    → type ID ;
assign_stmt  → ID = expr ;
if_stmt      → if ( expr ) block
             | if ( expr ) block else block
while_stmt   → while ( expr ) block
print_stmt   → print ( expr ) ;
block        → { stmt_list }
```

### Expression grammar (with precedence built in):

```
expr         → or_expr
or_expr      → and_expr ( || and_expr )*
and_expr     → not_expr ( && not_expr )*
not_expr     → ! not_expr | rel_expr
rel_expr     → add_expr ( rel_op add_expr )*
add_expr     → mul_expr ( ( + | - ) mul_expr )*
mul_expr     → unary ( ( * | / | % ) unary )*
unary        → - unary | primary
primary      → NUMBER | FLOAT_LIT | ID | ( expr )
```

### Key vocabulary:
| Term | Meaning | Example |
|------|---------|---------|
| **Terminal** | Actual tokens (leaves of tree) | `ID`, `NUMBER`, `+`, `;` |
| **Non-terminal** | Grammar rules (internal nodes) | `expr`, `stmt`, `program` |
| **Production** | A rule | `decl_stmt → type ID ;` |
| **ε (epsilon)** | Empty string / nothing | `stmt_list → ε` means the list can be empty |

---

## 8. Parse Trees

A **parse tree** shows HOW the parser derived the input using grammar rules. Every internal node is a non-terminal, every leaf is a terminal.

### Example: `a = 2 + 3 * 4 ;`

```
        assign_stmt
       /    |    \    \
     ID   ASSIGN  expr  SEMI
    [a]    [=]     |     [;]
              add_expr
             /    |    \
        mul_expr  +   mul_expr
           |         /    |    \
         NUMBER   NUMBER  *   NUMBER
          [2]      [3]        [4]
```

**Why this shape matters:**
- `*` is DEEPER in the tree than `+`
- Deeper = evaluated FIRST
- So `3 * 4 = 12` is computed first, then `2 + 12 = 14`
- **The tree structure encodes operator precedence!**

### `id + (id * id)` vs `id + id * id`

Both give the **same** parse tree! Because:
- Without parentheses: `*` has higher precedence, so it binds tighter → `id + (id * id)` is the implicit grouping
- With parentheses: explicitly says `id * id` first → same result

```
        add_expr                    add_expr
       /    |    \                 /    |    \
      id    +   mul_expr         id    +   paren_expr
               /    |    \                 /    |    \
              id    *    id              (   mul_expr  )
                                           / | \
                                         id  *  id
```

**Both are semantically equivalent** — they compute the same thing. The first one (without parentheses) is considered "more correct" or "cleaner" because the grammar already handles the precedence. The parentheses are redundant.

---

## 9. Operator Precedence & Associativity

### Precedence (highest to lowest in our grammar):

| Priority | Operators | Grammar Rule | Example |
|----------|----------|-------------|---------|
| 1 (highest) | `- (unary)` | `unary` | `-x` |
| 2 | `*  /  %` | `mul_expr` | `a * b` |
| 3 | `+  -` | `add_expr` | `a + b` |
| 4 | `== != < > <= >=` | `rel_expr` | `a < b` |
| 5 | `!` | `not_expr` | `!a` |
| 6 | `&&` | `and_expr` | `a && b` |
| 7 (lowest) | `\|\|` | `or_expr` | `a \|\| b` |

**How precedence is built into the grammar:**
- Lower precedence operators are **higher** in the grammar (parsed first, closer to root)
- Higher precedence operators are **lower** in the grammar (parsed later, deeper in tree)
- This means `*` is evaluated before `+` because `mul_expr` is nested inside `add_expr`

### Associativity:
- **Left-to-right** (our grammar uses `while` loops for repeated operators):
  - `a + b + c` = `(a + b) + c` — left associative
  - `a * b * c` = `(a * b) * c`
- The `while` loop in `parseAddExpr()` keeps building left-leaning trees

### The `a<b<c` question:

```
Input: a < b < c
```

**Will the parser parse it?** YES, it will — but it might give unexpected results.

In our grammar, `rel_expr → add_expr ( rel_op add_expr )*`, the `*` means we can have **multiple** relational operators. So the parser will parse `a < b < c` as:

```
       rel_expr
      /    |    \
  rel_expr  <    c
  /   |  \
 a    <   b
```

This means `(a < b) < c`. The result of `a < b` is 0 or 1, then compared with `c`. This is **syntactically valid** but **semantically questionable** (probably not what you meant). In languages like Python, `a < b < c` is chained comparison, but in C-like languages it's left-associative and gives weird results.

---

## 10. Recursive Descent Parsing

This is the **parsing technique** used in your lab. It's a **top-down** parser where:
- Each grammar rule becomes a **function**
- The parser starts from the top rule (`program`) and works down
- It looks at the **current token** to decide which rule to apply

### How it decides which rule to use:

```
parseStmt() {
    if current token is INT or FLOAT → parseDeclStmt()
    if current token is ID           → parseAssignStmt()
    if current token is IF           → parseIfStmt()
    if current token is WHILE        → parseWhileStmt()
    if current token is PRINT        → parsePrintStmt()
    if current token is {            → parseBlock()
    otherwise                        → ERROR!
}
```

**This is why FIRST sets matter** — the parser needs to know which tokens can START each rule.

### What happens if input starts with `+`?

```
Input: + 5 ;
```

The parser calls `parseStmt()`. It checks:
- Is current token `INT`/`FLOAT`? No
- Is current token `ID`? No
- Is current token `IF`? No
- Is current token `WHILE`? No
- Is current token `PRINT`? No
- Is current token `{`? No
- **None match!** → **SYNTAX ERROR**: "Unexpected token '+'"

Then **panic mode recovery** kicks in (see Section 13).

---

## 11. FIRST and FOLLOW Sets

### FIRST Set
**FIRST(A)** = the set of tokens that can appear as the **first token** of anything derived from A.

Think: "If I'm about to parse non-terminal A, what tokens could I see first?"

#### Computing FIRST for your grammar:

| Non-terminal | FIRST set |
|-------------|-----------|
| `program` | `{int, float, ID, if, while, print, {, EOF}` |
| `stmt` | `{int, float, ID, if, while, print, {}` |
| `decl_stmt` | `{int, float}` |
| `assign_stmt` | `{ID}` |
| `if_stmt` | `{if}` |
| `while_stmt` | `{while}` |
| `print_stmt` | `{print}` |
| `block` | `{{}` |
| `expr` | `{!, -, NUMBER, FLOAT_LIT, ID, (}` |
| `primary` | `{NUMBER, FLOAT_LIT, ID, (}` |

#### How to compute FIRST:

1. If the rule starts with a **terminal**: that terminal is in FIRST
   - `decl_stmt → type ID ;` → FIRST = {int, float} (type can be int or float)
2. If the rule starts with a **non-terminal**: include FIRST of that non-terminal
   - `expr → or_expr` → FIRST(expr) = FIRST(or_expr) = ... = {!, -, NUMBER, FLOAT_LIT, ID, (}
3. If a rule can produce **ε**: include ε in FIRST, and also check next symbol

#### The `{id, {}}` question:

The notation `{id, {}}` means: **FIRST set = {id, {} }** — i.e., the tokens `id` and `{` (left brace).

This would be the FIRST set of a non-terminal like `stmt` (since a statement can start with an identifier for assignment, or `{` for a block).

Wait — more precisely: **FIRST(stmt)** includes `ID` and `{` among others. The question is probably asking about a specific non-terminal whose FIRST set contains exactly `{id, {}}`.

### FOLLOW Set
**FOLLOW(A)** = the set of tokens that can appear **immediately after** A in any derivation.

Think: "After I finish parsing A, what token could come next?"

| Non-terminal | FOLLOW set |
|-------------|------------|
| `program` | `{$}` (end of input) |
| `stmt_list` | `{}, EOF}` |
| `stmt` | `{int, float, ID, if, while, print, {, }, EOF}` |
| `expr` | `{), ;}` |

---

## 12. Syntax Errors vs Lexical Errors

**This distinction is a VERY popular viva question!**

| | Lexical Error | Syntax Error |
|--|--------------|-------------|
| **Who catches it?** | Lexer | Parser |
| **What's wrong?** | Invalid character/token | Valid tokens in wrong order |
| **Example** | `@`, `$`, lone `&` | `int int;`, `= = x;`, missing `;` |
| **Phase** | Phase 1 | Phase 2 |

### Why is `%` NOT a lexical error but a syntax error?

**`%` IS a valid token!** The lexer recognizes `%` as `<MODULO, "%">`. So the lexer says "all good!"

But if `%` appears in the wrong place:
```c
int % = 5;
```

Tokenization succeeds: `<INT, "int"> <MODULO, "%"> <ASSIGN, "="> <NUMBER, "5"> <SEMI, ";">`

The **parser** then tries: `parseDeclStmt()` expects `type ID ;` but gets `type MODULO ...` → **SYNTAX ERROR**

**Key insight:** A character being "weird" doesn't make it a lex error. If the lexer has a rule for it, it's a valid token. The error only shows up when the parser tries to fit that token into the grammar.

---

## 13. Error Recovery (Panic Mode)

When the parser hits an error, it needs to **recover** so it can keep parsing and find more errors (instead of stopping at the first one).

### Panic Mode Recovery:

```
How it works:
1. Parser detects error (unexpected token)
2. Parser starts SKIPPING tokens
3. It keeps skipping until it finds a "synchronizing token"
4. Synchronizing tokens are usually: ; (semicolon), } (closing brace), EOF
5. After finding the sync token, resume normal parsing
```

### In your code, the `sync()` / `recover()` function:

```cpp
void sync() {
    // Skip tokens until we find ; or } or EOF
    while (!check(SEMI) && !check(RBRACE) && !check(END_OF_FILE))
        eat();
    // If we found ;, consume it too (so next statement starts fresh)
    if (check(SEMI)) eat();
}
```

### Example of panic mode:

```c
int = 5;        ← ERROR: expected identifier after "int"
int b;          ← parser should recover and parse this correctly
```

1. Parser sees `<INT>`, calls `parseDeclStmt()`
2. Expects `<ID>` but gets `<ASSIGN>` → ERROR reported
3. **Panic mode**: skip tokens until `;` → skips `=`, `5`, consumes `;`
4. Now parser is at `int b;` → parses normally!

### Why it's called "Panic" mode:
Because the parser "panics" and just throws away tokens until it finds safe ground. It's the **simplest** error recovery strategy.

### Other recovery strategies (for knowledge):
| Strategy | How it works |
|----------|-------------|
| **Panic mode** | Skip to synchronizing token (`;`, `}`) — **your lab uses this** |
| **Phrase-level** | Replace/insert/delete a token to fix locally |
| **Error productions** | Add grammar rules that match common mistakes |
| **Global correction** | Find minimum edits to make program valid (theoretical) |

---

## 14. Previous Viva Questions — Answered

Here are the exact questions from the previous batch, fully answered:

---

### Q1: `id + (id * id)` vs `id + id * id` — Why is first one semantically correct?

**Both produce the same result** because `*` has higher precedence than `+`.

Without parentheses: grammar forces `*` to bind first → `id + (id * id)`
With parentheses: explicitly groups `*` first → same thing

The professor's point: `id + (id * id)` makes the precedence **explicit** and matches the parse tree structure directly. `id + id * id` relies on **implicit** precedence rules. Both are correct, but the first is "semantically clearer."

---

### Q2: If both keyword and identifier are defined in same declaration, what happens?

```c
int int;
```

- **Lexer**: Produces `<INT, "int"> <INT, "int"> <SEMI, ";">`  — no error!
- **Parser**: Expects `type ID ;` but gets `type KEYWORD ;` → **SYNTAX ERROR**
- The keyword `int` can never be used as a variable name because the lexer always classifies it as a keyword first.

---

### Q3: `{id, {}}` — FIRST set of some non-terminal

This represents a FIRST set. The set contains: `id` and `{`.

Non-terminals with these in their FIRST set:
- **`stmt`** has FIRST = {int, float, **ID**, if, while, print, **{**}
- If the question is specifically about a non-terminal with FIRST = {id, {}}, it could be a simplified grammar's rule like `stmt → id = expr ; | { stmt_list }`

---

### Q4: If the input starts with `+`, then what will happen?

The parser calls `parseStmt()`:
- `+` doesn't match any statement start token (not int/float/ID/if/while/print/{)
- **SYNTAX ERROR**: "Unexpected token '+'"
- Parser enters **panic mode** → skips tokens until `;` or `}` or `EOF`

Note: `+` IS a valid token (the lexer is fine with it). The error is at the **parser** level.

---

### Q5: Why is `%` not a lex error but a syntax error?

**`%` is a valid token** → `<MODULO, "%">`. The lexer has a rule for it.

A **lexical error** means "I don't know what character this is." The lexer DOES know `%` — it's the modulo operator.

A **syntax error** happens when `%` appears where it shouldn't:
```c
int % = 5;    // Lexer: fine. Parser: "Expected identifier, got %"
```

---

### Q6: Is `(ID, "abc")` a token? What is the lexeme?

Yes, `(ID, "abc")` **is a token**. Specifically:
- **Token type** = `ID` (identifier)
- **Lexeme** = `abc` (the actual text from source code)
- **Token** = the whole pair `<ID, "abc">`

---

### Q7: `a<b<c` — Will the parser parse it?

**YES**, the parser will parse it successfully.

In the grammar: `rel_expr → add_expr ( rel_op add_expr )*`

The `*` (zero or more) means multiple relational operators are allowed. So:
- Parse `a`, then see `<`, parse `b` → now we have `a < b`
- See another `<`, parse `c` → now we have `(a < b) < c`

It parses as `(a < b) < c` (left-associative). **Syntactically valid, but semantically suspicious** — the result of `a < b` is 0 or 1, then compared with `c`.

---

### Q8: `intx=5` without space — What will be the tokens?

```
Lexer uses "maximal munch" (longest match):

Read: i, n, t, x → all alphanumeric, keep going → '=' stops it
Word: "intx" → Not a keyword → <ID, "intx">

Read: '=' → next is '5', not '=' → <ASSIGN, "=">

Read: '5' → <NUMBER, "5">
```

**Result: `<ID, "intx"> <ASSIGN, "="> <NUMBER, "5">`**

`int` is NOT recognized as a keyword because the lexer reads the **longest possible** identifier first.

---

### Q9: Panic Recovery Mode

See [Section 13](#13-error-recovery-panic-mode) above for the full explanation.

**Short answer:** When the parser hits an unexpected token, it skips tokens until it finds a "safe" synchronizing token (`;` or `}` or `EOF`), then resumes parsing. This allows reporting multiple errors in one pass instead of stopping at the first error.

---

## 15. LL(1) Parsing

### What is LL(1)?
- **L** = scan input **L**eft to right
- **L** = produce a **L**eftmost derivation
- **1** = use **1** lookahead token

It's a **top-down, table-driven** parser. You build a table, then parsing is mechanical — no recursion needed.

### Steps to build an LL(1) parser:

**Step 1: Eliminate left-recursion**

The original grammar has rules like `E → E + T | T`. This is **left-recursive** — the parser would loop forever trying to expand E.

Transform using the pattern:
```
A → Aα | β      becomes      A  → β A'
                              A' → α A' | ε
```

Example:
```
E → E + T | T     becomes     E  → T E'
                              E' → + T E' | ε
```

**Step 2: Compute FIRST sets**

FIRST(X) = set of terminals that can begin a string derived from X.

Rules:
1. If X is a terminal: FIRST(X) = {X}
2. If X → Y₁ Y₂ ... Yₖ: add FIRST(Y₁). If Y₁ can derive ε, also add FIRST(Y₂), etc.
3. If X → ε: add ε to FIRST(X)

**Step 3: Compute FOLLOW sets**

FOLLOW(A) = set of terminals that can appear immediately after A.

Rules:
1. Put $ in FOLLOW(start symbol)
2. If A → αBβ: add FIRST(β) - {ε} to FOLLOW(B)
3. If A → αB, or A → αBβ where ε ∈ FIRST(β): add FOLLOW(A) to FOLLOW(B)

**Step 4: Build the parsing table**

For each production A → α:
- For each terminal a in FIRST(α): set table[A][a] = A → α
- If ε ∈ FIRST(α): for each terminal b in FOLLOW(A): set table[A][b] = A → ε

**If any cell gets two entries, the grammar is NOT LL(1).**

**Step 5: Parse using the table**

```
Stack: $ S          Input: tokens... $

Repeat:
  top = stack top
  cur = current input
  
  If top == cur (both terminals):  pop stack, advance input  (MATCH)
  If top is non-terminal:          look up table[top][cur], pop, push RHS reversed
  If top == $ and cur == $:        ACCEPT
  Otherwise:                       ERROR
```

### Full worked example: `x = 3 + 4 ;`

```
Step  Stack                Input                   Action
1     $ S                  id = num + num ; $      S → A
2     $ A                  id = num + num ; $      A → id = E ;
3     $ ; E = id           id = num + num ; $      Match 'id'
4     $ ; E =              = num + num ; $         Match '='
5     $ ; E                num + num ; $           E → T E'
6     $ ; E' T             num + num ; $           T → F T'
7     $ ; E' T' F          num + num ; $           F → num
8     $ ; E' T' num        num + num ; $           Match 'num'
9     $ ; E' T'            + num ; $               T' → ε  (because + is not *, /, %)
10    $ ; E'               + num ; $               E' → + T E'
11    $ ; E' T +           + num ; $               Match '+'
12    $ ; E' T             num ; $                 T → F T'
13    $ ; E' T' F          num ; $                 F → num
14    $ ; E' T' num        num ; $                 Match 'num'
15    $ ; E' T'            ; $                     T' → ε
16    $ ; E'               ; $                     E' → ε
17    $ ;                  ; $                     Match ';'
18    $                    $                       ACCEPT
```

### Why LL(1) needs left-recursion removal:

If we had `E → E + T`, and the parser sees E on top with `num` on input, it would expand E → E + T, pushing `E` back on top — **infinite loop!** The primed form `E → T E'` always starts with T (not E), so it makes progress.

---

## 16. Shift-Reduce Parsing

### What is Shift-Reduce?
A **bottom-up** parser that builds the parse tree from leaves to root. It works by:
- **Shift**: push the next input token onto the stack
- **Reduce**: replace a sequence of symbols on the stack top (a **handle**) with the corresponding non-terminal

### Key terms:
| Term | Meaning |
|------|---------|
| **Handle** | A substring on the stack that matches the RHS of a production |
| **Shift** | Move next input token to the stack |
| **Reduce** | Replace handle with LHS non-terminal |
| **Viable prefix** | A prefix of a right-sentential form that can appear on the stack |

### The algorithm:

```
Stack: $                  Input: tokens... $

Repeat:
  1. Check if top of stack matches any production RHS (a "handle")
  2. If yes (and precedence allows): REDUCE — pop the handle, push LHS
  3. If no: SHIFT — push next input token onto stack
  4. If stack = $ S and input = $: ACCEPT
```

### Resolving Shift-Reduce conflicts with precedence:

When the stack has `E + T` and input has `*`:
- Should we **reduce** E + T → E? Or **shift** `*`?
- Since `*` has **higher precedence** than `+`, we **shift** → this makes `*` bind tighter

When the stack has `T * F` and input has `+`:
- Should we **reduce** T * F → T? Or **shift** `+`?
- Since `*` has **higher precedence** than `+`, we **reduce** → `*` evaluated first

**Rule: if stack operator precedence ≥ input operator precedence → REDUCE, else → SHIFT**

### Full worked example: `x = 3 + 4 * 5 ;`

```
Step  Stack                  Input                   Action
1     $                      id = num + num * num ;  Shift 'id'
2     $ id                   = num + num * num ;     Shift '='  (id not reduced: next is =)
3     $ id =                 num + num * num ;       Shift 'num'
4     $ id = num             + num * num ;           Reduce: F → num
5     $ id = F               + num * num ;           Reduce: T → F
6     $ id = T               + num * num ;           Reduce: E → T
7     $ id = E               + num * num ;           Shift '+'
8     $ id = E +             num * num ;             Shift 'num'
9     $ id = E + num         * num ;                 Reduce: F → num
10    $ id = E + F           * num ;                 Reduce: T → F
11    $ id = E + T           * num ;                 DON'T reduce E+T (prec(+)=1 < prec(*)=2) → Shift '*'
12    $ id = E + T *         num ;                   Shift 'num'
13    $ id = E + T * num     ;                       Reduce: F → num
14    $ id = E + T * F       ;                       Reduce: T → T * F  (prec(*)=2 ≥ prec(;)=0)
15    $ id = E + T           ;                       Reduce: E → E + T  (prec(+)=1 ≥ prec(;)=0)
16    $ id = E               ;                       Shift ';'
17    $ id = E ;             $                       Reduce: S → id = E ;
18    $ S                    $                       ACCEPT
```

**Notice step 11**: the parser correctly SHIFTS `*` instead of reducing `E + T`, because `*` has higher precedence. This ensures `4 * 5` is computed before `3 + 4`.

### Shift-Reduce vs LL(1) comparison:

| Feature | LL(1) | Shift-Reduce |
|---------|-------|-------------|
| Direction | Top-down | Bottom-up |
| Handles left-recursion? | NO (must eliminate) | YES (directly) |
| Uses a table? | Yes (parsing table) | Can use precedence rules |
| Stack contents | Grammar symbols (to be matched) | Partially reduced input |
| Builds tree from | Root to leaves | Leaves to root |
| Grammar power | Weaker (no left-recursion, no ambiguity) | Stronger (handles more grammars) |

### Context-sensitive handle detection:

Some tokens need special treatment:
- `id` after `int`/`float` → don't reduce to F (it's a declaration variable name)
- `id` before `=` → don't reduce to F (it's the assignment target)
- `( E )` after `print` → don't reduce to F (it's print syntax, not grouping)

---

## 17. LL(1) vs Shift-Reduce — When to Use Which?

| Situation | Better choice |
|-----------|--------------|
| Simple expressions, statements | Either works |
| Grammar has left-recursion | Shift-Reduce (or eliminate recursion for LL(1)) |
| Need predictive parsing | LL(1) — you always know which production to use |
| Grammar is ambiguous with precedence | Shift-Reduce — precedence rules resolve conflicts naturally |
| Want automatic parser generators | Shift-Reduce (yacc/bison use LR variants) |
| Want hand-written code | LL(1) or recursive descent |

---

## 18. Previous Viva Questions — Lab 4 Specific

### Q: Why can't LL(1) handle left-recursive grammars?

Because with `E → E + T`, the parser would try to expand E, push E back on the stack, and try to expand it again — **infinite loop**. The solution is to rewrite as `E → T E'` and `E' → + T E' | ε`.

### Q: How does the shift-reduce parser know when to shift vs reduce?

Using **operator precedence**. If the operator on the stack has higher or equal precedence to the operator on the input, reduce. Otherwise, shift. For non-operator tokens (`;`, `)`, `$`), always reduce because they signal the end of an expression.

### Q: Walk through how `2 * (3 + 4)` is parsed.

**LL(1)**: The table expands `E → T E'`, then `T → F T'`, matches `num`, then `T' → * F T'` for `*`, then `F → ( E )` for the parenthesized expression, recursively parsing `3 + 4` inside.

**Shift-Reduce**: Shifts `num`, reduces to F→T. Shifts `*`. Shifts `(`. Inside parens: shifts `num` → F → T → E, shifts `+`, shifts `num` → F → T, reduces E+T → E. Shifts `)`. Reduces `(E)` → F. Then reduces `T*F` → T → E.

### Q: What are FIRST and FOLLOW sets used for?

- **FIRST**: tells the LL(1) parser which production to use when it sees a particular input token
- **FOLLOW**: tells the parser when to apply epsilon (ε) productions — if the input token is in FOLLOW(A), then A can produce nothing

### Q: Is this grammar LL(1)? How do you verify?

A grammar is LL(1) if its parsing table has **no multiply-defined entries** (no cell with two productions). For our grammar, all FIRST sets of alternatives for the same non-terminal are disjoint, and FOLLOW sets don't conflict with FIRST sets where ε is involved → it IS LL(1).

### Q: What is a handle in shift-reduce parsing?

A **handle** is a substring on the top of the parsing stack that matches the right-hand side of some production AND whose reduction leads to a valid parse. For example, if the stack top is `T * F`, the handle is `T * F` and it reduces to `T` via production `T → T * F`.

---

## Quick Reference Card (Last-Minute Revision)

```
┌──────────────────────────────────────────────────────────┐
│  LEXER = character → tokens    (catches: bad characters) │
│  PARSER = tokens → parse tree  (catches: bad structure)  │
├──────────────────────────────────────────────────────────┤
│  Token = <TYPE, "lexeme">  e.g. <ID, "x">               │
│  Lexeme = the actual text from source code               │
│  Pattern = the rule (regex) that defines a token type    │
├──────────────────────────────────────────────────────────┤
│  Maximal Munch: lexer reads LONGEST possible token       │
│  → "intx" = one ID, not keyword "int" + ID "x"          │
├──────────────────────────────────────────────────────────┤
│  FIRST(A) = tokens that can START a derivation from A    │
│  FOLLOW(A) = tokens that can come AFTER A                │
├──────────────────────────────────────────────────────────┤
│  Precedence: deeper in grammar = higher precedence       │
│  unary > * / % > + - > relational > ! > && > ||          │
├──────────────────────────────────────────────────────────┤
│  Panic Recovery: skip tokens → find ; or } → resume      │
├──────────────────────────────────────────────────────────┤
│  % = valid token (MODULO), so NOT a lex error            │
│  int int; = valid tokens, SYNTAX error (parser catches)  │
│  @ = NOT a valid token, so IS a lex error                │
├──────────────────────────────────────────────────────────┤
│  LL(1) = top-down, table-driven, needs NO left-recursion │
│    Stack starts with $ S, expand via table[NT][token]    │
│    Terminal on top → match; Non-terminal → look up table │
├──────────────────────────────────────────────────────────┤
│  Shift-Reduce = bottom-up, handles left-recursion        │
│    Shift tokens onto stack, reduce handles (RHS→LHS)     │
│    Precedence resolves conflicts: stk_op >= inp_op → RED │
├──────────────────────────────────────────────────────────┤
│  Left-recursion removal: E→Eα|β  becomes  E→βE' E'→αE'|ε│
│  Example: E→E+T|T  becomes  E→TE'  E'→+TE'|ε           │
├──────────────────────────────────────────────────────────┤
│  Handle = stack-top pattern matching a production RHS    │
│  Viable prefix = what can legally appear on SR stack     │
└──────────────────────────────────────────────────────────┘
```

---

**Good luck with your viva!**
