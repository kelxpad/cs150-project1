# MiniTS Interpreter - Project 1

## Current milestone: Step 1 - parser foundation

The dialect is `pure-hm`. The shared AST and its compilation fixtures are
complete. Step 1 adds parser input state, errors, character/text matching,
and EOF checking, with 14 focused tests. MiniTS grammar parsing, checking,
evaluation, and the public source runner remain later checkpoints.

The project specification is `Project 1 (MiniTS Interpreter) - CS 150 26.1.pdf`.
The compiled language reference is `lectures-8-10.pdf`.

## Library loading

`tests.kl` includes `syntax.kl`, `helpers/support.kl`, and `parser.kl` exactly once in
that order, and supplies the sole `main: IO Unit`. Library files have no
`main` or internal includes.

## Parser foundation

`support.kl` defines `Result a e = Ok a | Err e`. Parser primitives operate
directly on immutable `ParseState source offset` values, with a zero-based
offset into the KLang string. `Parsed value nextState` carries a success;
`ParseError offset message` records failure at the first mismatching position.
Callers retain the original state when a match fails.

| Entry point | Behavior |
| --- | --- |
| `runParser parser source` | Starts the parser at offset zero; does not require EOF. |
| `pAnyChar state` | Consumes one character or fails at EOF. |
| `pChar expected state` | Matches exactly one character; rejects other matcher lengths. |
| `pText expected state` | Matches exact text; empty text succeeds without advancing. |
| `pEof state` | Succeeds without advancing only at the end of input. |

Text matching has no keyword-boundary policy: `pText "let"` matches the prefix
of `letter`. Whitespace/comments are not skipped. Those policies and parser
combinators belong to later checkpoints. No public MiniTS `parse` exists yet.

## AST conventions

| Construct | Representation |
| --- | --- |
| Identifiers and ordered named items | `Identifier` aliases `String`; `Named a` pairs a name with an `a`. |
| Type annotations | `TNumber`, `TBoolean`, `TString`, `TNull`, `TFunction`, `TRecord`. |
| Literals and names | `ENumber Float`, `EBool Bool`, `EString String`, `ENull`, `EVar Identifier`. |
| Assignment and operations | `EAssign`, `EUnary`, `EBinary`, with explicit operator tags. |
| Expression-bodied arrows and calls | `EArrow [Named Type] Type Expr`, `ECall Expr [Expr]`. |
| Immutable records and field access | `ERecord [Named Expr]`, `EField Expr Identifier`. |
| Initialized declarations | `SLet Identifier (Option Type) Expr`. |
| Statements and blocks | `SExpr`, `SBlock`, `SIf`, `SWhile`. |
| Global statement sequence | `Program` aliases `[Stmt]`. |

- Every declaration has an initializer. `None` requests later type synthesis;
  `Some annotation` preserves an explicit annotation. The new binding becomes
  visible only after its initializer has been checked and evaluated.
- Every arrow parameter and return type has a required annotation. Nested
  arrows, function annotations, and calls represent higher-order functions.
- An assignment target is an identifier, and assignment is an expression.
  Reassigning a declared function variable can later create recursive behavior.
- Lists preserve source order for parameters, arguments, and record fields.
  Names and duplicates remain in the raw AST for the future checker.
- Function-type parameter names are retained for validation. Future semantic
  type equality must ignore those names and record-field order; ordinary
  structural AST equality does not implement that rule.
- `SIf` branches and `SWhile` bodies hold block contents. A missing `else`
  becomes an empty second branch. `SBlock` marks an explicit lexical block;
  a `Program` is a global sequence whose bindings remain available later.
- Expression nesting represents grouping and postfix chains. There is no
  separate parentheses node or mutable-record-field assignment node.
- Runtime closures, environments, stores, finite-number validation, and
  operator/type restrictions belong to later stages. AST construction alone
  does not establish that a MiniTS program is well typed or executable.

## Verify through the WSL KLang bridge

Run these commands as the same Ubuntu WSL user that owns the connected bridge.
The bridge client is installed in WSL; the compiler and runtime run in the
connected browser tab. Keep that tab open.

```sh
klangb status
```

Require `Connected` and `Runtime: ready`. If a connection is needed, run:

```sh
klangb connect --manual
```

Open its fresh URL in the browser's address bar while the command is waiting.
In Edge, allow **Apps on device** (loopback access) for `klang.upd-dcs.work`.
Allowing **Local network** alone does not grant loopback access. Retry the
connection if needed, then check status from a second WSL terminal.

The bridge requires an explicit dialect even though `dialect.txt` records the
submission choice. Type-check the real test entry point and its include:

```sh
klangb --emit-ir=typed --dialect pure-hm /mnt/d/cs150/projects/cs150-project1/tests.kl
```

Require exit status zero and typed IR output. Then run the fixture harness:

```sh
klangb run --dialect pure-hm /mnt/d/cs150/projects/cs150-project1/tests.kl
```

Require exit status zero and this output:

```text
AST fixtures compiled successfully.
Parser foundation: 14 tests passed.
```

Neither command writes generated Python: IR prints to standard output, and
`run` only writes generated Python when an output path is explicitly supplied.
The include stays within the entry point's directory, as required by the
bridge's source snapshots. Record the KLang revision from `klangb status` when
reporting verification results.

### Positive fixture coverage

- Every type, literal, unary/binary operator, expression, and statement variant.
- Both kinds of declaration annotation, nested assignment, nested scopes,
  explicit conditional branches, and an omitted `else` represented by `[]`.
- Empty programs, blocks and records; zero-parameter arrows and zero-argument
  calls; higher-order and nested record annotations.
- Left-associated subtraction, right-associated assignment, chained postfix
  operations, ordered arguments/fields, and duplicates retained for checking.
- The specification's factorial, counter closure, `makeAdder`, and student
  record examples, plus a higher-order argument and assignment-based recursion.

These are compilation fixtures. Their success does not establish MiniTS
scoping, type synthesis, evaluation order, short-circuiting, closure capture,
or the final globals shown in the specification.

### Negative compilation probes

During verification with KLang revision
`f032243eb13e2a41c51cccdda408dc1445a2da90`, typed-IR mode returned empty output
and exit status zero even for invalid programs. Use `run` for these probes:
its compilation stage reports the expected type diagnostics and exits before
executing an invalid program. An IR command's exit status alone is insufficient.

Keep invalid constructor uses outside `tests.kl`. Each probe must include the
syntax definitions, give the invalid expression an explicit expected `Expr`
or `Stmt` type, and supply `let main = pureIO unit;`.

| Probe | Invalid declaration |
| --- | --- |
| Missing initializer | `let invalid: Stmt = SLet "x" None;` |
| Missing parameter annotation | `let invalid: Expr = EArrow [Named "x" None] TNumber (EVar "x");` |
| Missing return annotation | `let invalid: Expr = EArrow [] (ENumber 1.0);` |
| Non-identifier assignment target | `let invalid: Expr = EAssign (EField (EVar "record") "field") (ENumber 1.0);` |

For file-based probes, create a temporary directory, copy `syntax.kl` into it,
and create one probe entry point beside that copy. Includes cannot escape the
snapshot's project directory. For example, in WSL:

```sh
python3 - <<'PY'
from pathlib import Path
from tempfile import TemporaryDirectory
import shutil
import subprocess

source = Path('/mnt/d/cs150/projects/cs150-project1/syntax.kl')
probes = {
    'missing-initializer': 'let invalid: Stmt = SLet "x" None;',
    'missing-parameter-annotation':
        'let invalid: Expr = EArrow [Named "x" None] TNumber (EVar "x");',
    'missing-return-annotation': 'let invalid: Expr = EArrow [] (ENumber 1.0);',
    'non-identifier-assignment':
        'let invalid: Expr = EAssign (EField (EVar "record") "field") (ENumber 1.0);',
}
with TemporaryDirectory(prefix='minits-ast-negative-') as temporary:
    directory = Path(temporary)
    shutil.copyfile(source, directory / 'syntax.kl')
    for name, declaration in probes.items():
        entry = directory / (name + '.kl')
        entry.write_text(
            'include "syntax.kl";\n' + declaration +
            '\nlet main = pureIO unit;\n', encoding='utf-8')
        result = subprocess.run(
            ['klangb', 'run', '--dialect', 'pure-hm', str(entry)],
            capture_output=True, text=True, timeout=45)
        diagnostic = result.stderr + result.stdout
        print(name, 'exit:', result.returncode)
        print(diagnostic)
        if result.returncode != 1 or 'error[type]' not in diagnostic:
            raise SystemExit('Expected a compiler type error for ' + name)
print('All four AST restriction probes passed.')
PY
```

Each probe must produce a constructor/type diagnostic for its invalid
declaration and a nonzero compiler exit status. A connection error, timeout,
or `client-*`/`bridge-*` error is an infrastructure failure and does not count
as a passing negative check. Explicit expected types ensure that a partially
applied constructor cannot silently become a valid fixture function.
