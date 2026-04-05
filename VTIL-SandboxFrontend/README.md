# VTIL-SandboxFrontend

Vue frontend for VTIL-Sandbox.

It connects to the backend at `http://127.0.0.1:8090` and provides an interactive UI for block navigation, CFG analysis, and instruction editing.

## Requirements

- Node.js + npm
- Running backend (`sandbox.exe`) on `127.0.0.1:8090`

## Install

```bat
npm install
```

## Development

```bat
npm run serve
```

Default dev URL:

- `http://127.0.0.1:8080`

## Features

- Upload `.vtil` file and fetch live routine state
- Download modified routine as `.vtil`
- Block list with entry block and predecessor/successor shortcuts
- Interactive CFG:
  - wheel zoom
  - drag pan
  - zoom controls (`+`, `-`, `1:1`, `Fit`)
  - highlighted incoming/outgoing edges for selected block
- Instruction explorer:
  - search by VIP/mnemonic/text
  - configurable search priority
  - next/previous match navigation
  - expandable descriptor + operand metadata rows
  - pseudo-ASM panel
- Schema-guided editing:
  - immediate editor (single immediate operand patch)
  - instruction editor (mnemonic + operand validation/autocomplete)
- Theme toggle (dark/light)
- Connection status indicator and backend error surfacing

## API Dependencies

Frontend expects these backend endpoints:

- `GET /health`
- `GET /api/state`
- `GET /api/schema`
- `POST /api/upload?name=<filename>`
- `POST /api/edit/immediate?...`
- `POST /api/edit/instruction?...`
- `GET /api/download`

## Production Build

```bat
npm run build
```

Output directory is configured in `vue.config.js`:

- `../VTIL-Sandbox/builds/assets`

## Lint

```bat
npm run lint
```
