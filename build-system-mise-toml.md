# Build System — mise.toml

# Build System — `mise.toml`

## Purpose

`mise.toml` is the project-level tool version manifest. It declaratively pins the exact versions of all runtime and build tools required to develop, build, and test this project. By committing this file to version control, every contributor gets a consistent, reproducible toolchain with no manual setup beyond installing mise itself.

## How It Works

When a developer enters the project directory, mise reads `mise.toml` and automatically activates the specified tool versions. If a tool isn't installed locally, mise downloads and installs it on first use.

Activation happens via shell hooks (`eval "$(mise activate bash/zsh/fish)"`) or the `mise exec` / `mise run` commands.

## Declared Tools

| Tool | Version | Role |
|------|---------|------|
| `just` | 1.48 | Command runner. Repeats common project tasks (build, test, lint, etc.) without relying on OS-specific Make quirks. |
| `pnpm` | 10.33 | JavaScript package manager. Handles dependency installation, workspace management, and script running for the project's Node/TypeScript code. |
| `rust` | 1.92 | Rust toolchain (rustc + cargo). Required to compile any Rust source in the project. |

## Developer Workflow

### Initial Setup

```bash
# Install mise (if not already installed)
curl https://mise.run | sh

# Activate mise in your shell (one-time)
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc

# Enter the project — mise auto-installs pinned tools
cd /path/to/project
```

### Day-to-Day Usage

```bash
# Verify active tool versions
mise current

# Run a task defined in a justfile
just build

# Run pnpm commands
pnpm install
pnpm test
```

### Updating a Tool Version

1. Edit the version string in `mise.toml`.
2. Commit the change so the rest of the team picks it up.

```toml
# Example: upgrade pnpm
pnpm = "10.34"
```

## Conventions

- **Always pin exact versions** (e.g., `"1.48"`, not `"^1"` or `"latest"`). This guarantees reproducibility across machines and CI.
- **One tool, one line.** Keep the `[tools]` section flat and alphabetically sorted for readability.
- **Avoid `~/.config/mise/config.toml` overrides** for tools declared here. Local overrides can mask version mismatches that will break CI.

## CI Integration

CI pipelines should call `mise install` early in the pipeline to ensure all tools are available:

```yaml
# GitHub Actions example
- name: Install tools
  run: mise install
- name: Build
  run: just build
```

This guarantees CI uses the same pinned versions defined in `mise.toml`.