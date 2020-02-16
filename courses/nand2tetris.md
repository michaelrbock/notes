# nand2tetris: Part 1

[Course](https://www.coursera.org/learn/build-a-computer)

## Week 1

### Introduction

Understanding compueters is all about *abstraction*.

Each time you use a lower-level of abstraction, you assume it works, without worrying about *how*.

### Boolean Functions and Gate Logic

`AND`, `OR`, `NOT`.

Parenthesis for precedence.

Because the inputs to a boolean function are finite (just 0 or 1), we can easily create a *truth table* with the entirety of all possible inputs and their outputs.

#### Laws

Commutative: `x AND y = y AND x`

Associative: `x AND (y AND z) = (x AND y) AND z

Distributive: `x AND (y OR z) = (x AND y) OR (x AND z)

De Morgan: `NOT(x AND y) = NOT(x) OR NOT(y)`

Idempotence: `x AND x = x`

Double Negation: `NOT(NOT(x)) = x`

#### Boolean Function Syntesis

Take a truth table, create `AND` functions for each 1 output and then `OR` them together then simplify. (It's an NP-Hard problem to automatically find the shortest expression).

Any Boolean function can be represented with *just* `AND` and `NOT` operations. (`x OR y` = `NOT(NOT(x) AND NOT(y))`. `x NAND y` = `NOT(x AND y)`. So, any Boolean function can be represented with *just* `NAND` operations.

#### Logic Gates

Elementary: Nand, And, Or, Not, ...

Composite: 3-way And, Mux, Adder, ...

Interface = input/out abstraction for the user. Implementation = internals.

Circuit implementations uses latches (EE, not CS).

