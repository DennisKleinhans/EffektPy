This document outlines the internal architecture of the EffektPy interpreter. The project leverages **Algebraic Effects** provided by the Effekt language to handle compiler phases, state management, and side effects in a modular and functional way. 

## Architecture Overview

The interpreter follows a multi-stage pipeline. Each stage isolates specific concerns and communicates via effects. 
1. **Lexer**: Tokenizes input and attaches position information. 
2. **Parser**: Converts tokens into a Surface Syntax AST. 
3. **Desugarer**: Transforms the Surface AST into a simplified Core AST.
4. **Typechecker**: Infers and validates types using a Two-Phase pass. 
5. **Evaluator**: Executes the Core AST using a Two-Phase allocation strategy. 

### Directory Structure

```text 
src/ 
├── lib/ 
│ ├── desugar/ # Desugaring logic & Core AST 
│ ├── interpreter/ # Evaluation logic and runtime values 
│ ├── lexer/ # Tokenization and positioning effects 
│ ├── parser/ # Parsing logic and Surface AST 
│ ├── typechecker/ # Type inference and checking 
│ ├── runner/ # Pipeline orchestration (REPL & CLI) 
│ └── shared/ # common used effects, handlers and helpers (Errors, Logging) 
└── main.effekt # Entry point (CLI Argument Dispatcher)
```

## Detailed Component Analysis

### 1. Lexer & Positioning (`src/lib/lexer`)

The lexer is the first stage of the pipeline. It reads the raw character stream and produces a list of `Token`s.

- **Position Tracking**: Instead of manually passing line and column numbers around, the lexer utilizes a scoped **`Positioning` effect**.
- **Mechanism**: As the lexer advances through the source string, the effect handler updates the current state. When a token is created, it captures the current position from the effect. This allows precise error reporting ("Error at line 10, col 5") in later stages like parsing or typechecking without polluting the function signatures.

### 2. Parser (`src/lib/parser`)

The parser consumes the stream of tokens to produce the **Surface AST**.

- It handles operator precedence and structural rules (e.g., block nesting).
- It relies on the `Positioning` effect (propagated from the tokens) to annotate AST nodes with their source location.

### 3. Desugaring (`src/lib/desugar`)

Before typechecking, the **Surface AST** is transformed into a simplified **Core AST**. This stage reduces the complexity of the language by removing syntactic sugar.

- **Transformation Examples**:
    - Compound assignments (`a += b`) are expanded into standard assignments (`a = a + b`).
    - Implicit returns are made explicit in the AST structure if necessary.
- **Benefit**: The subsequent typechecker and interpreter only need to handle a minimal set of instructions, reducing code duplication.

### 4. Typechecker (`src/lib/typechecker`)

The typechecker implements a **Two-Phase Bidirectional Type Checking** system to support complex recursion patterns (like mutual recursion) without requiring forward declarations.

- **Phase 1 (Discovery)**: The typechecker scans a block/scope to identify all variable and function names. It registers their signatures in the environment using fresh type variables (placeholders).
- **Phase 2 (Validation)**: It visits the expressions again to solve the type equations. Because the names were registered in Phase 1, a function `A` can refer to a function `B` that is defined later in the code.
- **Effects**:
    - `Context[TypeBinding]`: To access variable definitions.
    - `Typing`: To manage unification equations.
    - `Exception[TypeError]`: To reject invalid programs (e.g., type mismatches, calling a non-function).

### 5. Interpreter (`src/lib/interpreter`)

The interpreter executes the Core AST. Similar to the typechecker, it uses a **Two-Phase Evaluation Strategy** to handle scoping correctly.

- **Phase 1 (Allocation)**: Before evaluating a block, the interpreter reserves memory addresses in the `Store` for all variables declared in that scope.
- **Phase 2 (Evaluation)**: The expressions are evaluated, and values are written to the pre-allocated addresses.
- **Closures**: This strategy ensures that when a closure is created, it captures an environment that already contains the addresses of its sibling functions, enabling **mutual recursion**.
- **Environment vs. Store**:
    - **Environment**: Maps names to memory addresses.
    - **Store**: Maps memory addresses to values. This separation allows for correct variable mutation semantics.

### 6. Built-ins & Standard Library

The system is bootstrapped with a standard library.

- **Variadic Support**: The interpreter supports variadic functions. The `print` function accepts `0..n` arguments, while `min` and `max` are defined to require at least two numeric arguments. This is enforced during the typechecking phase to prevent runtime errors like `min()` with no values. 
- **Implementation**: Built-ins are injected into the initial `TypeEnv` as specialized function types (e.g., using variadic type variables) and mapped to native Effekt implementations in the `RuntimeEnv`.

### 7. The REPL (`src/lib/runner/runRepl`)

The REPL orchestrates the incremental execution of the pipeline.

- **Persistence**: It maintains the state (`TypeEnv`, `Store`, `RuntimeEnv`) across iterations.
- **Brace Counting**: To support multi-line input (like function definitions), the REPL uses a heuristic to detect open braces and pauses evaluation until the input block is balanced.

### 8. Pipeline Orchestration & Runners (`src/lib/runner`)

The Runner module serves as the entry point for the interpreter, bridging the gap between effectful compiler stages and the final user output. It is responsible for error translation, state persistence, and command-line orchestration.

#### The `PipelineResult` Pattern

To ensure that the interpreter doesn't simply crash when an error occurs, every major stage returns a `PipelineResult[T]`.

```scala
type PipelineResult[T] {
  Success(data: T)
  Failure(msg: String)
}
```

This Algebraic Data Type (ADT) allows the runners to catch Exceptions and transform them into a data representation that can be easily matched and printed.

#### Boundary Handlers & Error Translation

Each runner (e.g., `runTypecheck`, `runEval`) acts as a **boundary handler**. They encapsulate the internal effectful logic and provide a clean, non-crashing interface.

- **Effect Conversion**: Inside a runner, a `try { ... } with Exception[...]` block is used. If a stage raises an exception (like a `TypeError` deep in the tree), the runner intercepts it and returns a `Failure("TypeError: ...")`.
- **Positioning Integration**: Runners maintain a local `currentPos` state. They handle the `Positioning` effect, providing the current line and column information to any stage that needs to report an error or annotate an AST node.

#### Incremental Execution & State Persistence

For the **REPL**, the runners support an incremental mode. This is implemented by passing state objects back and forth:

- **`EvalState`**: A record that captures the snapshot of the interpreter at a specific moment.
```scala
record EvalState(value: Value, env: Map[String, Address], store: Map[Address, Value])
```
- **Persistent Environments**: The REPL loop in `startRepl` maintains variables for the `typeEnv`, `runtimeEnv`, and `store`. After a successful `runEvalIncremental`, these variables are updated with the new state returned in the `PipelineResult::Success(newEvalState)`.

#### Multi-line Input Logic

The REPL implementation includes a `readInputBlock` function that uses a **brace-counting heuristic**. It suppresses evaluation and prompts for more input (`...` ) as long as the curly braces `{}` or parentheses `()` are not balanced. This allows users to define complex functions and blocks directly in the interactive shell.