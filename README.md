# OCaml Codex Plugin

Codex skills for OCaml and OxCaml development.

This plugin packages guidance for common OCaml engineering work: project setup,
Dune and opam conventions, module and interface design, Eio concurrency,
Cmdliner CLIs, testing, fuzzing, documentation, security review, Memtrace
profiling, JSON codecs with Jsont, npm publishing workflows, and OxCaml
performance extensions.

## Contents

- `ocaml` - general OCaml development guidance.
- `ocaml-code-style` - style, refactoring, naming, and API design patterns.
- `ocaml-project-setup` - project metadata, templates, CI, and release-oriented files.
- `ocaml-testing` - Alcotest, Cram, Eio mock tests, and test organization.
- `ocaml-fuzz` - Crowbar fuzzing patterns and AFL campaign setup.
- `ocaml-security` - parser/protocol security audit workflow and defensive tests.
- `ocaml-result` - stdlib `Result` patterns and version-aware syntax guidance.
- `ocaml-logs` - `Logs` setup, sources, levels, and tags.
- `ocaml-effects` - OCaml 5 effect design patterns.
- `ocaml-progress` - terminal progress bars and Eio integration.
- `ocaml-docs` - odoc warning fixes and reference syntax.
- `oxcaml` - OxCaml modes, stack allocation, unboxed types, SIMD, and zero-allocation checks.
- `eio` - Eio concurrency, capabilities, resources, cancellation, and mocks.
- `cmdliner` - command-line interface design and Cmdliner implementation patterns.
- `jsont` - typed JSON encoding and decoding with Jsont.
- `memtrace` - allocation tracing and hotspot analysis.
- `npm-publishing` - js_of_ocaml and wasm_of_ocaml npm publishing workflows.
- `rfc-integration` - RFC fetching, citation, and compliance review.
- `tutorials` - `.mld` and MDX tutorial authoring.

## Installation

This repository is a Codex plugin root. The plugin manifest lives at
`.codex-plugin/plugin.json` and points Codex at `./skills/`.

For local development, place or clone this repository where Codex can load local
plugins, then install or enable the `ocaml-dev` plugin from that location.

## Attribution

This Codex plugin was adapted from Anil Madhavapeddy's excellent Claude Code
plugin for OCaml:

https://github.com/avsm/ocaml-claude-marketplace

Current Codex plugin packaging and authorship are maintained at:

https://github.com/artempyanykh/ocaml-codex

## License

ISC. See `LICENSE`.
