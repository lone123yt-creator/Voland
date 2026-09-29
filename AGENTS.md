# Voland — Base44 Development Environment

## What this is

Voland is a Nintendo Switch emulator, web-first. A C11 core compiles to
WASM via Emscripten; a Solid.js + Vite frontend serves it. The project is
in early development (Phase 1 — Load & Memory). Nothing executes guest
code yet.

## Architecture (two-stage boot)

1. **`wasm-build`** (one-shot, `emscripten/emsdk:latest`): compiles the C
   core with `emcmake cmake` (Unix Makefiles — the emsdk image has no
   ninja) and copies `switch_core.{js,wasm}` to `platform/web/public/core/`.
2. **`web`** (`node:22`): installs pnpm 9, runs `pnpm install --frozen-lockfile`,
   then `pnpm dev -- --host 0.0.0.0` (Vite dev server on port 5173, mapped
   to host port 3000). Depends on `wasm-build` completing successfully.

## Key facts

- **Emscripten ≥6.0.9 required** (CMakeLists.txt enforces it). The
  `emscripten/emsdk:latest` image currently has 6.0.10.
- **emsdk image has `make` but not `ninja`** — use `-G "Unix Makefiles"`.
- **pnpm@10** — pnpm 9 rejects the `pnpm-workspace.yaml` (missing `packages`
  field). pnpm 10 accepts it and installs cleanly with `--frozen-lockfile`.
- **COOP/COEP headers** are set in `vite.config.ts` (`server.headers`).
  The app requires `crossOriginIsolated` for its 5.25 GiB shared
  `WebAssembly.Memory` (memory64). In an iframe preview this may not be
  available; the boot sequence handles it gracefully with a visible error.
- **`__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS`** is passed from the host
  environment so Vite accepts the preview proxy's Host header.
- **No external secrets needed** — the emulator is self-contained.
- **`platform/web/public/core/`** is gitignored (build artifacts).

## Verifying it works

```sh
docker compose -f docker-compose.base44.yml up -d --build
docker compose -f docker-compose.base44.yml ps
curl -sI http://localhost:3000/   # expect 200 + COOP/COEP headers
```

The boot log panel (right side of the page) shows the boot sequence:
capability detection, shared-memory allocation, CPU/GPU worker startup.
If `crossOriginIsolated` is false (iframe limitation), a fatal error
appears — this is expected behavior, not a bug.

## Native build (optional, for C tests)

```sh
cmake --preset native-noop && cmake --build --preset native-noop
ctest --preset native-noop
```
