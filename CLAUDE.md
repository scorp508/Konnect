# Claude Code Guidance for Konnect

This file provides repository-specific guidance for Claude Code and other
coding agents working on Konnect.

## Fork-Only Policy

Keep this file local or in the `scorp508/Konnect` fork. Never include it in a
push or pull request to the upstream `mixelpixx/Konnect` repository.

## Read This First

Before changing code, read [CONTRIBUTING.md](CONTRIBUTING.md). It is the
authoritative contribution guide for this repository and takes precedence over
this summary.

Also consult the documents relevant to the task:

- [GOVERNANCE.md](GOVERNANCE.md) for ownership, work claiming, and merge rules.
- [ROADMAP.md](ROADMAP.md) before proposing non-trivial features.
- [docs/NAMING_CONVENTIONS.md](docs/NAMING_CONVENTIONS.md) before changing any
  public name.
- [docs/DEVELOPER_OVERVIEW.md](docs/DEVELOPER_OVERVIEW.md) for the codebase map.
- [DEV.md](DEV.md) for detailed architecture and development instructions.
- [tool-directory.md](tool-directory.md) for the generated MCP tool catalog.

Do not treat this file as a replacement for those documents. If guidance
conflicts, follow `CONTRIBUTING.md` and the more specific repository document.

## Project Summary

Konnect is a Rust MCP server and native KiCad 10 plugin for AI-assisted
schematic and PCB design.

Its main boundaries are:

- `crates/konnect`: process startup, configuration, installation, CLI commands,
  and stdio/HTTP transports.
- `crates/konnect-core`: MCP dispatch, tool routing, tool schemas, domain
  policy, and user-visible tool responses.
- `crates/konnect-sexp`: low-level KiCad S-expression parsing, geometry,
  revision-aware writes, and transaction journals.
- `crates/konnect-schematic-editor`: typed schematic parsing and mutation.
- `crates/konnect-ipc`: KiCad 10 IPC transport, protobuf types, and board
  operations.
- `crates/konnect-render`: deterministic visual rendering support.
- `crates/konnect-vcs`: project version-control functionality.
- `crates/schematic-viewer`: separately built Tauri schematic viewer.
- `plugin` and `packaging`: the thin KiCad integration layer and PCM packages.

Put changes in the crate that owns the behavior. Keep transport and CLI
concerns out of domain handlers, file-format primitives out of MCP handlers,
and user-facing policy out of the low-level IPC crate.

## Engineering Rules

- Keep each change focused on one reviewable outcome. Do not mix unrelated
  cleanup with a feature or fix.
- Investigate existing helpers and patterns before adding new ones.
- Treat MCP tool names, schema fields, CLI flags, environment variables,
  configuration keys, and documented paths as public API.
- Preserve compatibility. If a public change is necessary, document the
  migration explicitly.
- Derive responses from observed or committed results, not from requested
  values. If evidence is unavailable, report an incomplete or diagnostic
  result rather than claiming success.
- Fail closed for ambiguous paths, unsupported geometry, stale revisions,
  unavailable verification evidence, and KiCad requests that were received
  but rejected.
- Do not silently fall back from an IPC rejection to file editing. File
  fallback is permitted only where the established code explicitly allows it
  after determining that IPC is unreachable.
- Preserve revision checks, cooperative locking, atomic replacement, and
  transaction journals around KiCad file writes.
- Use typed argument helpers such as `require_str`, `require_f64`,
  `require_array`, `require_u64`, and `get_path`. Do not use a default to hide
  an omitted or incorrectly typed required argument.
- Use `try_layer_from_name` for KiCad writes. Never allow an unknown layer name
  to become `BL_UNDEFINED`.
- Base parser and geometry fixtures on files produced by real KiCad whenever
  possible. Synthetic fixtures alone are not sufficient evidence.
- Keep comments and documentation clear for readers who may not speak English
  as their first language. Explain non-obvious constraints and design reasons,
  not self-evident syntax.

## Changing or Adding an MCP Tool

Before adding a tool, search for an existing tool or shared helper that already
provides the required behavior.

When a tool is added, removed, or renamed:

1. Update the owning module under `crates/konnect-core/src/tools`.
2. Update `crates/konnect-core/src/router/registry.rs`.
3. Update the tool row in [tool-directory.md](tool-directory.md).
4. Add compatibility handling and update
   [docs/API_MIGRATIONS.md](docs/API_MIGRATIONS.md) when public behavior or
   naming changes.
5. Update bundled skills, agents, examples, and documentation that reference
   the tool.
6. Run `cargo xtask fix-doc-counts`; do not edit repeated catalog totals by
   hand.
7. Add tests for required arguments, incorrect types, success responses,
   refusal paths, and backend read-back where applicable.

Tool responses must say what the backend actually observed. A successful write
that cannot be trusted from its immediate return value should use a bounded
read-back or independent verification.

## Validation

Use the smallest targeted test while developing, then run the applicable
repository gates before considering the work complete:

```text
cargo test --workspace --locked --lib --tests
cargo test --workspace --locked --doc
cargo clippy --workspace --locked --all-targets -- -D warnings
cargo fmt --all -- --check
```

`protoc` and its well-known protobuf definitions are required to build
`konnect-ipc`. The schematic viewer is outside the main Cargo workspace and
must be built or tested separately when it changes.

Do not claim a KiCad integration works based only on mocked or synthetic
tests when a real-KiCad check is necessary to prove the behavior.

## Pull Requests

Follow [CONTRIBUTING.md](CONTRIBUTING.md) for the complete requirements.
Use an imperative PR title such as:

```text
fix(schematic): preserve tab-indented wire blocks
```

The PR description should identify:

1. the user-visible problem and scope;
2. the root cause and chosen design;
3. compatibility or migration effects;
4. tests run and intentionally skipped tests;
5. risks and rollback considerations.

Keep generated artifacts, downloaded catalogs, personal configuration, build
outputs, and unrelated formatting changes out of the diff.
