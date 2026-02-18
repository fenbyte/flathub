# Generating Offline Dependency Bundles

Three files are gitignored due to size and must be generated locally before building the Flatpak.
All steps assume you are working inside the extracted Fluxer source tree.

---

## Prerequisites

| Tool | Purpose |
|------|---------|
| `node` + `npm` | Bootstrapping pnpm |
| `pnpm` v10.26.0 | Fetching Node.js dependencies |
| `go` ≥ 1.21 | Fetching Go modules |
| `python3` | Running flatpak-cargo-generator |
| `git` | Cloning flatpak-builder-tools |

---

## 0 — Extract the pinned source

```sh
wget https://github.com/fluxerapp/fluxer/archive/cb31608.tar.gz
tar -xf cb31608.tar.gz
cd fluxer-cb31608
```

All subsequent steps run from inside `fluxer-cb31608/`.

---

## 1 — `pnpm-offline-store.tar.gz`

The build runs `pnpm install --offline` twice — once at the repo root and once
inside `fluxer_app/`. Both share the same store, so populate it in one pass.

```sh
npm install -g pnpm@10.26.0

mkdir pnpm-offline-store

pnpm install --store-dir "$PWD/pnpm-offline-store"

cd fluxer_app
pnpm install --store-dir "$PWD/../pnpm-offline-store"
cd ..

tar -czf /path/to/flathub/pnpm-offline-store.tar.gz pnpm-offline-store
```

---

## 2 — `go-module-cache.tar.gz`

The build sets `GOPROXY=file:///run/build/fluxer/go-module-cache,off`, which expects
the Go module download cache layout (`download/` tree with `.zip`, `.info`, and `.mod`
files per module).

```sh

# Optional but recommended: Clear your cache first so we don't pack gigabytes of old junk from other projects!
go clean -modcache

# 1. Download the root modules
go mod download

# 2. Download the app proxy modules
cd fluxer_app/proxy
go mod download

# 3. Download the app scripts modules
cd ../scripts
go mod download

# Go back to the root of the source folder
cd ../..

# Archive only the download/ subdirectory — this is what GOPROXY=file:// expects
tar -czf /path/to/flathub/go-module-cache.tar.gz \
    -C "$(go env GOPATH)/pkg/mod/cache" download
```

---

## 3 — `cargo-sources.json`

Generated from `Cargo.lock` using the official `flatpak-cargo-generator` tool.
This tells flatpak-builder how to vendor all Rust crates before the build starts.

```sh
git clone --depth=1 https://github.com/flatpak/flatpak-builder-tools.git

python3 flatpak-builder-tools/cargo/flatpak-cargo-generator.py \
    fluxer_app/crates/libfluxcore/Cargo.lock \
    -o /path/to/flathub/cargo-sources.json
```

---

## 4 — Build

Once all three files are in the `flathub/` directory alongside the manifest:

```sh
flatpak-builder --force-clean build app.fluxer.Fluxer.yml
```
