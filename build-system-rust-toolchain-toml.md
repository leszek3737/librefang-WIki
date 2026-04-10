# Build System — rust-toolchain.toml

# rust-toolchain.toml

## Overview

This file configures the Rust toolchain for the project. It ensures consistent Rust version and tooling across all developers and CI environments by pinning specific settings that `rustup` reads automatically when operating in the project directory.

## Configuration

### Channel

```toml
channel = "stable"
```

The project targets the **stable** Rust channel, meaning it uses the latest officially released Rust compiler. This ensures:

- The project builds with a well-tested, production-ready compiler
- No unstable features are required
- Maximum compatibility with the Rust ecosystem

### Components

```toml
components = ["rustfmt", "clippy"]
```

Two additional components are installed alongside the base toolchain:

| Component | Purpose |
|-----------|---------|
| `rustfmt` | Code formatter; ensures consistent code style across the codebase |
| `clippy` | Linter; provides additional warnings and suggestions beyond the compiler |

## Toolchain Resolution

When `cargo`, `rustc`, or other Rust tools run from this directory or any subdirectory, `rustup` automatically reads this file and ensures the specified toolchain is active.

The resolution order is:

1. `rust-toolchain.toml` or `rust-toolchain` file in current directory
2. `rust-toolchain.toml` or `rust-toolchain` in parent directories (walked upward)
3. `RUSTUP_TOOLCHAIN` environment variable override
4. Default toolchain set via `rustup default`

## Developer Workflow

### Running Formatter Checks

```bash
cargo fmt -- --check
```

### Running the Linter

```bash
cargo clippy
```

### Updating the Toolchain

To switch to a different Rust version locally (for testing against nightly, beta, or a specific version):

```bash
rustup override set nightly
rustup override set 1.70.0
rustup override unset  # Remove override, return to toolchain.toml
```

This override takes precedence over `rust-toolchain.toml` until unset.

### Verifying Current Toolchain

```bash
rustup show
```

Shows the currently active toolchain and its source (toolchain file vs. override vs. default).