# User Guide

## Overview
MSHP-IDE is a static, browser-only Python IDE. It runs CPython via Pyodide in a
Web Worker and provides a file editor, console input/output, and a turtle canvas.
All data stays in the browser.

## Requirements
- A modern browser with WebAssembly and Web Workers.
- A local or hosted static server (file:// URLs are not supported).
- Supported browsers: Chrome, Chromium, Safari.
- Unsupported browsers: Firefox and other non-Chromium/non-Safari engines.

## Quick start
1. Serve the folder locally. You can run `serve.bat` or any static server.
2. Open the site in your browser.
3. Click "New project" and press Run.

## Interface tour
- Landing page: create a new project or open a recent one.
- IDE view:
  - Files panel (left): create, rename, duplicate, delete files.
  - Editor (center): Python code with syntax highlighting.
  - Console (bottom right): stdout/stderr and input.
  - Turtle canvas (top right): turtle drawing and input.

## Projects and files
- Each project contains multiple .py files plus optional assets.
- The active file is the entry point when you click Run.
- File name rules:
  - Allowed characters: A-Z, a-z, 0-9, dot, dash, underscore.
  - Example: `main.py`, `utils.py`.
- Limits:
  - Max files: 30
  - Max file size: 50 KB per file
  - Max total text size: 250 KB

## Assets
- Assets are uploaded via the Assets panel and stored in IndexedDB.
- Assets are available to Python code as files in the project directory.
- Snapshots do not include assets. Use Export if you need to share assets.

## Running code
- Run executes the active file as __main__.
- Stop interrupts execution by restarting the worker.
- Output is shown in the console. Errors are highlighted.
- Input (`input()`) is supported through the console input field.
- Run safeguards:
  - Wall-time timeout: 10 seconds
  - Max output size: 2 MB (output is truncated after that)

## Turtle canvas
- Turtle drawing appears in the turtle panel.
- Focus the canvas to capture keyboard input for turtle events.
- You can drag the turtle with pointer or touch input.
- Use the speed control slider to adjust animation speed.
- Click "Clear" to reset the canvas.

## Share links (snapshots)
- Share creates an immutable snapshot URL for the current project.
- The snapshot is read-only by default.
- You can still edit locally; edits are stored as a draft overlay.
- Remix creates a new editable project from a snapshot.
- Reset discards local draft edits and restores the snapshot baseline.
- Share limits:
  - No assets (share only includes code)
  - Same size limits as above

## Export
- Export offers JSON or ZIP formats.
- JSON includes files and assets encoded in base64.
- ZIP includes files and assets as a standard archive.

## Embed mode
- Use `#/embed` with query parameters:
  - display=side|output|toggle
  - mode=runOnly|consoleOnly|allowEither
  - autorun=1|0
  - readonly=1|0

## Storage and privacy
- Projects and assets are stored in IndexedDB.
- In private browsing modes, storage may be unavailable or non-persistent.
- The app falls back to in-memory storage when IndexedDB is blocked.

## Troubleshooting
- If the app is stuck on loading, refresh once to allow COI setup.
- If sharing fails, reduce the project size or remove large files.
- If input is ignored, make sure the console input is enabled and focused.
