# Build System — rust-toolchain.toml

# Build System — `rust-toolchain.toml`

## Overview

This file pins the Rust toolchain for the entire project. It is read automatically by **rustup** whenever any Rust command (`cargo build`, `cargo test`, `rustc`, etc.) is invoked inside this repository, ensuring that every developer and CI runner uses the same compiler version and component set.

## Configuration

```toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy"]
```

| Field | Value | Purpose |
|---|---|---|
| `channel` | `"stable"` | Selects the latest stable release of the Rust compiler. This channel is updated every six weeks when a new Rust edition ships. |
| `components` | `["rustfmt", "clippy"]` | Installs `rustfmt` (code formatting) and `clippy` (linting) alongside the compiler so they are guaranteed to be available. |

## How It Works

When a Rust command runs in this directory, rustup checks for `rust-toolchain.toml` (or the legacy `rust-toolchain` file). If found, it:

1. Resolves the specified `channel` to an exact compiler version (e.g., `stable-x86_64-unknown-linux-gnu`).
2. Ensures that toolchain is installed, downloading it if necessary.
3. Ensures every listed `component` is installed for that toolchain.
4. Activates the toolchain for the current command.

This happens transparently — no manual `rustup default` or `rustup override` is needed.

## Why `stable`

Using the `stable` channel means:

- The project relies only on features that have passed Rust's stabilization process.
- No nightly-only APIs are used, reducing breakage risk from upstream changes.
- CI and onboarding are predictable; there is no need to track a specific nightly date.

If a specific compiler version must be pinned for reproducibility, the `channel` value can be changed to an exact version string (e.g., `"1.78.0"`).

## Modifying This File

**Adding a component.** If a new tool is needed project-wide (e.g., `llvm-tools-preview` for coverage), append it to the `components` array:

```toml
components = ["rustfmt", "clippy", "llvm-tools-preview"]
```

**Switching channels.** Change `channel` to `"nightly"` or a pinned version only with deliberate team agreement, as this affects every build in the repository.

**Do not delete this file.** Removing it causes each developer's default toolchain to be used, which may differ in version or available components, leading to inconsistent formatting, lint results, or compilation errors.

## Relationship to the Rest of the Build

This file is the foundation for every other Rust-based step in the project:

- **`cargo build` / `cargo test`** — compile and test against the pinned compiler.
- **`cargo fmt --check`** — requires the `rustfmt` component declared here.
- **`cargo clippy`** — requires the `clippy` component declared here.
- **CI pipelines** — rely on this file to provision the correct toolchain automatically, with no extra configuration in the CI YAML.

No module calls into this file and it calls nothing; it is purely declarative configuration consumed by the toolchain layer.