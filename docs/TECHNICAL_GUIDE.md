# Technical Guide

## Overview
MSHP-IDE is a static SPA that runs Python via Pyodide in a Web Worker. The
main thread handles UI, storage, and turtle rendering. A service worker adds
COOP/COEP headers for optional cross-origin isolation.

The project also ships a Skulpt-based variant (`skulpt.html`) with a fully
separate code path and assets.

## Directory structure
- index.html: Pyodide SPA shell and CSP
- skulpt.html: Skulpt SPA shell and CSP
- assets/app.js: Pyodide UI/runtime logic (router, editor, storage, share)
- assets/skulpt-app.js: Skulpt UI/runtime logic (independent copy)
- assets/worker.js: Pyodide runtime worker
- assets/styles.css: Pyodide styling
- assets/skulpt-styles.css: Skulpt styling (independent copy)
- assets/fflate.esm.js: gzip fallback for browsers without CompressionStream
- assets/skulpt-fflate.esm.js: Skulpt gzip fallback (independent copy)
- pyodide-0.29.1/pyodide/: self-hosted Pyodide runtime
- vendor/skulpt/skulpt-dist-master/: Skulpt runtime + stdlib
- sw.js: COI service worker

## Boot flow
1. index.html loads assets/app.js as a module.
2. init() binds UI handlers, registers the service worker, and checks runtime
   support (Worker and WebAssembly).
3. ensureRuntimeCompatibility() triggers a one-time reload if COI is needed.
4. IndexedDB is opened; a memory fallback is used if blocked.
5. A Web Worker is spawned and initialized with Pyodide.

## Routing
Hash-based routing supports GitHub Pages:
- /#/ : landing
- /#/p/{projectId} : editable project
- /#/s/{shareId}?p={payload} : snapshot
- /#/embed : embed view with query settings

## Storage model
IndexedDB database: mshp-ide (Pyodide)
IndexedDB database: mshp-ide-skulpt (Skulpt)
Object stores:
- projects: { projectId, title, files, assets, lastActiveFile, updatedAt }
- blobs: { blobId, data }
- drafts: { key, overlayFiles, deletedFiles, draftLastActiveFile, updatedAt }
- recent: { key: "recent", list: [projectId...] }

When IndexedDB is unavailable, a Map-based in-memory store is used.

## Worker runtime
assets/worker.js:
- importScripts() loads Pyodide from indexURL.
- loadPyodide() initializes the runtime.
- Stdout/stderr are proxied to the main thread.
- input() is implemented via a main thread queue.
- Each run resets /project in the Pyodide FS and writes files/assets.
- Entry point is executed with runpy.run_path(..., run_name="__main__").

## Turtle pipeline
- TURTLE_SHIM replaces the standard turtle module in the worker.
- Turtle events are serialized and posted to the main thread.
- The main thread renders to a canvas with animation and speed controls.
- Pointer/touch events are normalized and sent back to the worker.

## Sharing and snapshots
- Share serializes project state to JSON and encodes as UTF-8 bytes.
- Payload compression:
  - Uses CompressionStream("gzip") when available.
  - Falls back to gzipSync from fflate for compatibility.
- Payload encoding: base64url with a prefix:
  - g.<data> for gzip
  - u.<data> for uncompressed
- shareId:
  - SHA-256 via crypto.subtle when available.
  - Fallback to FNV-1a based hash when crypto.subtle is missing.
- Snapshots are immutable; local edits are stored in drafts.

## CSP and COI
- CSP (meta tag in index.html) includes:
  - script-src 'self' 'wasm-unsafe-eval' 'unsafe-eval'
  - worker-src 'self' blob:
- sw.js injects COOP/COEP/CORP headers for cross-origin isolation.

## Browser compatibility
Key fallbacks in assets/app.js:
- TextEncoder/TextDecoder polyfills
- UUID generation without crypto.randomUUID
- CompressionStream/DecompressionStream -> fflate gzip
- crypto.subtle digest -> FNV-1a
- navigator.clipboard -> document.execCommand
- IndexedDB -> in-memory Map store
- Pointer events -> mouse/touch handlers
- fetch() worker load -> direct Worker URL

## Limits and safeguards
- RUN_TIMEOUT_MS: 10 seconds (soft interrupt then hard stop)
- MAX_OUTPUT_BYTES: 2,000,000 bytes
- MAX_FILES: 30
- MAX_SINGLE_FILE_BYTES: 50,000
- MAX_TOTAL_TEXT_BYTES: 250,000

## Updating Pyodide
1. Replace pyodide-0.29.1/pyodide with a new version.
2. Update the indexURL in assets/app.js (worker init).
3. Ensure pyodide.js and pyodide.asm.wasm match the new version.

## Swapping Pyodide and Skulpt pages (AI agent instructions)
If the Skulpt experiment should become the default version:
1. Make a backup copy of `index.html` (optional).
2. Replace `index.html` with a copy of `skulpt.html`.
3. Ensure the topbar/landing links still point to the alternate page.
4. Keep both `assets/app.js` and `assets/skulpt-app.js` intact so rollback is easy.

To switch back to Pyodide, restore the original `index.html` (or copy it back
from version control) and keep `skulpt.html` as the alternate page.

## Local development
- Use a static server (serve.bat or any HTTP server).
- No build step is required.
- Service worker can be disabled by removing sw.js registration.

## Known limitations
- Sharing excludes assets.
- Snapshots are not encrypted; they are encoded and optionally compressed.
- External network requests are blocked by CSP.
