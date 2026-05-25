---
name: ocaml-testing
description: Testing strategies for OCaml libraries. Use when discussing tests, Dune test setup, inline tests, expect tests, ppx_inline_test, ppx_expect, Alcotest, cram tests, property tests, Eio mocks, test structure, promotion workflows, or test-driven development in OCaml projects.
---

# OCaml Testing

## Core Philosophy

1. **Choose the right test style**: Use inline tests, expect tests, Alcotest,
   cram, property tests, or fuzz tests according to the contract being tested.
2. **Unit tests first**: Prioritize focused tests for individual modules and
   functions.
3. **Stable behavior coverage**: Test public functions exposed in `.mli` files,
   including success, error, and edge cases.
4. **Isolated tests**: Each test should be independent and avoid hidden external
   state.
5. **Clear test names**: Test names should describe what they test, not how.
6. **Test inclusion**: Make sure the relevant Dune test target actually
   exercises the code path, not just generated files.

## Choosing A Test Style

| Style | Use for | Typical Dune shape |
| --- | --- | --- |
| `let%test` / `let%test_unit` | Compact boolean/unit assertions close to code or in a test-only library | `(library ... (inline_tests) (preprocess (pps ppx_inline_test)))` |
| `let%expect_test` | Visible behavior, diagnostics, pretty-printer output, parser manifests, generated reports | `(library ... (inline_tests) (preprocess (pps ppx_expect)))` |
| Alcotest executable tests | Explicit runners, named suites, custom testables, existing Alcotest organization | `(test (name test) (libraries mylib alcotest ...))` |
| Cram tests | CLI/shell behavior and executable transcripts | `.t` files, `.t/` directories, and optional Dune `(cram ...)` stanzas |
| Property/fuzz tests | Parser/codec invariants, generated malformed input, roundtrips | QCheck/Crowbar/fuzzer-specific setup |

Do not force every project into an Alcotest runner. Dune-managed inline test
libraries do not need a hand-written `test.ml` runner. Conversely, when a
project already uses Alcotest, preserve the runner/suite organization unless
there is a clear reason to migrate.

Before adding an expect test, ask whether the printed output is itself the
reviewed contract. If the test only prints `true`, `false`, `Ok`, or another
throwaway assertion result, use `let%test` or `let%test_unit` instead. Expect
tests are for meaningful transcripts: pretty-printer output, diagnostics,
parser/conformance reports, CLI-like output, or other text whose exact shape
should be reviewed and promoted.

Avoid tests that only restate the behavior of upstream libraries or compiler
primitives through a thin wrapper. Prefer tests for local contracts, boundary
handling, composition, and invariants the project owns.

## Dune Inline Tests

Inline tests live in libraries, including test-only libraries under `test/`.
They are run by Dune through an inline-test backend.

```dune
(library
 (name test_mylib)
 (libraries mylib)
 (inline_tests
  (deps data.txt))
 (preprocess
  (pps ppx_expect)))
```

Use `ppx_expect` when any `let%expect_test` is present. Use
`ppx_inline_test` directly for plain `let%test` / `let%test_unit` suites that
do not need expect blocks.

If inline tests read files, declare them in `(inline_tests (deps ...))`. For
large fixture trees, use Dune dependency forms such as `(source_tree ...)` when
appropriate.

Verification commands:

```sh
dune build @runtest
dune runtest
dune build @runtest-mylib
```

`@runtest-mylib` is the library-specific inline-test alias for library `mylib`.
Prefer `dune build @runtest` for test-related changes when you need to validate
the full build/test graph.

### Inline-Test Runner Pitfalls

Do not infer whether inline tests are registered by reading
`.inline-tests/main.ml-gen`. A minimal generated runner such as:

```ocaml
let () = Ppx_inline_test_lib.exit ();;
```

can be normal. Tests are registered through linked modules and PPX-generated
side effects. To debug whether a test is exercised, run Dune normally, make a
temporary failing assertion, use inline-test runner flags such as `-verbose` or
`-only-test` when available, or inspect preprocessed modules for
`Ppx_inline_test_lib.test_unit`, `test_module`, or expect-test registration
sites. Treat generated runner internals as implementation details.

Inline-test execution follows module evaluation. Tests in functor bodies run
when the functor is applied. Side effects inside `module%test` are only
initialized when tests run, which is useful for test-only setup.

## Inline Assertion Tests

Use assertion-style inline tests for small, stable predicates:

```ocaml
let%test "empty input is invalid" =
  match Parser.parse "" with
  | Error Invalid_json -> true
  | Ok _ | Error _ -> false

let%test_unit "roundtrip small object" =
  let json = {|{"x":1}|} in
  match Parser.parse json with
  | Ok value -> assert (Parser.equal expected value)
  | Error err -> failwith (Format.asprintf "parse failed: %a" Parser.pp_error err)
```

Use `module%test` or `let%test_module` to group related setup:

```ocaml
module%test Parser_numbers = struct
  let%test _ = Parser.accepts "0"
  let%test _ = not (Parser.accepts "01")
end
```

Keep inline tests deterministic and avoid hidden dependencies on wall-clock
time, environment, random state, or filesystem layout.

## Expect Tests

Use expect tests when output is the contract or when behavior is easiest to
review as a concise transcript.

```ocaml
let%expect_test "parse error report" =
  Parser.parse {|{"x":}|} |> Parser.print_result;
  [%expect {| Error Invalid_json |}]
```

Each `[%expect]` block matches output produced since the previous expect block
or since the start of the test. Multiple expect blocks can make a scenario read
as steps.

Good expect tests:

- Write helper functions to set up scenarios concisely.
- Write custom pretty-printers that surface only the state the reader needs.
- Keep output deterministic; sort maps/sets before printing when needed.
- Avoid real I/O, sleeps, and wall-clock time unless carefully controlled.
- Use `%S` or explicit escaping for raw snippets where whitespace matters.
- Avoid dumping large unstable structures unless the large manifest is
  intentionally the reviewed artifact.

Expect-test failures produce diffs and corrected files. Review the diff before
accepting it.

```sh
dune runtest
dune promote
dune runtest --auto-promote
```

Use `dune promote` after inspecting the correction. Use `--auto-promote` only
when regenerated output is intentionally the new contract.

## Alcotest Directory Structure

Use an explicit `test/` runner structure when the project uses Alcotest:

```text
lib/
├── foo.ml
└── bar.ml
test/
├── dune
├── test.ml          # Main runner controlling initialization order
├── test_foo.ml      # suite : (string * unit Alcotest.test_case list) list
└── test_bar.ml
```

For single-module libraries, a single `test_foo.ml` as runner is acceptable.

## Alcotest Dune Configuration

```dune
(test
 (name test)
 (libraries mylib alcotest logs logs.fmt fmt.tty))
```

## Alcotest Main Runner Pattern

The main `test.ml` controls initialization order for side effects:

```ocaml
(* 1. Initialize RNG before any test module is loaded, if needed. *)
let () = Crypto_rng_unix.use_default ()

(* 2. Set up logging. *)
let () = Fmt_tty.setup_std_outputs ()
let () = Logs.set_reporter (Logs_fmt.reporter ())
let () = Logs.set_level (Some Logs.Debug)

(* 3. Run all test suites. *)
let () = Alcotest.run "mylib" Test_foo.suite
```

For multiple modules:

```ocaml
let () = Crypto_rng_unix.use_default ()
let () = Alcotest.run "mylib" (Test_foo.suite @ Test_bar.suite)
```

## Alcotest Module Test Pattern

Each module exports a `suite` value. Do not initialize RNG or run Alcotest here.

```ocaml
(** Tests for Foo module. *)

let test_basic () =
  let result = Foo.process "input" in
  Alcotest.(check string) "expected output" "output" result

let test_empty () =
  let result = Foo.process "" in
  Alcotest.(check string) "empty input" "" result

let suite =
  [
    ( "process",
      [
        Alcotest.test_case "basic" `Quick test_basic;
        Alcotest.test_case "empty_input" `Quick test_empty;
      ] );
  ]
```

## Alcotest Patterns

### Custom Testables

```ocaml
let result_testable ok_t =
  Alcotest.result ok_t Alcotest.string

let my_type_testable =
  Alcotest.testable My_type.pp My_type.equal
```

### Common Checks

```ocaml
Alcotest.(check int) "count" 42 actual
Alcotest.(check string) "name" expected actual
Alcotest.(check bool) "flag" true actual
Alcotest.(check (list int)) "items" [ 1; 2; 3 ] actual
Alcotest.(check (option string)) "maybe" (Some "x") actual
Alcotest.(check (result int string)) "result" (Ok 42) actual
```

### Testing Exceptions

```ocaml
let test_raises () =
  Alcotest.check_raises "should fail" (Invalid_argument "bad") (fun () ->
      Foo.parse "bad")
```

### Running Alcotest Executables

Dune attaches `(test ...)` stanzas to `@runtest`, so prefer `dune runtest` for
normal verification. To run a test executable manually, pass arguments after the
executable name:

```sh
dune exec -- test/test.exe
dune exec -- test/test.exe test "suite_name"
```

Alcotest filters by test name and case index through its command-line query
language. Do not assume that a test case label can always be selected directly
by name; check the executable help or run `list` when narrowing a suite.

## Initialization Order And Lazy State

For executable test runners, initialize global state before `Alcotest.run`:

1. **RNG**: `Crypto_rng_unix.use_default ()` or
   `Mirage_crypto_rng_unix.use_default ()`.
2. **Logging**: `Logs.set_reporter` and `Logs.set_level`.
3. **Other global state**: environment setup, temp directories, mock services.

This ensures deterministic test ordering and proper side-effect sequencing.

For inline tests, there may be no central `test.ml`. Avoid module-load-time
side effects unless they are intentional. If a test module needs state at load
time, use lazy evaluation:

```ocaml
let key = lazy (Crypto_rng.generate 32)
let key () = Lazy.force key

let test_encrypt () =
  let ciphertext = Foo.encrypt ~key:(key ()) plaintext in
  ...
```

This defers setup until tests actually run.

## Test Logging

Set up logging in tests using the standard `Logs` library:

```ocaml
let () = Fmt_tty.setup_std_outputs ()
let () = Logs.set_reporter (Logs_fmt.reporter ())
let () = Logs.set_level (Some Logs.Debug)
```

Defaulting tests to debug logging is useful because Alcotest captures output by
default, so verbose logs do not clutter successful terminal output. Captured
output is shown when tests fail.

For per-source control, set levels after the reporter:

```ocaml
let () = Logs.Src.set_level Conpool.src (Some Logs.Debug)
let () = Logs.Src.set_level Requests.src (Some Logs.Warning)
```

## Naming Conventions

- **Test suite names**: lowercase, single words such as `"users"`,
  `"commands"`, `"process"`.
- **Test case names**: lowercase with underscores, concise but descriptive,
  such as `"basic"`, `"empty_input"`, `"parse_error"`.
- **Expect test names**: describe the visible behavior or report being frozen.
- **Inline anonymous tests**: use `_` only for tiny local invariants where a
  name would not add useful context.

## Writing Good Tests

**Function coverage**: Test all public functions exposed in `.mli` files,
including success, error, and edge cases.

**Test data**: Use helper functions to create test data:

```ocaml
let make_user ?(name = "test") ?(id = 1) () = User.v ~name ~id
```

For repeated fixture construction, extract small helpers at the nearest shared
test scope. If all tests already live inside one `module%test`, put the helpers
directly at that module's top level rather than adding an unnecessary nested
`Test_helpers` module.

**Failure quality**: Failures should identify the broken case or fixture, not
only the larger suite.

**Parser contracts**: For parsers, distinguish ordinary input rejection from
implementation defects. Recoverable invalid input should usually be asserted as
`Error`; internal exceptions should be explicit and reserved for bugs or
unimplemented paths when that is the contract.

## Property-Based Testing

For complex logic, use property tests when a broad invariant is clearer than a
long list of examples:

```ocaml
let test_roundtrip =
  QCheck.Test.make ~count:1000 ~name:"encode then decode is identity"
    QCheck.string
    (fun s -> Codec.decode (Codec.encode s) = Ok s)
```

For parsers and encoders, prioritize:

- parse/print/parse roundtrips
- accepted input stays accepted after normalization
- malformed input returns ordinary parser errors, not internal exceptions
- boundary sizes, nesting, number formats, escaping, and invalid encodings

Use fuzz tests for long-running malformed-input campaigns or security-sensitive
parser work.

## Testing Eio Code

When testing Eio code, also use the `ocaml-codex:eio` skill. Its
`skills/eio/references/testing.md` file is the source of truth for `eio.mock`
setup, mock flows, chunked reads, mock networks, deadlock detection, and backend
exception normalization.

Keep this skill focused on test organization: choose whether Eio behavior fits
an inline test, an Alcotest runner, or a Cram/integration test; then follow the
Eio skill for the Eio-specific mechanics. When scaffolding a new Eio mock test
file, adapt `skills/ocaml-testing/templates/test_eio_mock.ml` only after
checking the Eio testing reference.

## End-to-End Testing With Cram

Cram tests verify CLI executable behavior: command lines, exit status,
stdout/stderr, and shell-visible integration.

Use a single `.t` file for self-contained shell transcripts. Use a directory
ending in `.t` with a `run.t` file when the test needs fixture files, binary
data, or multiple inputs.

```text
test/
└── help.t
```

Create actual test files. Avoid embedding nontrivial code within `run.t` using
heredocs.

```text
test/
└── my_feature.t/
    ├── run.t
    ├── input.txt
    └── expected.json
```

Keep library semantics in OCaml tests; use cram for the executable interface.
When a Cram test invokes a built executable, declare it in a `(cram (deps ...))`
stanza, commonly with `%{bin:tool}` for installed public names or the executable
path for private test binaries.

## Test Quality Checklist

- The relevant Dune target exercises the code path.
- Fixture dependencies are declared in Dune.
- Test names describe behavior.
- Failures identify the broken case or fixture.
- Public `.mli` behavior has success, error, and edge-case coverage.
- Recoverable errors are asserted as `Error`.
- Internal exceptions are asserted only when that is explicitly the contract.
- Expect output is concise, deterministic, and intentionally promoted.
- Broad manifests report unexpected passes and failures, not only totals.
- Test setup avoids accidental global state and hidden filesystem assumptions.

## Running Tests

```sh
# All default tests
dune build @runtest
dune runtest

# Watch for changes
dune runtest -w

# Specific inline-test library
dune build @runtest-mylib

# Alcotest executable, when present
dune exec -- test/test.exe
dune exec -- test/test.exe test "suite_name"

# Promote reviewed expect-test corrections
dune promote

# Auto-promote intentional expect output changes
dune runtest --auto-promote

# Coverage, when bisect_ppx is configured
dune runtest --instrument-with bisect_ppx
bisect-ppx-report summary
```
