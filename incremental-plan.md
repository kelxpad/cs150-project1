# Incremental implementation roadmap

## Starting point and working rule

The shared AST and compilation fixtures are complete. The project is now in `D:\cs150\projects\cs150-project1`.

**One implementation request will cover one checkpoint below.** After its tests pass, I'll explain the change, show the diff, and stop for review. Completing one checkpoint will not authorize starting the next.

Each checkpoint should contain one coherent capability and a few focused tests. If it becomes substantially larger than expected, we split it before continuing.

## Phase 1: parser and checker

Build the parser in layers. Each layer produces the existing AST or supports the next parser layer.

| Step | Deliverable | Completion check |
|---|---|---|
| 1 | Parser foundation: shared `Result`, input cursor, parse errors, character/text matching, EOF | Success, failure, cursor advancement, and error offsets work |
| 2 | Parser combinators: sequencing, alternatives, repetition, optional input, separated lists | Backtracking restores the cursor; repetition cannot loop without consuming input |
| 3 | Whitespace, comments, identifiers, and keyword boundaries | Comments are skipped between elements; `letter` remains an identifier |
| 4 | Number, Boolean, string, and null literals | Correct AST values; malformed strings and non-finite numbers rejected |
| 5 | Type annotations | Primitive, function, and record types parse, including empty forms |
| 6 | Basic expressions | Literals, variables, parentheses, and unary operators parse |
| 7 | Binary operators | Precedence and left associativity match the grammar |
| 8 | Assignment expressions | Identifier-only targets and right associativity work |
| 9 | Records and postfix expressions | Record fields, calls, and field-access chains preserve source order |
| 10 | Arrow functions | Required annotations work; backtracking distinguishes arrows from parentheses |
| 11 | Basic statements | Declarations, expression statements, and nested blocks parse |
| 12 | Control flow and complete programs | `if`, optional `else`, and `while` parse; public `parse` rejects trailing text |
| 13 | Type validation and equivalence | Duplicate annotation names rejected; record order and function parameter names handled correctly |
| 14 | Basic expression checking | Literals, variables, operators, and assignment produce types or appropriate errors |
| 15 | Function and record checking | Annotations, return types, exact arity, fields, and higher-order types checked |
| 16 | Statement and program checking | Initialization, shadowing, duplicate locals, and block scopes follow the rules |

**Phase 1 is complete after step 16:** source text can be parsed and checked, with tests covering valid and rejected programs.

Section 1b acts as a checklist throughout these steps; it is not a separate implementation task.

## Phase 2: evaluator and complete runner

| Step | Deliverable | Completion check |
|---|---|---|
| 17 | Runtime values, environments, and store | Lookup, updates, fresh allocation, and retained locations work |
| 18 | Primitive operations | Float arithmetic, primitive equality, operand restrictions, and division-by-zero errors work |
| 19 | Basic expression evaluation | Variables, assignment, records, field access, evaluation order, and short-circuiting work |
| 20 | Declarations and blocks | Initializers run before binding; local bindings disappear while store updates survive |
| 21 | Conditionals and loops | Selected branches execute; each loop body has a fresh scope; false-condition effects survive |
| 22 | Functions and closures | Lexical capture, exact arity, argument order, fresh parameters, escaping closures, and recursion work |
| 23 | Complete public interface | `run` performs parse -> check -> execute; errors retain their stage; `getGlobal` retrieves results |
| 24 | Final specification audit | Every requirement and supplied example has appropriate coverage |

These checkpoints will test runtime behavior directly against ASTs until the final source-text pipeline is connected.

## Verification and scope controls

At every checkpoint:

1. Add focused tests for the new capability and relevant failure cases.
2. Preserve earlier passing tests.
3. Run the tests through the established `pure-hm` workflow.
4. Review the diff for changes outside that checkpoint.
5. Report what now works, what remains incomplete, and stop.

Use actual execution for decisive verification because typed-IR mode has the documented error-reporting issue. If the bridge hits its message-size limit, report that separately and make a tooling decision before further expansion.

No evaluator work will be bundled into a parser/checker checkpoint. No bridge modifications will be bundled into a language-feature checkpoint.

## Immediate next checkpoint

**Step 1 only: parser foundation.**

Add the small shared result type, parser input/error representation, character and literal-text primitives, and EOF checking. Add targeted tests and update the README’s commands to the new project directory.

Whitespace handling, literal parsing, grammar rules, checking, and evaluation belong to subsequent checkpoints.
