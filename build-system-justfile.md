# Build System — justfile

# Build System — justfile

## Overview

The `justfile` is the project's task runner, powered by [just](https://github.com/casey/just). It serves as the single entry point for all build, test, lint, release, and maintenance commands across the LibreFang workspace. Most non-trivial operations delegate to `cargo xtask` commands implemented in the `xtask` crate, keeping this file as a thin routing layer.

Running `just` with no arguments prints the full recipe list via `just --list`.

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| `just` | Recipe runner | `cargo install just` |
| `cargo` | Rust builds, xtask runner | rustup |
| `pnpm` | Dashboard frontend builds | `npm install -g pnpm` |
| `tauri-cli` | Desktop app builds (optional) | `cargo install tauri-cli` |

## Core Development Recipes

These cover the standard edit–build–test cycle.

| Recipe | Command | Description |
|--------|---------|-------------|
| `build` | `cargo build --workspace --lib` | Compile all workspace libraries |
| `test` | `cargo test --workspace` | Run all workspace tests |
| `check` | `cargo check --workspace` | Type-check without producing artifacts |
| `lint` | `cargo clippy --workspace --all-targets -- -D warnings` | Clippy with warnings as errors |
| `fmt` | `cargo fmt --all` | Format all Rust code in place |
| `fmt-check` | `cargo fmt --all -- --check` | Verify formatting without modifying files |
| `doc` | `cargo doc --workspace --no-deps --open` | Build and open rustdoc |

The `ci` recipe runs `cargo xtask ci`, which combines build, test, clippy, and web linting into a single pass for local CI simulation before pushing.

## Web & Desktop Builds

The project ships a React dashboard compiled to static assets, consumed by both the API server and the Tauri desktop app.

```
dashboard-build
       │
       ▼
desktop-build ─────► cargo tauri build
desktop-dev   ─────► cargo tauri dev
install       ─────► ~/.librefang/bin
install-full  ─────► ~/.librefang/bin + ~/.librefang/dashboard
```

- **`build-web *ARGS`** — General-purpose frontend build, delegates to `cargo xtask build-web`.
- **`dashboard-build`** — Builds only the React dashboard assets (shorthand for `build-web --dashboard`).
- **`dash`** — Starts the React dev server with hot reload. Requires the API running on `:4545`. Changes directory to `crates/librefang-api/dashboard` and runs `pnpm install && pnpm dev`.
- **`desktop-build`** and **`desktop-dev`** — Build or run the Tauri desktop app. Both depend on `dashboard-build` to ensure frontend assets are current before the Tauri bundle is created.

## Installation Recipes

### Unix (`~/.librefang/bin`)

- **`install`** — Builds `librefang-cli` with the `release-local` profile and copies the binary to `~/.librefang/bin/librefang`.
- **`install-full`** — Extends `install` by also deploying a fresh dashboard to `~/.librefang/dashboard` and writing a `.version` file extracted from `cargo metadata`.

### Windows (`%USERPROFILE%\.librefang\bin`)

- **`install`** — Same as the Unix variant, using `cmd` syntax (`if not exist`, `copy /Y`).

Both platform variants are gated with `[unix]` and `[windows]` attributes. The `set windows-shell := ["cmd", "/c"]` directive at the top ensures correct shell invocation on Windows.

## Xtask-Delegated Recipes

Most operational complexity lives behind `cargo xtask <name>` calls. The justfile passes through any extra arguments via the `*ARGS` variadic parameter, so each recipe supports xtask-specific flags.

### Build & Release

| Recipe | Xtask | Purpose |
|--------|-------|---------|
| `dist *ARGS` | `xtask dist` | Cross-platform release binaries |
| `docker *ARGS` | `xtask docker` | Build and optionally push Docker images |
| `release *ARGS` | `xtask release` | Cut a new release |
| `publish-sdks *ARGS` | `xtask publish-sdks` | Publish SDKs to npm, PyPI, crates.io |
| `publish-npm-binaries *ARGS` | `xtask publish-npm-binaries` | Publish CLI binaries to npm |
| `publish-pypi-binaries *ARGS` | `xtask publish-pypi-binaries` | Publish CLI wheels to PyPI |

### Code Quality & Analysis

| Recipe | Xtask | Purpose |
|--------|-------|---------|
| `coverage *ARGS` | `xtask coverage` | Test coverage reports |
| `bench *ARGS` | `xtask bench` | Criterion benchmarks |
| `deps *ARGS` | `xtask deps` | Dependency vulnerability/updates audit |
| `license-check *ARGS` | `xtask license-check` | Dependency license validation |
| `check-links *ARGS` | `xtask check-links` | Broken link detection in docs |
| `loc *ARGS` | `xtask loc` | Lines of code and dependency graph stats |

### Code Generation & Documentation

| Recipe | Xtask | Purpose |
|--------|-------|---------|
| `codegen *ARGS` | `xtask codegen` | OpenAPI spec and other generated code |
| `api-docs *ARGS` | `xtask api-docs` | API docs from OpenAPI spec |
| `changelog *ARGS` | `xtask changelog` | Generate CHANGELOG from merged PRs |
| `contributors *ARGS` | `xtask contributors` | Contributor/star history SVGs |

### Environment & Maintenance

| Recipe | Xtask | Purpose |
|--------|-------|---------|
| `setup *ARGS` | `xtask setup` | First-time dev environment setup |
| `dev *ARGS` | `xtask dev` | Start daemon + dashboard hot reload |
| `doctor *ARGS` | `xtask doctor` | Diagnose environment issues |
| `db *ARGS` | `xtask db` | Database info, backup, reset |
| `clean-all *ARGS` | `xtask clean-all` | Remove all build artifacts (Rust + web) |
| `update-deps *ARGS` | `xtask update-deps` | Update Rust and web dependencies |
| `sync-versions *ARGS` | `xtask sync-versions` | Synchronize crate versions across workspace |
| `fmt-all *ARGS` | `xtask fmt` | Format both Rust and web code |
| `validate-config *ARGS` | `xtask validate-config` | Validate config.toml |
| `pre-commit *ARGS` | `xtask pre-commit` | fmt + clippy + test in one pass |
| `migrate *ARGS` | `xtask migrate` | Migrate agents from other frameworks |

### Testing

| Recipe | Xtask | Purpose |
|--------|-------|---------|
| `integration-test *ARGS` | `xtask integration-test` | Live integration test suite |

## Passing Arguments

All variadic recipes use `*ARGS` which forwards extra arguments to the underlying xtask. For example:

```bash
# Run integration tests against a specific host
just integration-test --host http://staging.example.com

# Build Docker image with a specific tag
just docker --tag v2.1.0 --push

# Run benchmarks for a specific crate
just bench --bench crate_name
```

## Recipe Dependency Graph

Recipes with a `:` suffix (e.g., `desktop-build: dashboard-build`) declare dependencies. The following recipes chain automatically:

```
desktop-build ──► dashboard-build
desktop-dev   ──► dashboard-build
install       ──► dashboard-build    (both unix and windows)
install-full  ──► dashboard-build
```

When you run `just desktop-build`, `just` first runs `dashboard-build` to compile the React assets, then proceeds with the Tauri build. No manual ordering is required.