---
name: cmdliner
description: "Designing and implementing robust command-line interfaces using OCaml's cmdliner library. Use when Codex needs to: (1) Design a new CLI or subcommand layout, (2) Implement cmdliner terms and combinators, (3) Enforce clear, predictable, orthogonal options, (4) Produce high-quality --help output and error messages, (5) Integrate cmdliner CLIs into dune-based OCaml projects."
---

## Start Here

When designing or changing a cmdliner CLI:

- Keep command and option semantics clear, predictable, and orthogonal.
- Separate cmdliner parsing from business logic.
- Write concise `--help` text that documents defaults, accepted values, and
  useful examples.
- Provide small, pasteable examples that compile on recent OCaml and cmdliner
  versions.
- Use British spelling.

## Core Design Principles

1. **Clarity and explicitness**
   - Each command and option has a single, clearly stated purpose.
   - Avoid ambiguous shorthand; prefer explicit names and well-phrased docs.
   - Make defaults explicit in documentation and error messages.

2. **Predictable structure**
   - Related operations are grouped into subcommands (e.g. `mytool build`, `mytool check`, `mytool format`).
   - Options with similar names behave the same way across all commands.
   - Positional arguments appear in a stable, predictable order.

3. **Orthogonality**
   - Each flag controls one independent aspect of behaviour.
   - Avoid flags that silently alter multiple concerns.
   - Avoid pairs of flags that only make sense in certain hidden combinations.

4. **Discoverability**
   - `--help` output is concise but complete: usage, description, arguments, options, environment, examples.
   - Default values and accepted ranges or enumerations are documented.
   - Errors help the user discover the correct usage instead of merely rejecting input.

5. **Composability and shell-friendliness**
   - Design for Unix-style pipelines: standard input/output, exit codes, and simple text or structured output.
   - Avoid implicit file I/O if explicit paths or `-o` flags are possible.
   - Offer machine-friendly output formats where relevant (e.g. JSON) and document them.

6. **Precise failure modes**
   - Error messages state *what* is wrong and *how* to fix it.
   - Ambiguous or partial input is rejected with clear guidance.
   - Exit codes are chosen deliberately (e.g. `0` success, `1` user error, `2` internal failure).

7. **Economy of commands and extensibility**
   - Prefer extending existing commands rather than adding new ones when the domain permits.
   - Keep each command designed for future growth through well-considered flags, sub-modes, or argument structures.
   - Add new commands only for genuinely distinct operational domains.

## Good and Bad Examples

### Option Naming

**Bad**: Ambiguous or inconsistent names
```ocaml
(* Unclear what -f does without reading docs *)
let file = Arg.(value & opt (some string) None & info ["f"])

(* Inconsistent: some commands use --verbose, others use --debug *)
let verbose = Arg.(value & flag & info ["v"; "verbose"])
let debug = Arg.(value & flag & info ["d"; "debug"])  (* same thing? *)
```

**Good**: Clear, explicit names with consistent patterns
```ocaml
(* Self-documenting option name *)
let config_file =
  Arg.(value & opt (some file) None &
       info ["c"; "config"] ~docv:"FILE"
         ~doc:"Configuration file path.")

(* Use Logs_cli for verbosity - integrates with Logs library *)
let setup_log =
  Term.(const Logs_fmt.setup $ Fmt_cli.style_renderer () $ Logs_cli.level ())
(* Provides -v, -v -v, --verbosity=debug, etc. *)
```

### Subcommand Design

**Bad**: Flat command namespace with overlapping concerns
```ocaml
(* Explosion of top-level commands *)
let cmds = [
  create_user_cmd; delete_user_cmd; list_users_cmd;
  create_group_cmd; delete_group_cmd; list_groups_cmd;
  create_role_cmd; delete_role_cmd; list_roles_cmd;
]
```

**Good**: Hierarchical grouping with consistent verbs
```ocaml
(* Grouped by resource, consistent verbs *)
let create_cmd = Cmd.v (Cmd.info "create") create_user_term
let delete_cmd = Cmd.v (Cmd.info "delete") delete_user_term
let list_cmd = Cmd.v (Cmd.info "list") list_users_term

let user_cmd =
  let info = Cmd.info "user" ~doc:"Manage users" in
  Cmd.group info ~default:list_users_term [create_cmd; delete_cmd; list_cmd]

let main_cmd =
  let info = Cmd.info "mytool" ~version:"1.0" in
  Cmd.group info [user_cmd; group_cmd; role_cmd]
```

### Error Messages

**Bad**: Unhelpful error that doesn't guide the user
```ocaml
let validate_port p =
  if p < 0 || p > 65535 then `Error (false, "invalid port")
  else `Ok p
```

**Good**: Error explains what's wrong and how to fix it
```ocaml
let validate_port p =
  if p < 0 || p > 65535 then
    `Error (false, Printf.sprintf
      "port %d is out of range (must be 0-65535)" p)
  else `Ok p
```

### Separating Parsing from Logic

**Bad**: Business logic mixed with cmdliner parsing
```ocaml
let run_term =
  let open Term in
  const (fun config_file ->
    (* Business logic embedded in term *)
    let config = read_config config_file in
    let db = connect_db config in
    run_server db)
  $ config_file_arg
```

**Good**: Terms only parse; separate function does the work
```ocaml
(* Pure business logic function *)
let run ~config_file =
  let config = read_config config_file in
  let db = connect_db config in
  run_server db

(* Term just wires up arguments *)
let run_term = Term.(const run $ config_file_arg)
```

### Flag Orthogonality

**Bad**: Flags with hidden interactions
```ocaml
(* --json silently disables --color, user doesn't know *)
let output_format json color =
  if json then Json else if color then Colored else Plain
```

**Good**: Orthogonal flags, explicit conflicts
```ocaml
(* Either format flag, not both *)
let output_format =
  Arg.(value & vflag Plain [
    Json, info ["json"] ~doc:"Output as JSON.";
    Colored, info ["color"] ~doc:"Output with ANSI colors.";
  ])
```

## Cmdliner-Specific Guidance

When writing or revising cmdliner code, follow these patterns:

- Use `Cmd.v` with a `Term.t` and `Cmd.info` for each command or subcommand.
- Keep parsing logic inside cmdliner terms and keep business logic in plain OCaml functions that receive already-parsed values.
- Use `Arg.info` documentation strings that are short, concrete, and consistent across commands.
- Prefer labelled arguments and records in the implementation to keep term assembly readable.
- Ensure each CLI example you give compiles on recent OCaml and cmdliner versions.

### Typical Structure

When the user asks for a new CLI, aim to provide:

1. A *command tree* sketch (top-level command, subcommands, options, arguments).
2. Example `Cmd.t` and `Term.t` definitions.
3. Example `dune` stanzas required to build the executable.
4. Example usage snippets showing common workflows.

## Response Format

Unless the user requests otherwise, structure your responses as:

1. **Overview** – brief description of the CLI design or change.
2. **Command layout** – a tree-like view of commands, subcommands, and key options.
3. **Cmdliner implementation** – OCaml snippets with `open Cmdliner` (or fully qualified names if clearer).
4. **Help and examples** – sample `--help` output and real-world usage examples.
5. **Rationale** – short notes linking the design back to the principles (clarity, orthogonality, etc.).

Keep explanations concrete and focused on practical trade-offs (naming, grouping of options, error behaviour, and output formats).

## CLI Output Design

For terminal styling, output modes, progress bars, tables, logging setup, and
complete output examples, load `references/output-design.md`.

Use these rules in the main cmdliner design:

- Offer a machine-readable mode such as `--output=json` when output may be
  consumed by scripts.
- Keep human output readable, but disable colours and progress bars when stdout
  is not a TTY.
- Use `Fmt` for formatted output, `Logs_cli` for verbosity, and the **ocaml-progress**
  skill for long-running operations.
- Make errors actionable: state the invalid input, the affected argument, and a
  concrete hint.
