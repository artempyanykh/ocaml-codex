---
name: project-setup
description: "Standards for OCaml project metadata files. Use when initializing a new OCaml library/module, preparing for opam release, setting up CI, discussing project structure, or ensuring proper .mli/.ocamlformat files exist."
---

# OCaml Project Setup

## Required Files

Every OCaml project needs:

| File | Purpose |
|------|---------|
| `dune-project` | Build configuration, opam generation |
| `dune` (root) | Top-level build rules |
| `.ocamlformat` | Code formatting (required) |
| `.gitignore` | VCS ignores |
| `LICENSE.md` | License file |
| `README.md` | Project documentation |
| CI config | GitHub Actions / GitLab CI / Tangled |

## Interface Files (.mli)

Create an `.mli` file for each public library module:
- Clear API boundaries
- Proper encapsulation
- Documentation surface

Keep module API design, documentation wording, naming, and refactoring guidance
in the **ocaml** and **code-style** skills. This skill only defines the expected
project files and metadata.

## OCamlFormat Configuration

**Required**: `.ocamlformat` in project root.

```
profile = default
version = 0.29.0
```

Run `dune fmt` before every commit.

## License Headers

Every source file starts with license header:

```ocaml
(*---------------------------------------------------------------------------
  Copyright (c) {{YEAR}} {{AUTHOR}}. All rights reserved.
  SPDX-License-Identifier: ISC
 ---------------------------------------------------------------------------*)
```

## Project Structure

```
project/
├── dune-project
├── dune
├── .ocamlformat
├── .gitignore
├── LICENSE.md
├── README.md
├── lib/
│   ├── dune
│   ├── foo.ml
│   └── foo.mli         # Required for every .ml
├── bin/
│   ├── dune
│   └── main.ml
├── test/
│   ├── dune
│   ├── test.ml
│   └── test_foo.ml
├── .github/workflows/  # GitHub Actions
├── .gitlab-ci.yml      # GitLab CI
└── .tangled/workflows/ # Tangled CI
```

## dune-project

```lisp
(lang dune 3.22)

(name project_name)

(license ISC)

(authors "Name <email@example.com>")

(maintainers "Name <email@example.com>")

(source (tangled user.domain/project_name))

(generate_opam_files true)

(package
 (name project_name)
 (synopsis "Short description")
 (description "Longer description")
 (depends
  (ocaml (>= 5.2))
  (alcotest (and :with-test (>= 1.7.0)))))
```

Use the lowest `(lang dune X.Y)` version that supports the project features you
need. New projects can start from the current stable Dune language version, but
existing projects should not bump this field without a reason.

**Source options**:
- `(source (tangled handle/repo))` - Tangled hosting (default for monopam)
- `(source (github user/repo))` - GitHub hosting
- `(source (gitlab user/repo))` - GitLab hosting

**Note**: Don't add `(version ...)` - added at release time.

### Tangled Source Syntax

For projects hosted on tangled.org, use the succinct source stanza:

```lisp
(source (tangled user.domain/project-name))
```

Examples:
- `(source (tangled anil.recoil.org/ocaml-brotli))`
- `(source (tangled user.example.org/my-library))`

## Tangled CI Configuration

For projects hosted on tangled.org, create `.tangled/workflows/build.yml`:

```yaml
when:
  - event: ["push", "pull_request"]
    branch: ["main"]

engine: nixery

dependencies:
  nixpkgs:
    - shell
    - stdenv
    - findutils
    - binutils
    - libunwind
    - ncurses
    - opam
    - git
    - gawk
    - gnupatch
    - gnum4
    - gnumake
    - gnutar
    - gnused
    - gnugrep
    - diffutils
    - gzip
    - bzip2
    - gcc
    - ocaml
    - pkg-config

steps:
  - name: opam
    command: |
      opam init --disable-sandboxing -a -y

  - name: repo
    command: |
      opam repo add aoah https://tangled.org/anil.recoil.org/aoah-opam-repo.git

  - name: deps
    command: |
      opam install . --confirm-level=unsafe-yes --deps-only

  - name: build
    command: |
      opam exec -- dune build

  - name: test
    command: |
      opam install . --confirm-level=unsafe-yes --deps-only --with-test
      opam exec -- dune runtest --verbose
```

### Tangled Workflow Syntax

| Field | Description |
|-------|-------------|
| `when` | Trigger conditions: `event` (push/pull_request) and `branch` |
| `engine` | Build engine, use `nixery` for Nix-based builds |
| `dependencies.nixpkgs` | List of Nix packages to include |
| `environment` | Global or per-step environment variables |
| `steps` | Build steps with `name` and `command` |

Per-step environment variables:

```yaml
steps:
  - name: test
    environment:
      MY_VAR: value
    command: |
      echo $MY_VAR
```

## Templates

See `templates/` directory for:
- `dune-project.template`
- `dune-root.template`
- `ci-github.yml`
- `ci-gitlab.yml`
- `ci-tangled.yml`
- `gitignore`
- `ocamlformat`
- `LICENSE-ISC.md`
- `LICENSE-MIT.md`
- `README.template.md`
