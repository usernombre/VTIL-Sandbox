# VTIL-Sandbox

VTIL-Sandbox is a local VTIL exploration and editing environment made of:

- `VTIL-Sandbox` (C++ backend): deserializes `.vtil`, validates edits, and exposes JSON/bytes over HTTP
- `VTIL-SandboxFrontend` (Vue 2 frontend): visual graph/instruction UI with schema-guided editing tools

The project is designed for local workflow on `127.0.0.1`.

## Architecture

- Backend API: `http://127.0.0.1:8090`
- Frontend dev server: `http://127.0.0.1:8080`
- End-to-end flow:
  1. Upload `.vtil` to `POST /api/upload`
  2. Backend parses VTIL and exposes routine state/schema
  3. Frontend renders blocks + CFG + instruction table
  4. User applies edits (`/api/edit/*`) and can download updated bytes (`/api/download`)

## Repository Layout

- `VTIL-Core/`: VTIL core libraries and dependencies
- `VTIL-Sandbox/`: backend server source (`main.cpp`)
- `VTIL-SandboxFrontend/`: Vue frontend source
- `CMakeLists.txt`: root build entry point

## Prerequisites

- Windows + MSVC toolchain
- CMake `>= 3.14.5`
- Node.js + npm (frontend)

## Quick Start

### 1) Configure and build backend

```bat
cmake -S . -B output -DVTIL_SANDBOX_BUILD_FRONTEND=OFF
cmake --build output --config Release --target sandbox
```

Use `Debug` while iterating on backend logic:

```bat
cmake --build output --config Debug --target sandbox
```

### 2) Run backend

```bat
output\VTIL-Sandbox\Release\sandbox.exe
```

Debug build:

```bat
output\VTIL-Sandbox\Debug\sandbox.exe
```

Expected log:

- `Backend listening on http://127.0.0.1:8090`

### 3) Run frontend

```bat
cd VTIL-SandboxFrontend
npm install
npm run serve
```

Open:

- `http://127.0.0.1:8080`

## Frontend Highlights

- Toolbar with upload/download, manual refresh, and theme toggle
- Connection status indicator (Connected/Offline)
- Resizable two-pane layout (block list + detail pane)
- CFG viewer with zoom in/out, 1:1 reset, fit-to-view, wheel zoom, and drag pan
- Edge highlighting for incoming/outgoing edges of selected block
- Instruction table with:
  - search by VIP/mnemonic/text (priority-selectable)
  - next/previous match navigation
  - expandable descriptor/operand details
  - pseudo-ASM preview panel
- Editing workflows:
  - Immediate operand editor (validated numeric input)
  - Instruction editor with schema-guided mnemonic/operand validation + autocomplete
- Poll-based routine refresh while backend is online

## Backend API

All endpoints are served from `http://127.0.0.1:8090`.

### Health

- `GET /health`
  - returns: `{ "ok": true }`

### Routine state and schema

- `GET /api/state`
  - returns routine metadata, blocks, CFG edges, instructions, descriptors, and operand metadata
- `GET /api/schema`
  - returns editor schema derived from the loaded routine (mnemonics, discovered registers, and instruction descriptor rows)

### Upload and download

- `POST /api/upload?name=<filename>`
  - content-type: `application/octet-stream`
  - body: raw `.vtil` bytes
- `GET /api/download`
  - returns serialized `.vtil` bytes
  - includes `Content-Disposition` filename

### Edit endpoints

- `POST /api/edit/immediate?block=<vip>&instruction=<index>&operand=<index>&value=<num>`
  - updates one immediate operand value
  - `value` accepts decimal, hex (`0x...`), or negative
- `POST /api/edit/instruction?block=<vip>&instruction=<index>`
  - body: plain instruction text (for example `mov vr0, 0x10:64`)
  - validates mnemonic + operand types against known descriptors/registers

### Error behavior

- Validation or parse failures return `400` with `{ "ok": false, "error": "..." }`
- Unknown route returns `404`
- Internal exceptions return `500`

## Security and Limits

- Loopback-only bind (`127.0.0.1`)
- Request header limit: `64 KB`
- Upload payload limit: `32 MB`
- Per-client socket timeout: `15 s`
- Max concurrent clients: `64`

## Frontend Production Build

```bat
cd VTIL-SandboxFrontend
npm run build
```

By configuration, frontend artifacts are emitted to:

- `VTIL-Sandbox/builds/assets`

## Optional CMake Frontend Target

Root CMake exposes `VTIL_SANDBOX_BUILD_FRONTEND` (default `ON`) and a custom target:

- `sandbox-frontend`

Build it explicitly with:

```bat
cmake --build output --config Release --target sandbox-frontend
```
