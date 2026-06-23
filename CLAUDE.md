# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is PMTiles

PMTiles is a single-file archive format for tiled map data that enables serverless map applications on cloud storage (S3, R2, GCS). This is a polyglot monorepo containing reference implementations across multiple languages, a format specification, and deployment tooling.

## Repository Structure

| Directory | Language | Purpose |
|-----------|----------|---------|
| `js/` | TypeScript | Core PMTiles client library (npm: `pmtiles`) |
| `app/` | TypeScript + Solid.js | PMTiles Viewer web app (pmtiles.io) |
| `openlayers/` | TypeScript | OpenLayers integration (npm: `ol-pmtiles`) |
| `serverless/aws/` | TypeScript | AWS Lambda function for serving tiles |
| `serverless/cloudflare/` | TypeScript | Cloudflare Workers integration |
| `serverless/shared/` | TypeScript | Shared test suite for serverless packages |
| `python/pmtiles/` | Python | Core library + CLI tools (PyPI: `pmtiles`) |
| `python/rio-pmtiles/` | Python | Rasterio plugin (PyPI: `rio-pmtiles`) |
| `cpp/` | C++ | Header-only reference implementation (`pmtiles.hpp`) |
| `spec/v3/` | Markdown | PMTiles v3 format specification + test fixtures |

Each package is independently versioned and manages its own dependencies — there is no root-level workspace.

## Build and Test Commands

### JavaScript / TypeScript

Each JS/TS package is built and tested independently from its own directory.

```bash
# js/ — core library
cd js
npm run build       # compile (tsup)
npm test            # run tests (tsx + MSW)
npm run biome       # format and lint fix
npm run biome-check # lint check only

# app/ — web viewer
cd app
npm run build       # vite build
tsc --noEmit        # type-check

# openlayers/
cd openlayers
npm run build
npx prettier --check .

# serverless/aws/
cd serverless/aws
npm run build       # produces Lambda ZIP + CloudFormation stack
tsc --noEmit

# serverless/cloudflare/
cd serverless/cloudflare
npm run build
npm test            # runs shared/index.test.ts
```

### Python

```bash
# python/pmtiles/
cd python/pmtiles
python -m unittest test/test_*

# python/rio-pmtiles/
cd python/rio-pmtiles
pip install -e .
PYTHONPATH=. pytest tests
```

### C++

```bash
cd cpp
make test     # compiles test.cpp with clang and runs it
make indent   # clang-format (Google style)
```

### Version consistency check

```bash
python .github/check_examples.py   # validates version alignment across packages and HTML examples
```

## Architecture

### Core library (`js/`)

- Minimal by design: only external dependency is `fflate` (decompression).
- Exports `PMTiles` (archive reader), `Protocol` (MapLibre protocol handler), and low-level tile-id utilities.
- Supports HTTP range requests to fetch only the needed bytes from a remote archive.
- Built with `tsup`; linted/formatted with **Biome** (`js/biome.json`).

### Serverless packages

Both `serverless/aws/` and `serverless/cloudflare/` wrap the `js/` core and add cloud-provider-specific request/response handling. They share a test suite at `serverless/shared/index.test.ts`.

### Python packages

`python/pmtiles/` is pure Python with no external dependencies (provides both library API and CLI). `python/rio-pmtiles/` depends on rasterio and mercantile for raster-to-PMTiles export.

### C++ (`cpp/pmtiles.hpp`)

Header-only library; no external dependencies. Uses `minunit.h` (included) for unit tests.

### Spec (`spec/v3/spec.md`)

Canonical format specification. The `.pmtiles` files in `spec/v3/` are test fixtures used by implementations.

## Linting and Formatting

- **JS/TS (`js/`, `serverless/`)**: Biome — run `npm run biome` (fix) or `npm run biome-check` (check only).
- **JS/TS (`openlayers/`)**: Prettier.
- **C++**: clang-format with Google style via `make indent`.

## CI

`.github/workflows/actions.yml` runs two parallel jobs:

- **build**: compiles all JS/TS packages and generates TypeDoc; deploys `app/` to GitHub Pages on `main`.
- **test**: runs all language test suites + the version consistency check.

`check_examples.py` validates that packages depending on `pmtiles` use a compatible major version and that HTML examples in `app/` reference correct CDN versions.
