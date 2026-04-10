# Build System — flake.nix

# LibreFang Build System — `flake.nix`

## Overview

This module defines the Nix flake for LibreFang, providing reproducible builds, automated checks, and a standardized development environment across all supported platforms. The flake uses [crane](https://github.com/ipetkov/crane) to integrate Rust tooling with Nix, ensuring efficient incremental builds and cache-friendly workflows.

## Input Dependencies

The flake declares four primary inputs that form the foundation of the build system:

| Input | Source | Purpose |
|-------|--------|---------|
| `nixpkgs` | `github:NixOS/nixpkgs/nixpkgs-unstable` | Base package set and Nix utilities |
| `crane` | `github:ipetkov/crane` | Rust-specific build helpers and caching |
| `flake-utils` | `github:numtide/flake-utils` | Cross-platform compatibility helpers |
| `rust-overlay` | `github:oxalica/rust-overlay` | Rust toolchain management with latest stable |

The `rust-overlay` input follows `nixpkgs` to maintain version consistency between the two.

## Build Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        flake.nix                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐     ┌──────────────────────────────────────┐ │
│  │   nixpkgs    │────▶│         pkgs (with rust overlay)     │ │
│  │  (unstable)  │     └──────────────────────────────────────┘ │
│  └──────────────┘                      │                      │
│         │                              ▼                      │
│         │              ┌───────────────────────────────┐      │
│         │              │      rustToolchain           │      │
│         │              │   (stable.latest.default)    │      │
│         │              │   + rust-src, rust-analyzer, │      │
│         │              │     clippy extensions        │      │
│         │              └───────────────────────────────┘      │
│         │                              │                      │
│         ▼                              ▼                      │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                     craneLib                             │ │
│  │  (mkLib pkgs with custom toolchain)                      │ │
│  └──────────────────────────────────────────────────────────┘ │
│                              │                                 │
│         ┌────────────────────┼────────────────────┐           │
│         ▼                    ▼                    ▼           │
│  ┌─────────────┐    ┌────────────────┐    ┌──────────────┐   │
│  │ cleanCargo  │    │ buildDepsOnly  │    │ buildPackage │   │
│  │ Source src  │    │ (cargoArtifacts)    │ (librefang)  │   │
│  └─────────────┘    └────────────────┘    └──────────────┘   │
│         │                   │                    │           │
│         └───────────────────┴────────────────────┘           │
│                             ▼                                 │
│              ┌───────────────────────────────┐              │
│              │         checks                │              │
│              │  (clippy, fmt, librefang)     │              │
│              └───────────────────────────────┘              │
│                             │                                 │
│         ┌────────────────────┼────────────────────┐           │
│         ▼                    ▼                    ▼           │
│  ┌─────────────┐    ┌────────────────┐    ┌──────────────┐   │
│  │  packages   │    │    apps        │    │  devShells   │   │
│  │ default     │    │  default       │    │   default    │   │
│  └─────────────┘    └────────────────┘    └──────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Key Components

### Rust Toolchain Configuration

```nix
rustToolchain = pkgs.rust-bin.stable.latest.default.override {
  extensions = [ "rust-src" "rust-analyzer" "clippy" ];
};
```

The toolchain is built from `rust-bin` (instead of nixpkgs' `rustc`) to ensure:
- **Latest stable Rust** via `stable.latest`
- **`rust-src`** — Required for procedural macros and build scripts
- **`rust-analyzer`** — Language server for IDE support in the dev shell
- **`clippy`** — Linter used by the CI checks

### Source Filtering

```nix
src = craneLib.cleanCargoSource ./.;
```

`cleanCargoSource` filters the repository to only include files relevant to Rust compilation, excluding:
- `.git` directory
- `target/` build artifacts
- Documentation files not needed for compilation
- Other non-Rust source files

This optimization improves build caching and reduces the evaluation surface.

### Dependency Caching Strategy

```nix
cargoArtifacts = craneLib.buildDepsOnly commonArgs;

librefang = craneLib.buildPackage (commonArgs // {
  inherit cargoArtifacts;
  # ...
});
```

The build system uses a two-phase approach:
1. **`buildDepsOnly`** — Compiles `Cargo.lock` dependencies first, producing `cargoArtifacts`
2. **`buildPackage`** — Builds the actual project using cached dependencies

This pattern enables granular caching: dependency compilation can be reused across rebuilds when only application code changes.

### Native Dependencies

```nix
nativeBuildInputs = with pkgs; [
  pkg-config
  rustToolchain
];

buildInputs = with pkgs; [
  openssl
] ++ pkgs.lib.optionals pkgs.stdenv.isDarwin [
  pkgs.apple-sdk
  pkgs.libiconv
];
```

- **`pkg-config`** — Locates library headers during build
- **`openssl`** — Required by the Rust `openssl` crate
- **Darwin-specific** — `apple-sdk` provides system headers, `libiconv` handles text encoding

### Strict Dependency Mode

```nix
commonArgs = {
  # ...
  strictDeps = true;
};
```

`strictDeps = true` ensures all dependencies must be explicitly declared in `nativeBuildInputs` or `buildInputs`. This catches missing dependencies early and improves reproducibility.

## Output Types

### Checks

The `checks` output provides reproducible validation of code quality:

| Check | Command | Purpose |
|-------|---------|---------|
| `librefang` | `cargo build` | Verifies the project compiles |
| `librefang-clippy` | `cargo clippy --workspace` | Runs linter with `-D warnings` |
| `librefang-fmt` | `cargo fmt --check` | Validates code formatting |

These checks run automatically in CI via `nix flake check`.

### Packages

```nix
packages = {
  default = librefang;
  librefang-cli = librefang;
};
```

The CLI binary is the default package. Users can build it with:

```bash
nix build                      # default package
nix build .#librefang-cli      # explicit reference
```

### Apps

```nix
apps.default = flake-utils.lib.mkApp {
  drv = librefang;
};
```

Enables running the application directly:

```bash
nix run .                      # run the CLI
```

### Development Shell

```nix
devShells.default = craneLib.devShell {
  checks = self.checks.${system};
  packages = with pkgs; [ ... ];
  inputsFrom = [ librefang ];
  shellHook = '' ... '';
};
```

The dev shell provides:

| Component | Description |
|-----------|-------------|
| `checks` | Runs clippy/fmt checks when entering the shell |
| `inputsFrom` | Inherits build dependencies from `librefang` |
| `cargo-watch` | Auto-rebuild on file changes |
| `cargo-edit` | CLI for adding/removing dependencies |
| `cargo-expand` | Macro debugging |
| `just` | Command runner (alternative to Makefile) |
| `gh`, `git`, `nodejs`, `python3` | Standard development tools |

Enter the development environment with:

```bash
nix develop
```

## How to Use

### Building

```bash
# Build the CLI binary
nix build

# Find the result
./result/bin/librefang --version
```

### Development Workflow

```bash
# Enter the dev environment
nix develop

# Run clippy with auto-rebuild
cargo watch -x clippy

# Format code
cargo fmt

# Check formatting
cargo fmt -- --check

# Add a dependency
cargo add tokio
```

### CI/Testing

```bash
# Run all checks locally (same as CI)
nix flake check

# Check a specific system
nix flake check --system x86_64-linux
```

### Reproducing CI Locally

The dev shell's `checks = self.checks.${system}` ensures that entering the shell runs clippy and format checks, matching the CI pipeline exactly.

## Build Targets

The flake targets the workspace package `librefang-cli`:

```nix
cargoExtraArgs = "--package librefang-cli";
```

This corresponds to a Cargo workspace member defined in `Cargo.toml`. To build different workspace crates, modify `cargoExtraArgs` or reference other packages by their workspace names.