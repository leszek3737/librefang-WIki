# Build System — mise.toml

# Build System — `mise.toml`

## Overview

This file is the **mise configuration** for the project. [Mise](https://mise.jdx.dev/) (formerly rtx) is a development environment manager that handles tool version management, replacing tools like `nvm`, `rustup`, and `pyenv`.

When you run commands in this project, mise automatically activates the correct versions of these tools based on this configuration.

## Configured Tools

| Tool | Version | Purpose |
|------|---------|---------|
| `just` | 1.48 | Command runner / task automation |
| `pnpm` | 10.33 | Node.js package manager |
| `rust` | 1.92 | Rust compiler and toolchain |

## How It Works

### Automatic Tool Activation

When you enter this project directory, mise reads `mise.toml` and ensures the specified tool versions are available. This happens through mise's shell integration or the `.mise.toml` detection system.

### Tool Usage

**Just** (`just`) serves as the project's task runner. Common commands:

```bash
just              # List available recipes
just <recipe>     # Run a specific recipe
```

**pnpm** manages JavaScript/TypeScript dependencies if the project includes Node.js components:

```bash
pnpm install
pnpm build
pnpm test
```

**Rust** provides the compiler and toolchain for any Rust components in the project:

```bash
cargo build
cargo test
cargo fmt
```

## Project Integration

```
┌─────────────────────────────────┐
│         mise.toml               │
│  (version declarations)         │
└────────────┬────────────────────┘
             │ mise reads on directory entry
             ▼
┌─────────────────────────────────┐
│    Active Tool Versions         │
│  • just 1.48                    │
│  • pnpm 10.33                   │
│  • rust 1.92                    │
└────────────┬────────────────────┘
             │ available to
             ▼
┌─────────────────────────────────┐
│   Project Commands              │
│  • just (task runner)           │
│  • pnpm (package manager)       │
│  • cargo/rustc (compiler)       │
└─────────────────────────────────┘
```

## Updating Tool Versions

To upgrade a tool version, edit `mise.toml`:

```toml
[tools]
just = "1.50"       # Update from 1.48
pnpm = "10.40"      # Update from 10.33
rust = "1.95"       # Update from 1.92
```

Then run:

```bash
mise install
```

## Relationship to Other Configuration

This file defines **runtime versions** of tools. It complements:

- **`package.json`** — Node.js dependency declarations (uses pnpm)
- **`Justfile`** — Task definitions (uses just to execute)
- **`Cargo.toml`** — Rust package definitions (uses rust toolchain)

## Prerequisites

Mise must be installed on your system:

```bash
# macOS
brew install mise

# Linux
curl https://mise.run | sh

# Verify installation
mise --version
```

After installation, enable mise in your shell (add to `.bashrc`, `.zshrc`, etc.):

```bash
eval "$(mise activate bash)"  # or zsh, fish, etc.
```