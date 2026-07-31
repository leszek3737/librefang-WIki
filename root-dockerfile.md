# Root — Dockerfile

# Dockerfile

The production container image for **librefang** — a Rust daemon with an embedded React dashboard. This is a three-stage multi-stage build that compiles the frontend, compiles the Rust binary with compile-time-embedded assets, and assembles a minimal runtime image.

---

## Build Pipeline

```mermaid
flowchart LR
    A["Stage 1: dashboard-builder
    node:20_20_2-alpine["20.20.2-alpine"]"] -->|"static/react"| C
    B["Stage 2: builder
    rust:1_94-slim-bookworm["1.94-slim-bookworm"]"] -->|"librefang binary"| C
    B -->|"packages/"| C
    C["Stage 3: runtime
    node:22.11.0-bookworm-slim"]
    D["deploy/
    docker-entrypoint_sh["docker-entrypoint.sh"]"] --> C
```

### Stage 1 — Dashboard Builder (`dashboard-builder`)

Builds the React frontend that ships inside the Rust binary's static asset directory.

| Decision | Rationale |
|---|---|
| `node:20.20.2-alpine` (pinned minor) | Reproducible builds months after a tag. A floating `node:20-alpine` could silently shift patch versions and produce a different builder image. Matches the Node 20 LTS used by CI (`.github/workflows/ci.yml`, `.github/workflows/dashboard-build.yml`). |
| `corepack@latest` before `corepack prepare` | The keyring bundled in the Node base image goes stale as pnpm rotates signing keys, causing `"Internal Error: Cannot find matching keyid"`. Refreshing corepack first fixes this. |
| `corepack prepare pnpm@10.33.0 --activate` | Bypasses `fetchLatestStableVersion2` (which flakes against the npm registry) by activating the exact version from `packageManager` in `package.json` directly. |
| `pnpm install --frozen-lockfile --ignore-scripts` | Lockfile-strict install with postinstall scripts skipped for hermeticity. |
| Node ≥ 20.19 required | Vite 8 / rolldown's optional native bindings declare `engines: ^20.19.0`. Without it, `pnpm install` silently skips the `linux-x64-musl` binding and `vite build` fails at require-time. |

**Output:** compiled React assets at `/build/static/react`, consumed by Stage 2.

### Stage 2 — Rust Builder (`builder`)

Compiles the `librefang` release binary.

| Decision | Rationale |
|---|---|
| `rust:1.94-slim-bookworm` (pinned minor) | Satisfies the workspace MSRV in `Cargo.toml` (`[workspace.package].rust-version = "1.94.1"`). Floating `rust:1-slim-bookworm` could land on a newer compiler without notice. |
| `libdbus-1-dev` | Required by `libdbus-sys`, a transitive dependency of `keyring`'s `sync-secret-service` feature (added in #3180). Without it the build script panics with exit 101 — the same root cause as #3259, which blocked the v2026.4.27-beta6 image publish. |
| Build cache mounts | `cargo/registry`, `cargo/git`, and `target` are mounted as BuildKit cache mounts so dependency compilation is cached across rebuilds. |

#### Compile-time embedded assets

Several crates use `include_dir!` / `include_str!` macros that require specific source trees to be present at build time. The Dockerfile copies exactly these subtrees (`.dockerignore` excludes the rest):

| Source path | Embedded by | Failure without it |
|---|---|---|
| `sdk/python/librefang` | `librefang-channels` via `include_dir!("$CARGO_MANIFEST_DIR/../../sdk/python/librefang")` in `embedded_sdk.rs` (#5472) | Proc macro panic: `"sdk/python/librefang is not a directory"` |
| `deploy/` | `librefang-api` via `include_str!("../../../deploy/...")` for observability stack configs (#3062) | `"couldn't read deploy/grafana/..."` |
| `crates/librefang-api/static/react` | Copied from Stage 1 output | Frontend not served |

#### Build command

```bash
cargo build --release --bin librefang --features telemetry
```

Only `telemetry` is enabled — this is the full daemon image. Channel adapters run as out-of-process sidecars (#5408 / #5461), so the old `all-channels` / `core-channels` feature aliases no longer exist. This matches the CLI's default feature set.

The resulting binary is copied to `/usr/local/bin/librefang` to escape the cache-mounted `target/` directory.

### Stage 3 — Runtime Image

| Base | `node:22.11.0-bookworm-slim` (pinned Node 22 LTS minor) |
|---|---|
| Rationale for pinning | A floating `node:lts-bookworm-slim` could quietly land on a new major when the `lts` alias rolls forward. |

#### Runtime dependencies

| Package | Reason |
|---|---|
| `ca-certificates` | TLS verification for outbound HTTPS |
| `curl` | Used by the `HEALTHCHECK` |
| `python3`, `python3-venv` | Python SDK runtime support |
| `libicu72` | ICU runtime for text processing |
| `libdbus-1-3` | Runtime `.so` that `libdbus-sys` links against. The keyring init path runs early in boot; if the `.so` can't be resolved, the process exits 101. |
| `gosu` | Privilege-drop tool used by `docker-entrypoint.sh` |

#### Security posture

- A dedicated `librefang` user (uid/gid 1001) is created via `addgroup` / `adduser`.
- CIS Docker Benchmark §4.1 compliance: the login shell is set to `/sbin/nologin` via `usermod`. The Dockerfile deliberately avoids a redundant `groupadd -r librefang && useradd -r ...` block (introduced by #3948) that collides with the already-created user — `groupadd` exits with code 9 ("group already exists"), breaking `docker build` on clean trees.
- `/opt/librefang/packages` is chowned to the `librefang` user since `COPY` defaults to `root:root`.

---

## Entrypoint and Command

```
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["librefang", "start", "--foreground"]
```

The entrypoint (`deploy/docker-entrypoint.sh`) runs as **root** so it can chown bind-mounts and initialize the data directory before dropping privileges via `gosu`. The actual daemon runs as the `librefang` user.

The entrypoint also rewrites `api_listen` based on the `$PORT` environment variable injected by Railway, Render, or Fly.

---

## Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s \
  CMD curl -fsS http://127.0.0.1:${PORT:-4545}/api/ready || exit 1
```

| Aspect | Detail |
|---|---|
| Endpoint | `/api/ready` (not `/api/health`) — changed in #6633 |
| Why readiness, not liveness | Docker's `HEALTHCHECK` does not restart unhealthy containers. Its consumer is Compose's `depends_on: condition: service_healthy` gate, which is readiness semantics: "may dependents start yet?" The old `/api/health` endpoint returns 200 even when its body reports `status: degraded`, so it could never fail the gate. |
| Shell form (`CMD ...`) not exec form | Required so `${PORT:-4545}` expands at runtime from environment variables. |
| Start period (20s) | Gives the daemon time to bind, run `librefang init` on first boot, and start the axum server. |
| Kubernetes | Ignores `HEALTHCHECK` entirely. K8s probes are declared separately in `deploy/kubernetes/`. |

---

## Environment

| Variable | Default | Purpose |
|---|---|---|
| `LIBREFANG_HOME` | `/data` | Data directory for persistent state |
| `PORT` | `4545` (fallback) | Listen port, injected by PaaS providers. Expanded at runtime in the healthcheck and rewritten by the entrypoint. |

---

## Exposed Port

`EXPOSE 4545` — the default API listen port. Override at runtime via `$PORT`.

---

## Key Invariants and Gotchas

1. **Never unpin base image tags.** Every `FROM` line pins a specific minor. Floating tags (`node:20-alpine`, `rust:1-slim-bookworm`, `node:lts-*`) break reproducibility or risk silent major-version bumps.
2. **`sdk/python/librefang` must be copied.** It's embedded at compile time. Only this subtree is needed; `.dockerignore` excludes the rest of `sdk/`.
3. **`deploy/` must be copied into the builder.** Observability configs (Prometheus, Tempo, OTEL Collector, Grafana) are embedded via `include_str!`. `flake.nix` lists the same paths in its source fileset.
4. **Don't re-add the `groupadd`/`useradd` block.** See #3948 — it collides with the existing `librefang` user and breaks the build.
5. **`--ignore-scripts` is intentional** in the dashboard pnpm install. Removing it reintroduces non-hermetic postinstall side effects.