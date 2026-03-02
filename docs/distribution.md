# Distribution & Publishing

How Photon gets from source code to users' machines.

---

## Distribution Channels

Photon is distributed through **three channels**, each triggered by pushing a git tag (`v*`):

| Channel | What ships | Install command | Target audience |
|---|---|---|---|
| **PyPI** | Pre-built binary wheels | `pip install photon-imager` | Easiest path — anyone with Python |
| **GitHub Releases** | Compressed tarballs | Download + extract | Users without Python, CI pipelines |
| **Source** | Cargo workspace | `cargo build --release` | Rust developers, contributors |

### Why PyPI for a Rust binary?

Photon has zero Python code. It uses **maturin** with `bindings = "bin"` — this is a trick to distribute a compiled Rust binary as a Python package. Maturin packages the Rust binary into a platform-specific `.whl` file, and `pip install` simply extracts the binary onto `$PATH`. The user never imports anything in Python — they just get the `photon` CLI command. This gives us PyPI's massive distribution network (mirrors, caching, `pip install` muscle memory) without writing a single line of Python.

---

## Channel 1: PyPI (`pypi.yml`)

**Trigger:** `git push` of a `v*` tag, or manual `workflow_dispatch`

### Build matrix

| Platform | Runner | Build method |
|---|---|---|
| macOS Apple Silicon | `macos-14` | `maturin-action@v1` (native) |
| Linux x86_64 | `ubuntu-latest` | `maturin` via pip (native) |
| Linux aarch64 | `ubuntu-24.04-arm` | `maturin` via pip (native ARM runner) |

All builds are **native** — no cross-compilation. This matters because `ort` (the ONNX Runtime crate) downloads platform-specific prebuilt binaries at compile time, which don't work under cross-compilation.

### Linux: manylinux compatibility

Linux wheels use `--compatibility manylinux_2_39` (glibc 2.39, GCC 14 baseline). This is unusually high for a manylinux tag — most Python packages target `manylinux_2_28` or lower. The reason: `ort`'s prebuilt ONNX Runtime links against `CXXABI_1.3.15` (GCC 14), which isn't available in older manylinux baselines. This means Photon's Linux wheels require a fairly modern distro (Ubuntu 24.04+, Fedora 40+, etc.).

### macOS vs Linux build split

macOS uses `PyO3/maturin-action@v1` directly. Linux uses a **manual maturin install via pip** instead. The split exists because `maturin-action` injects `--manylinux off` when you set `manylinux: 'off'`, and `--manylinux`/`--compatibility` are aliases — passing both causes maturin to reject the duplicate. By installing maturin via pip on Linux, we bypass the action entirely and call `maturin build --compatibility manylinux_2_39` directly.

### Publishing

Uses `pypa/gh-action-pypi-publish@release/v1` with **OIDC trusted publishing** — no API tokens stored in secrets. The GitHub Actions environment `pypi` is configured as a trusted publisher on PyPI. The `id-token: write` permission lets the action mint an OIDC token that PyPI verifies.

### Package identity

- **PyPI name:** `photon-imager` (the name `photon` was taken; `photonai` already existed)
- **Version:** Kept in sync between `Cargo.toml` (workspace) and `pyproject.toml`
- **License:** MIT OR Apache-2.0

---

## Channel 2: GitHub Releases (`release.yml`)

**Trigger:** `git push` of a `v*` tag

### Build matrix

| Platform | Runner | Artifact |
|---|---|---|
| macOS Apple Silicon | `macos-14` | `photon-aarch64-apple-darwin.tar.gz` |
| Linux x86_64 | `ubuntu-latest` | `photon-x86_64-unknown-linux-gnu.tar.gz` |
| Linux aarch64 | `ubuntu-24.04-arm` | `photon-aarch64-unknown-linux-gnu.tar.gz` |

Each build compiles with `cargo build --release --target <triple>`, then packages the binary into a gzipped tarball. The release job collects all three tarballs and creates a GitHub Release with auto-generated release notes via `softprops/action-gh-release@v2`.

### Not supported

- **macOS Intel (x86_64-apple-darwin):** Dropped in v0.7.3 — `ort` has no prebuilt ONNX Runtime binaries for Intel Mac cross-compilation from an ARM runner.

---

## Channel 3: Source (`cargo build`)

For Rust developers:

```bash
git clone https://github.com/hejijunhao/photon
cd photon
cargo build --release
# Binary at target/release/photon
```

`ort` downloads ONNX Runtime prebuilt binaries during compilation (via its build script), so no manual library installation is needed. The only requirement is a Rust toolchain.

---

## CI Quality Gate (`ci.yml`)

**Trigger:** Push or PR to `master`

Before anything gets tagged and published, CI enforces:

| Check | Runner(s) | Command |
|---|---|---|
| Compile | macOS-14, Ubuntu | `cargo check --workspace` |
| Tests | macOS-14, Ubuntu | `cargo test --workspace` (226 tests) |
| Formatting | Ubuntu | `cargo fmt --all -- --check` |
| Lints | Ubuntu | `cargo clippy --workspace -- -D warnings` |

---

## Release Workflow (end-to-end)

```
1. Bump version in Cargo.toml (workspace) + pyproject.toml
2. Commit + push to master
3. CI runs (check, test, fmt, clippy)
4. Tag: git tag v0.X.Y && git push --tags
5. Two workflows fire in parallel:
   ├── pypi.yml  → build 3 wheels → publish to PyPI
   └── release.yml → build 3 tarballs → create GitHub Release
6. Users install:
   ├── pip install photon-imager          (PyPI)
   ├── Download from GitHub Releases      (tarball)
   └── cargo build --release              (source)
```

---

## Key Files

| File | Purpose |
|---|---|
| `pyproject.toml` | Maturin config — package name, version, `bindings = "bin"` |
| `Cargo.toml` (root) | Workspace version, shared dependencies |
| `crates/photon/Cargo.toml` | CLI crate manifest (what maturin builds) |
| `.github/workflows/pypi.yml` | PyPI wheel builds + trusted publishing |
| `.github/workflows/release.yml` | GitHub Release tarballs |
| `.github/workflows/ci.yml` | Pre-merge quality gate |

---

## Lessons Learned (from the changelog)

The PyPI publishing journey from v0.7.0 to v0.7.10 was a 10-version saga of CI fixes:

1. **v0.7.0** — Initial PyPI setup with maturin `bindings = "bin"`
2. **v0.7.1** — Renamed `photon-ai` → `photon-imager` (name collision)
3. **v0.7.2** — `native-tls` → `rustls-tls` (eliminate OpenSSL dependency)
4. **v0.7.3** — Dropped Intel Mac (no `ort` prebuilt binaries)
5. **v0.7.4–v0.7.5** — manylinux Docker images too old for `ort`'s glibc requirements
6. **v0.7.6** — Native ARM64 runner instead of cross-compilation
7. **v0.7.7** — `maturin-action` upload → `pypa/gh-action-pypi-publish` (Docker failures)
8. **v0.7.8–v0.7.9** — `--manylinux`/`--compatibility` flag conflict → split build jobs
9. **v0.7.10** — `manylinux_2_38` → `manylinux_2_39` (GCC 14 ABI requirement)

The root cause of most issues: `ort`'s prebuilt ONNX Runtime binaries impose hard constraints on the build environment (modern glibc, specific platform availability), which conflicts with Python's "build on old systems for max compatibility" culture.
