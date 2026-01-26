# EffektPy

EffektPy is a lightweight interpreter for a Python-like programming language, implemented purely in **Effekt**. It features a fully functioning pipeline including a lexer, parser, desugarer, typechecker, and evaluator, all built using algebraic effects to manage state, environments, and error handling.

## Features

* **Python-like Syntax**: Familiar syntax with braces `{}` for blocks.
* **Type Inference**: Types for variables and function parameters can often be inferred, making type annotations optional in many cases.
* **Smart Recursion**: Support for **mutual recursion** and **nested recursion** (functions defined inside other functions).
* **Interactive REPL**: A Read-Eval-Print Loop with persistent state, multi-line input support, and smart history.
* **First-Class Functions**: Support for closures, higher-order functions, lambdas and default parameter values.
* **Control Flow**: `if/else`, `while` loops, `break`, `continue`, and implicit returns.
* **Built-in Standard Library**: Includes essential functions like `print`, `input`, and string conversions.
* **Syntactic Sugar**: Supports convenient operators like `+=` and `-=`.

## Prerequisites

To build and run this project, you need:

1.  **The Effekt Compiler**: [Install Effekt](https://effekt-lang.org/docs/getting-started)
2.  **Node.js**: Required as the JavaScript backend runtime.

## Installation & Building

Compile the project to JavaScript:

```bash
effekt --compile --backend=js src/main.effekt
```

## Running the Interpreter

Once compiled, the executable JavaScript file is located in the `out` directory (typically named `main__main.js`). You use Node.js to execute it.


**1. Start the REPL (Interactive Mode):**
Run without arguments to enter the interactive shell.
```bash
node out/main__main.js
```

**2. Execute a File:**
Pass a filename to execute a specific script.
```bash
node out/main__main.js examples/fibonacci.pff
```

**3. Show Help:**
```bash
node out/main__main.js --help
```

## Language Guide

### Variables & Type Inference

You can annotate types explicitly, but EffektPy can also infer them.

```scala
// Explicit typing
val pi: Int = 3
var count: Int = 0

// Type Inference (types are automatically detected)
val num = 42
var active = true

// Syntactic sugar for assignment
count += 1
```

### Functions & Recursion

Functions support **implicit returns** (the last expression is returned) and **default parameter values**.

```Scala
// Return type inferred, 'return' keyword omitted
def add(a, b) {
    a + b
}

// Default arguments
def add(a, b = 42) {
    a + b
}

add(8) // Returns 50
```

EffektPy handles complex recursion patterns, including **mutual recursion** (functions calling each other) and **nested recursion**, without needing forward declarations.
```scala
def isEven(n) {
    if n == 0 then true else isOdd(n - 1)
}

def isOdd(n) {
    if n == 0 then false else isEven(n - 1)
}

print(isEven(4)) // true
```

EffektPy's type inference is powerful enough to handle most Higher-Order Functions (HOFs) automatically.
```scala
// Type inference works even for HOFs 
def apply(f, x) { 
	f(x) 
} 
def increment(n) { n + 1 } 
apply(increment, 10) // Returns 11
```

**Note on Advanced Types:** While inference works for standard cases, explicit type annotations are required for complex scenarios such as **Rank-2 types** (functions that take polymorphic functions as arguments) or **self-application** patterns.

### Standard Library (Built-ins)

The interpreter comes with a set of built-in functions available in the global scope:

- **`print(args...)`**: Prints values to the console (accepts variable arguments of different types).
- **`input(prompt = "")`**: Reads a string from standard input.
- **`str(value)`**: Converts any value to its string representation.
- **`min(a, b, ...)`** / **`max(a, b, ...)`**: Returns the minimum or maximum of at least two numbers.

## The REPL

The REPL supports incremental type checking and evaluation. It features a brace-counting heuristic to support **multi-line input**.
```
> def fib(n) {
...   if (n <= 1) { n }
...   else { fib(n-1) + fib(n-2) }
... }
> fib(10)
55
```
