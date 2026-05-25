---
name: ocaml-code-style
description: "OCaml coding style and refactoring patterns. Use when the user asks to tidy, clean up, refactor, or improve OCaml code, reviewing code quality, enforcing naming conventions, or reducing complexity."
---

# OCaml Code Style

## Core Philosophy

1. **Interface-First**: Design `.mli` first. Clean interface > clever implementation.
2. **Modularity**: Small, focused modules. Compose for larger systems.
3. **Simplicity (KISS)**: Clarity over conciseness. Avoid obscure constructs.
4. **Explicitness**: Explicit control flow and error handling. No exceptions for recoverable errors.
5. **Purity**: Prefer pure functions. Isolate side-effects at edges.
6. **NEVER use Obj.magic**: Breaks type safety. Always a better solution.

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Files | `lowercase_underscores` | `user_profile.ml` |
| Modules | `Snake_case` | `User_profile` |
| Types | `snake_case`, primary type is `t` | `type user_profile`, `type t` |
| Values | `snake_case` | `find_user`, `create_channel` |
| Variants | `Snake_case` | `Waiting_for_input`, `Processing_data` |

**Function naming**:
- `find_*` returns `option` (may not exist)
- `get_*` returns value directly (must exist)

**Avoid**: Long names with many underscores (`get_user_profile_data_from_database_by_id`).

For test and fixture helpers, make the constructed value clear from the name.
Prefer `create_*` names for constructors such as `create_incrementing_bytes` or
`create_single_nonzero_lane` over ambiguous names like `lane_from` or `one_at`.

## Refactoring Patterns

### Option/Result Combinators

```ocaml
(* Before *)
match get_value () with Some x -> Some (x + 1) | None -> None

(* After *)
Option.map (fun x -> x + 1) (get_value ())
```

Prefer: `Option.map`, `Option.bind`, `Option.value`, `Result.map`, `Result.bind`

### Monadic Syntax (let*/let+)

Use `Result.Syntax` only when the project baseline is OCaml 5.4 or later. For
projects supporting OCaml 5.2 or 5.3, keep explicit matches for short chains or
use one shared project-local compatibility module.

```ocaml
(* Before - nested matches *)
match fetch_user id with
| Ok user -> (match fetch_perms user with Ok p -> Ok (user, p) | Error e -> Error e)
| Error e -> Error e

(* After *)
let open Result.Syntax in
let* user = fetch_user id in
let+ perms = fetch_perms user in
(user, perms)
```

### Pattern Matching Over Conditionals

```ocaml
(* Before *)
if x > 0 then if x < 10 then "small" else "large" else "negative"

(* After *)
match x with
| x when x < 0 -> "negative"
| x when x < 10 -> "small"
| _ -> "large"
```

## Function Design

**Keep functions small**: Under 50 lines. One purpose per function.

**Avoid deep nesting**: Max 4 levels of `match`/`if`. Extract helpers.

**High complexity signal**: Many branches = split into focused helpers.

```ocaml
(* Bad - high complexity *)
let check x y z =
  if x > 0 then if y > 0 then if z > 0 then ... else ... else ... else ...

(* Good - factored *)
let all_positive x y z = x > 0 && y > 0 && z > 0
let check x y z = if not (all_positive x y z) then "invalid" else ...
```

Avoid unnecessary local aliases for functions or operators that are clearer at
the call site. For module-scoped operators, prefer a local open around the
expression, for example `Int8x16.(left lor right)`, rather than rebinding the
operator with `let ( || ) = Int8x16.( lor )`.

## Error Handling

**Use `result` for recoverable errors**. Exceptions only for programming errors.

**Never catch-all**:
```ocaml
(* Bad *)
try f () with _ -> default

(* Good *)
try f () with Failure _ -> default
```

**Don't silence warnings**: Fix the issue, don't use `[@warning "-nn"]`.

## Library Preferences

| Instead of | Use | Why |
|------------|-----|-----|
| `Str` | `Re` | Better API, no global state |
| `Printf` | `Fmt` | Composable, type-safe |
| `yojson` (manual) | `jsont` | Type-safe codecs |

## Module Hygiene

**Abstract types**: Keep `type t` abstract. Expose smart constructors.

```ocaml
(* Good - .mli *)
type t
val create : name:string -> t
val name : t -> string
val pp : t Fmt.t
```

**Avoid generic names**: Not `Util`, `Helpers`. Use `String_ext`, `Json_codec`.

## API Design

**Avoid boolean blindness**:
```ocaml
(* Bad *)
let create_widget visible bordered = ...
let w = create_widget true false  (* What does this mean? *)

(* Good *)
type visibility = Visible | Hidden
let create_widget ~visibility ~border = ...
```

## Red Flags

- Match that just rewraps: `Some v -> Some (f v) | None -> None`
- Nested Result/Option matches → use let*/let+
- Deep if/then/else → pattern matching
- Missing `pp` function on types
- Unlabeled boolean parameters
- `Obj.magic` anywhere
