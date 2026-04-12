# Mini Compiler — Compiler Construction Lab

A mini compiler for a simple imperative language built incrementally across lab sessions.

## What it compiles

The language supports integer and float variables, arithmetic and boolean expressions, if-else, while loops, and print statements.

## Pipeline

- **Lexer** — tokenizes source code
- **Parser** — recursive descent, builds a parse tree
- **LL(1) + SLR** — table-driven parsers with FIRST/FOLLOW sets and stack traces
- **Symbol Table** — tracks variables with type, scope, and memory offset
- **Semantic Analyzer** — checks types, undeclared variables, and scope rules

## Build and run

```bash
g++ -std=c++17 -o compiler main.cpp
./compiler
```

Reads from `eval_program.txt` by default. Pass any file as an argument:

```bash
./compiler myprogram.txt
```
