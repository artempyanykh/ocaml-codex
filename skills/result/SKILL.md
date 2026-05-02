---
name: result
description: "OCaml Result type patterns using the stdlib. Use when Codex needs to: (1) Handle errors with Result types, (2) Chain Result operations with let* where the project baseline supports it, (3) Extract values from Ok/Error, (4) Refactor Result-heavy code without adding unnecessary local operators"
---

# OCaml Result Patterns

The OCaml stdlib provides `Result` helpers for explicit error handling. `Result.Syntax`
is available in OCaml 5.4 and later; projects supporting OCaml 5.2 or 5.3 need a
small compatibility module or explicit `Result.bind`/`match` code instead.

## Result.Syntax

For projects with `ocaml >= 5.4`, use `open Result.Syntax` to get `let*` and
`let+` bindings:

```ocaml
open Result.Syntax

let process request =
  let* req = validate request in
  let* auth = authenticate req in
  let* _ = authorize auth in
  execute req
```

For projects that support OCaml 5.2 or 5.3, do not use `Result.Syntax`.
Prefer explicit control flow for short chains:

```ocaml
let process request =
  match validate request with
  | Error _ as err -> err
  | Ok req ->
      match authenticate req with
      | Error _ as err -> err
      | Ok auth ->
          match authorize auth with
          | Error _ as err -> err
          | Ok () -> execute req
```

If a project uses Result syntax pervasively while supporting OCaml < 5.4, define
one project-local compatibility module and reuse it. Avoid scattered local
definitions of `let ( let* ) = Result.bind`.

## Extracting Values

| Function | Behavior on Error |
|----------|-------------------|
| `Result.get_ok r` | Raises `Invalid_argument` |
| `Result.get_error r` | Raises `Invalid_argument` (OCaml >= 5.4) |
| `Result.value r ~default` | Returns default |

Use `Result.get_ok` only when failure is a programming error:

```ocaml
(* Startup/config - crash on failure is intentional *)
let config = Result.get_ok (Config.load ())

(* Test setup - failure means test bug *)
let client = Result.get_ok (Tls.Config.client ~authenticator ())
```

## Custom get_ok

Only define custom `get_ok` when you need different exception behavior:

```ocaml
(* Raises domain-specific Protocol_error instead of Invalid_argument *)
let get_ok = function
  | Ok x -> x
  | Error e -> raise (Protocol_error e)
```

If you just want `Invalid_argument`, use `Result.get_ok` directly.
