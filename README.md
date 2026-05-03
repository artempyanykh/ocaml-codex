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

The plugin id is `ocaml-codex`. The human-readable name shown in Codex is
`OCaml Development`.

### Install from the Marketplace File

This repository includes a Codex marketplace file at
`.agents/plugins/marketplace.json`. The marketplace exposes this plugin through
`./plugins/ocaml-codex`, which is a symlink back to the repository root.

1. In Codex, open `/plugin`.

2. Choose **Add marketplace**.

3. Add this repository as the marketplace source:

   ```text
   https://github.com/artempyanykh/ocaml-codex
   ```

4. Select the `OCaml Codex` marketplace.

5. Install `OCaml Development`.

### Local Development Install

Use this flow when you want Codex to load your local checkout through a personal
marketplace instead of adding this repository as a marketplace source.

The important Codex-specific pieces are:

- `~/.agents/plugins/marketplace.json` declares your personal local marketplace.
- `~/.codex/plugins/ocaml-codex` points at this plugin checkout.
- `source.path` in the marketplace is relative to your home directory.

1. Create the local plugin directory and symlink this checkout into it. Replace
   `<repo>` with the path to this repository.

   ```bash
   mkdir -p ~/.codex/plugins
   ln -sfn <repo> ~/.codex/plugins/ocaml-codex
   ```

2. Create the personal marketplace directory.

   ```bash
   mkdir -p ~/.agents/plugins
   ```

3. Create `~/.agents/plugins/marketplace.json` with this exact content.

   ```json
   {
     "name": "local",
     "interface": {
       "displayName": "Local Plugins"
     },
     "plugins": [
       {
         "name": "ocaml-codex",
         "source": {
           "source": "local",
           "path": "./.codex/plugins/ocaml-codex"
         },
         "policy": {
           "installation": "AVAILABLE",
           "authentication": "ON_INSTALL"
         },
         "category": "Productivity"
       }
     ]
   }
   ```

   The `source.path` value is relative to your home directory, so
   `./.codex/plugins/ocaml-codex` resolves to
   `~/.codex/plugins/ocaml-codex`.

4. Restart Codex.

5. Open `/plugin`, select the `Local Plugins` marketplace, and install
   `OCaml Development`.

### Troubleshooting

- If the plugin appears in `/plugin` but installation fails with
  `Plugin detail unavailable` or `plugin/install failed in TUI`, check:

  ```bash
  tail -n 80 ~/.codex/log/codex-tui.log
  ```

- If Codex cannot find the plugin details, verify the symlink:

  ```bash
  ls -la ~/.codex/plugins/ocaml-codex
  test -f ~/.codex/plugins/ocaml-codex/.codex-plugin/plugin.json
  ```

## Attribution

This Codex plugin was adapted from Anil Madhavapeddy's excellent Claude Code
plugin for OCaml:

https://github.com/avsm/ocaml-claude-marketplace

Current Codex plugin packaging and authorship are maintained at:

https://github.com/artempyanykh/ocaml-codex

## License

ISC. See `LICENSE`.
