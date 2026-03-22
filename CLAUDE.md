# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development (runs TS watch + Svelte dev server concurrently)
npm start

# Production build (compile TS → build/, build Svelte → svelte/public/, copy static files)
npm run build

# Individual steps
npm run tsc-watch       # TypeScript watch with auto-rebuild
npm run public:dev      # Svelte Rollup watch (run inside svelte/ or via root npm start)
npm run public:build    # Svelte production build

# Testing the tool locally
cat some-file.log | node bin/global.js
tail -f /var/log/app.log | node bin/global.js --port 8080
```

No test suite exists (`npm test` is a stub).

## Architecture

Web Log Viewer is a CLI tool that pipes stdin logs to a browser UI via WebSocket. It has two parts:

### Server (`src/` → compiled to `build/`)
- **`server.ts`**: Express HTTP server + WebSocket server. Reads stdin line-by-line, parses each line, indexes tokens, and broadcasts `LogMessage` events to all connected WebSocket clients. Each client has independent state (mode, filter, scroll offset).
- **`config.ts`**: CLI parsing via `commander` (port, `--parser`, `--index-keys`, `--stdout`).
- **`log-index.ts`**: Accent-insensitive full-text token index. Used to match filter queries against log messages.
- **`defaultMessageParser.ts`**: Parses each line as JSON5. Users can provide a custom `--parser <script.js>` that exports a function `(line: string) => object`.
- **`stream-utils.ts`**: Wraps Node.js readline into a `@most/core` stream.

### Client (`svelte/src/` → built to `svelte/public/`)
- **`App.svelte`**: Main component. Renders the virtualized log table (fixed `ROW_HEIGHT=30px`), filter input, and wires up keyboard shortcuts. Scroll position determines which rows are visible.
- **`log-store.ts`**: Svelte writable store managing WebSocket lifecycle, incoming message accumulation, formatter application, and filter state. This is the central state hub.
- **`log-formatter.ts`**: Persists formatter function config in `localStorage`/`sessionStorage`. The formatter is user-defined JavaScript executed via `eval()` — intentional by design.
- **`LogMessageDetails.svelte`**: Modal for viewing a single log message. Uses Ace editor for syntax-highlighted JSON5 display and for editing the formatter function.

### Data Flow
```
stdin → readline → parser → log-index → WebSocket broadcast → log-store → App.svelte (virtual table)
```

### Two View Modes
- **Tail**: Auto-scrolls to follow latest messages (live follow).
- **Static**: Frozen at a specific offset; detail viewer shows prev/next navigation.

## Key Design Decisions
- **Formatter via `eval()`**: The client evaluates user-provided JS to transform raw log objects into display columns. Formatter config is per-session (sessionStorage) with fallback to localStorage.
- **Virtual scrolling**: Only visible rows render. Row height is fixed at `ROW_HEIGHT` pixels.
- **Multi-client**: Each WebSocket client is independent — different clients can have different filters/modes simultaneously.
- **Frontend bundler**: Rollup (configured in `svelte/rollup.config.js`); separate from root TypeScript compilation via `tsc`.


## Git / GitHub
- `origin` = `rjbmanvr/web-log-viewer` (fork) — push commits here
- Issues and PRs target `rjbma/web-log-viewer` (upstream)
- SSH remote uses the `github_rjbmanvr` host alias
- When opening a PR: `gh pr create --repo rjbma/web-log-viewer --head rjbmanvr:main`
- on commit messages, don't include information about co-authoring the code with Claude or any other tool