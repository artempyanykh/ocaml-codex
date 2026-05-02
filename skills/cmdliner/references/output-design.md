# CLI Output Design

A good CLI is both useful and readable. Use this reference when a cmdliner task
also needs human-friendly output, machine-readable output, progress reporting, or
logging setup.

## Core Libraries

| Library | Purpose |
|---------|---------|
| `fmt` | Styled terminal output |
| `progress` | Progress bars and spinners |
| `logs` + `logs-cli` | Structured logging with verbosity levels |
| `notty` | Full terminal UI for complex tools |

## Output Modes

Every CLI should support at least two output modes:

```ocaml
type output_format = Human | Json

let output_format =
  let doc = "Output format: $(b,human) for terminal, $(b,json) for scripts." in
  Arg.(value & opt (enum ["human", Human; "json", Json]) Human &
       info ["o"; "output"] ~doc ~docv:"FORMAT")
```

Human mode can use colours, progress bars, tables, and status indicators. JSON
mode must be machine-parseable, avoid ANSI codes, and use newline-delimited
records for streaming output.

## Colour Scheme

Use consistent semantic colours across all tools:

```ocaml
let success = Fmt.(styled `Green string)
let error = Fmt.(styled `Red string)
let warning = Fmt.(styled `Yellow string)
let info = Fmt.(styled `Cyan string)
let dimmed = Fmt.(styled `Faint string)
let bold = Fmt.(styled `Bold string)
let code = Fmt.(styled `Cyan string)

let pp_status ppf = function
  | `Ok -> Fmt.pf ppf "%a" Fmt.(styled `Green string) "OK"
  | `Error -> Fmt.pf ppf "%a" Fmt.(styled `Red string) "ERROR"
  | `Warning -> Fmt.pf ppf "%a" Fmt.(styled `Yellow string) "WARN"
  | `Info -> Fmt.pf ppf "%a" Fmt.(styled `Cyan string) "INFO"
  | `Pending -> Fmt.pf ppf "%a" Fmt.(styled `Blue string) "PENDING"
```

## Progress Bars

Use the `progress` library for long-running operations. For reporter semantics,
multi-line displays, byte counters, and Eio integration, load the **progress**
skill.

```ocaml
open Progress

let with_progress ~total f =
  let bar =
    Line.(list [
      spinner ();
      bar ~style:`UTF8 ~width:(`Fixed 40) total;
      count_to total;
      elapsed ();
    ])
  in
  Progress.with_reporter bar f

let process_files files =
  let total = List.length files in
  with_progress ~total (fun report ->
    List.iter (fun file ->
      process_file file;
      report 1
    ) files)
```

For indeterminate operations, use spinners:

```ocaml
let with_spinner ~message f =
  let line = Line.(list [spinner (); const message]) in
  Progress.with_reporter line (fun _report -> f ())
```

## Tables

For tabular data, use aligned columns:

```ocaml
let pp_table ppf rows =
  let widths = compute_column_widths rows in
  List.iter (fun row ->
    List.iteri (fun i cell ->
      let width = List.nth widths i in
      Fmt.pf ppf "%-*s  " width cell
    ) row;
    Fmt.pf ppf "@."
  ) rows

let pp_table_with_header ppf ~headers rows =
  List.iter (fun h -> Fmt.pf ppf "%a  " bold h) headers;
  Fmt.pf ppf "@.";
  List.iter (fun h -> Fmt.pf ppf "%s  " (String.make (String.length h) '-')) headers;
  Fmt.pf ppf "@.";
  List.iter (fun row ->
    List.iter (fun cell -> Fmt.pf ppf "%s  " cell) row;
    Fmt.pf ppf "@."
  ) rows
```

## Error Output

Errors should be clear, actionable, and visually distinct:

```ocaml
let pp_error ppf ~context ~message ~hint =
  Fmt.pf ppf "@[<v>%a %a@,%a@,%a %a@]@."
    Fmt.(styled `Red string) "error:"
    Fmt.(styled `Bold string) message
    dimmed (Printf.sprintf "  in %s" context)
    Fmt.(styled `Cyan string) "hint:"
    Fmt.string hint
```

Example output:

```text
error: Invalid port number '70000'
  in --port argument
hint: Port must be between 0 and 65535
```

## Summary Output

For commands that process multiple items:

```ocaml
let pp_summary ppf ~processed ~succeeded ~failed ~skipped =
  Fmt.pf ppf "@.%a@."
    Fmt.(styled `Bold string) "Summary:";
  Fmt.pf ppf "  - %d processed@." processed;
  if succeeded > 0 then
    Fmt.pf ppf "  OK %d succeeded@." succeeded;
  if failed > 0 then
    Fmt.pf ppf "  ERROR %d failed@." failed;
  if skipped > 0 then
    Fmt.pf ppf "  WARN %d skipped@." skipped
```

## TTY Detection

Always check if stdout is a terminal before using colours or progress bars:

```ocaml
let setup_formatter () =
  let style_renderer =
    if Unix.isatty Unix.stdout then `Ansi_tty else `None
  in
  Fmt.set_style_renderer Fmt.stdout style_renderer

let setup_term =
  Term.(const Fmt_tty.setup_std_outputs $ Fmt_cli.style_renderer ())
```

## Verbosity Levels

Integrate with `Logs` for consistent verbosity:

```ocaml
let setup_log style_renderer level =
  Fmt_tty.setup_std_outputs ?style_renderer ();
  Logs.set_level level;
  Logs.set_reporter (Logs_fmt.reporter ())

let setup_log_term =
  Term.(const setup_log $ Fmt_cli.style_renderer () $ Logs_cli.level ())

Logs.debug (fun m -> m "Processing file %s" path);
Logs.info (fun m -> m "Converted %d records" count);
Logs.warn (fun m -> m "Deprecated format, consider upgrading");
Logs.err (fun m -> m "Failed to parse: %s" reason);
```

## Complete Example

```ocaml
open Cmdliner

let success fmt = Fmt.pf Fmt.stdout ("OK " ^^ fmt ^^ "@.")
let error fmt = Fmt.pf Fmt.stderr ("ERROR " ^^ fmt ^^ "@.")
let info fmt = Fmt.pf Fmt.stdout ("INFO " ^^ fmt ^^ "@.")

let convert ~input ~output ~format =
  info "Converting %s to %s format" input format;
  match do_convert input output format with
  | Ok bytes ->
      success "Wrote %d bytes to %s" bytes output;
      `Ok ()
  | Error msg ->
      error "Conversion failed: %s" msg;
      `Error (false, msg)

let term =
  let open Term in
  const convert $ input_arg $ output_arg $ format_arg

let cmd =
  let info =
    Cmd.info "convert"
      ~doc:"Convert between formats"
      ~man:[`S "EXAMPLES"; `P "$(iname) input.json -o output.cbor"]
  in
  Cmd.v info Term.(ret (const setup $ setup_log_term $ term))
```

## Checklist

- [ ] Supports `--output=json` for machine-readable output
- [ ] Uses semantic colours consistently
- [ ] Uses progress bars for operations longer than about one second
- [ ] Gives clear error messages with hints
- [ ] Prints summary output for batch operations
- [ ] Detects TTYs before enabling colours or progress
- [ ] Uses `Logs_cli` for `-v` / `--verbosity`
- [ ] Matches the project conventions
