---
name: ocaml-fuzz
description: "OCaml fuzz testing with Crowbar for protocol implementations. Use when Codex needs to: (1) Write fuzz tests for parsers and encoders, (2) Test roundtrip invariants (parse(encode(x)) = x), (3) Verify boundary conditions and error handling, (4) Test state machines and transitions, (5) Organize fuzz test suites for large codebases, (6) Run long-lived AFL campaigns with Crowbar"
---

# OCaml Fuzz Testing with Crowbar

## Core Philosophy

1. **One fuzz file per module**: `fuzz_foo.ml` tests `lib/foo.ml`. Keeps tests organized and discoverable.
2. **Roundtrip everything**: If you have `encode` and `decode`, test `decode(encode(x)) = x`.
3. **Crash-safety first**: Parsers must never crash on arbitrary input, even malformed data.
4. **Boundary conditions matter**: Test edge cases (0, max values, empty input, overflow).
5. **State machines need transition coverage**: Test all valid and invalid state transitions.

## Build Configuration

### Simple single-file setup (per-package)

For standalone packages, use one fuzz file per package:

```
ocaml-foo/
├── lib/
├── fuzz/
│   ├── dune
│   └── fuzz_foo.ml
└── dune-project
```

**fuzz/dune:**

```lisp
(executable
 (name fuzz_foo)
 (modules fuzz_foo)
 (libraries foo crowbar))

; Quick check with Crowbar (no AFL instrumentation)
(rule
 (alias fuzz)
 (deps fuzz_foo.exe)
 (action
  (run %{exe:fuzz_foo.exe})))

; AFL-instrumented build target (use with --profile=afl)
(rule
 (alias fuzz-afl)
 (deps
  (source_tree input)
  fuzz_foo.exe)
 (action
  (echo "AFL fuzzer built: %{exe:fuzz_foo.exe}\n")))
```

**Seed corpus**: Create `fuzz/input/` with sample inputs:

```bash
mkdir -p fuzz/input
echo -n "" > fuzz/input/empty
# Add representative samples as seed inputs
```

**fuzz/fuzz_foo.ml:**

```ocaml
open Crowbar

let test_parse_crash_safety buf =
  ignore (Foo.parse buf);
  check true

let () =
  add_test ~name:"foo: parse crash safety" [ bytes ] test_parse_crash_safety
```

### Multi-module setup (large codebases)

For larger projects with many modules:

```lisp
(executable
 (name fuzz)
 (libraries crowbar borealis)
 (modules
  fuzz
  fuzz_common
  fuzz_foo
  fuzz_bar))
```

Main entry point (`fuzz/fuzz.ml`):

```ocaml
(* Force linking of modules that register tests via side effects *)
let () =
  Fuzz_common.run ();
  Fuzz_foo.run ();
  Fuzz_bar.run ()
```

Each fuzz module ends with:

```ocaml
let run () = ()
```

This ensures the module is linked and its `add_test` calls execute.

---

## Style Guidelines

When writing fuzz tests, follow these conventions:

1. **Define test functions separately** at the top of the file
2. **Register all tests at the end** with grouped `add_test` calls
3. **Use `bytes` directly** instead of custom generators
4. **Use a `truncate` helper** to limit input size for protocol messages
5. **Return `()` directly** - no need for `check true` in most cases
6. **Add `Crypto_rng_unix.use_default ()`** at the top if crypto is used

### Example structure

```ocaml
(** Fuzz tests for Foo module. *)

open Crowbar
open Fuzz_common

(** Decode - must not crash on arbitrary input. *)
let test_decode buf =
  let buf = truncate buf in
  let _ = Foo.decode (to_bytes buf) in
  ()

(** Roundtrip - valid values must round-trip. *)
let test_roundtrip buf =
  let buf = truncate buf in
  match Foo.decode (to_bytes buf) with
  | Error _ -> ()
  | Ok v ->
      let encoded = Foo.encode v in
      match Foo.decode encoded with
      | Error _ -> fail "re-decode failed"
      | Ok v' -> if v <> v' then fail "roundtrip mismatch"

(** Pretty-print - must not crash. *)
let test_pp n =
  let v = Foo.of_int (n mod 4) in
  let _ = Format.asprintf "%a" Foo.pp v in
  ()

(* All add_test calls in run function - no side effects at module init *)
let run () =
  add_test ~name:"foo: decode crash safety" [ bytes ] test_decode;
  add_test ~name:"foo: roundtrip" [ bytes ] test_roundtrip;
  add_test ~name:"foo: pp" [ uint8 ] test_pp
```

**Main entry point (fuzz/fuzz.ml):**

```ocaml
(* Initialize crypto RNG if needed by any module *)
let () = Crypto_rng_unix.use_default ()

(* Register all fuzz tests *)
let () =
  Fuzz_common.run ();
  Fuzz_foo.run ();
  Fuzz_bar.run ()
```

---

## Test Patterns

Choose patterns from the table below, then load `references/test-patterns.md`
for concrete Crowbar implementations and generator details.

| Pattern | Use when |
|---------|----------|
| Crash-safety | Parser, decoder, reader, or `of_*` function accepts untrusted input |
| Encode/decode roundtrip | Valid values can be encoded and decoded |
| Constrained type roundtrip | Smart constructor accepts a bounded domain |
| Boundary tests | Type has min/max valid values or protocol limits |
| Invalid input rejection | Values outside a valid range must be rejected |
| Pretty-printer safety | Public values have `pp` functions |
| State machine transitions | API has valid and invalid state transitions |
| Unit conversion roundtrips | API converts between units or representations |
| Resource operations | API creates, stores, fetches, or deletes resources |

---

## Common Module: fuzz_common.ml

```ocaml
(** Common utilities for fuzz tests. *)

open Crowbar

let to_bytes buf =
  let len = String.length buf in
  let b = Bytes.create len in
  Bytes.blit_string buf 0 b 0 len;
  b

let catch_invalid_arg f =
  try f () with Invalid_argument _ -> check true

let run () = ()
```

---

## File Organization

```
fuzz/
├── fuzz.ml              # Main entry, links all modules
├── fuzz_common.ml       # Shared utilities
├── fuzz_tc_frame.ml     # Tests for lib/frames/tc_frame.ml
├── fuzz_tm_frame.ml     # Tests for lib/frames/tm_frame.ml
├── fuzz_apid.ml         # Tests for lib/frames/apid.ml
├── fuzz_keyid.ml        # Tests for lib/sdls/keyid.ml
└── ...
```

**Naming convention**: `fuzz_<module>.ml` tests `lib/**/<module>.ml`

---

## Running Fuzz Tests

### Without AFL (quick check)

```bash
dune exec fuzz/fuzz.exe
# Or use the alias:
dune build @fuzz
```

### With AFL (thorough fuzzing) - Manual

```bash
dune build fuzz/fuzz.exe
mkdir -p fuzz/input
echo -n "" > fuzz/input/empty
afl-fuzz -m none -i fuzz/input -o _fuzz -- \
  _build/default/fuzz/fuzz.exe @@
```

### With crow (recommended for multiple targets)

Use **crow** to orchestrate long-running AFL campaigns across multiple targets:

```bash
# Initialize workspace (creates dune-workspace with afl profile if needed)
crow init

# Build all fuzz targets with AFL instrumentation
dune build --profile=afl @fuzz-afl

# List discovered fuzz targets
crow list

# Start a campaign with 8 CPUs for 24 hours
crow start --cpus=8 --duration=24h

# Monitor progress
crow status

# Stop the campaign
crow stop
```

crow automatically:
- Discovers all `*/fuzz/dune` files with crowbar dependencies
- Allocates CPU cores across targets (main + secondary instances)
- Creates/updates `dune-workspace` with AFL profile if missing
- Tracks campaign state and aggregates statistics

### Check for duplicate test names

```bash
grep -h 'add_test ~name:"' fuzz/fuzz_*.ml | \
  sed 's/.*~name:"\([^"]*\)".*/\1/' | sort | uniq -d
```

## Coverage Checklist

For each module with a public API (`.mli` file):

- [ ] **Crash safety**: All `decode_*`, `parse_*`, `read_*`, `of_*` functions
- [ ] **Roundtrip**: All `encode`/`decode`, `to_*`/`of_*` pairs
- [ ] **Boundaries**: Min/max valid values, edge cases
- [ ] **Invalid input**: Values outside valid range rejected
- [ ] **State machines**: All transitions (valid and invalid)
- [ ] **Pretty-printers**: All `pp_*` functions don't crash
- [ ] **Comparison**: `equal` and `compare` are consistent
