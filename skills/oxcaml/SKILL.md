---
name: oxcaml
description: Working with OxCaml extensions to OCaml. Use when the oxcaml compiler is available and Codex needs high-performance OCaml with modes, stack allocation, unboxed types, zero-allocation checking, SIMD, comprehensions, or data-race-free parallelism.
---

# OxCaml Development

Use this skill when the project is compiled with OxCaml or explicitly opts into
Jane Street OCaml extensions. Assume standard OCaml knowledge and load only the
reference files for the features being used.

## Reference Selection

| Need | Read |
|------|------|
| Locality, uniqueness, portability, contention, modal types | [references/modes.md](references/modes.md) |
| `local_`, `stack_`, `exclave_`, local returns | [references/stack-allocation.md](references/stack-allocation.md) |
| `float#`, `int32#`, unboxed records, mixed blocks | [references/unboxed.md](references/unboxed.md) |
| Kinds such as `value`, `float64`, `bits32`, kind-polymorphic APIs | [references/kinds.md](references/kinds.md) |
| `unique`, `aliased`, `once`, `many`, ownership-sensitive APIs | [references/uniqueness.md](references/uniqueness.md) |
| List and array comprehensions | [references/comprehensions.md](references/comprehensions.md) |
| Vector types, SSE/AVX intrinsics, numeric kernels | [references/simd.md](references/simd.md) |
| `ppx_template`, generated specialisations, name mangling | [references/templates.md](references/templates.md) |
| `[@zero_alloc]`, allocation checking, performance annotations | [references/zero-alloc.md](references/zero-alloc.md) |
| Jane Street Base extensions for OxCaml | [references/base.md](references/base.md) |
| Jane Street Core extensions for OxCaml | [references/core.md](references/core.md) |

## Quick Syntax

```ocaml
(* Stack allocation and locality *)
let make_pair x y = exclave_ stack_ (x, y)
let consume (x @ local) = ...

(* Modes on values and signatures *)
let f (x @ local unique once) = ...
val g : t @ global -> t @ local

(* Unboxed values and mixed blocks *)
let x : float# = #3.14
let y : int32# = #42l
type sample = { tag : int; value : float# }
type point = { x : int; y : int }  (* also provides point# *)

(* Kinds *)
type ('a : float64) vector = ...
val id : ('a : value). 'a -> 'a

(* Comprehensions *)
[ x * 2 for x = 1 to 10 when x mod 2 = 0 ]
[| y for y in arr when y > 0 |]

(* Labeled tuples and immutable arrays *)
let point = ~x:1, ~y:2
let ~x, ~y = point
let xs : int iarray = [: 1; 2; 3 :]
let first = xs.:(0)

(* Unboxed tuple destructuring *)
let #(a, b) = some_unboxed_pair

(* Allocation checking *)
let[@zero_alloc] fast_add x y = x + y
```

## Working Rules

- Confirm the project is actually using OxCaml before introducing extension syntax.
- Prefer standard OCaml unless an OxCaml feature provides a concrete allocation, layout, parallelism, or API-safety benefit.
- Treat locality and uniqueness errors as design feedback, not as syntax issues to work around.
- Keep public interfaces explicit about modes and kinds when callers depend on them.
- Prefer an ordinary boxed record when callers need both boxed and unboxed forms;
  eligible boxed records automatically provide an implicit unboxed `t#` type.
- Add `[@zero_alloc]` only when the implementation and called functions can satisfy it; verify with the compiler rather than assuming.
- Benchmark and inspect allocation behaviour before and after performance-oriented changes.

## Common Pitfalls

- A `local` value must not escape its scope; use `exclave_` only for tail-position local returns.
- Global values can be used where local values are expected, but local values cannot be stored into global data.
- Unboxed types often require kind annotations in reusable abstractions.
- `unique` values lose uniqueness when aliased; design APIs so ownership is consumed clearly.
- SIMD code is architecture-sensitive; isolate feature-specific implementations behind a small module boundary.
