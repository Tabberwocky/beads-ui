# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Beads UI is a local-first web UI for the [Beads](https://github.com/steveyegge/beads) (`bd`) CLI issue tracker. It runs as a daemon serving an Express backend with WebSocket real-time updates to a lit-html SPA frontend.

## Commands

```bash
npm run all            # lint + tsc + test + prettier:check (full CI suite)
npm start              # start dev server with debug logging
npm run build          # esbuild bundle → app/main.bundle.js
npm test               # vitest run (all projects)
npm run test:watch     # vitest in watch mode
npx vitest run path/to/file.test.js  # run a single test file
npm run tsc            # TypeScript type-check (noEmit)
npm run lint           # ESLint
npm run prettier:write # auto-format
npm run prettier:check # format check
```

## Architecture

### Two-process design

- **Server** (`server/`): Express app + WebSocket server. Shells out to the `bd` CLI (`server/bd.js`) for all data mutations. Watches the Beads SQLite database for changes (`server/watcher.js`) and pushes updates to connected clients via subscription-based refresh.
- **Frontend** (`app/`): Vanilla JS SPA using `lit-html` for templating. No build framework — esbuild bundles `app/main.js` into `app/main.bundle.js`. In dev, the server serves source files directly.

### WebSocket protocol

`app/protocol.js` is shared between client and server. All messages are JSON with request/reply correlation by `id`. The server can also push unsolicited events (`snapshot`, `upsert`, `delete`).

### Frontend data flow

`app/main.js` bootstraps the shell and wires together:
- **State** (`app/state.js`): Minimal store with subscription (filters, view, workspace).
- **Data layer** (`app/data/`): Subscription-based stores that manage issue lists per filter/view. `providers.js` is the data-layer factory; `subscriptions-store.js` tracks active list subscriptions.
- **Views** (`app/views/`): Issues list, epics, board (kanban), detail panel, dialogs.
- **Router** (`app/router.js`): Hash-based routing (`#issues`, `#epics`, `#board`, `#detail/<id>`).

### Server data flow

`server/index.js` → creates HTTP server → `server/app.js` (Express) + `server/ws.js` (WebSocket handler). `server/subscriptions.js` is a registry of active list subscriptions. `server/watcher.js` watches the SQLite DB and triggers debounced refresh of active subscriptions.

### CLI

`bin/bdui.js` → `server/cli/index.js` → subcommands (`start`, `stop`, `status`) in `server/cli/commands.js`. Supports daemonization.

## Code Conventions

- **All runtime code is `.js`** with JSDoc type annotations (TypeScript `checkJs` mode). `.ts` files in `types/` are for type definitions only.
- **Naming**: `lower_snake_case` for variables/params, `camelCase` for functions, `PascalCase` for classes, `UPPER_SNAKE_CASE` for constants, `kebab-case` for files.
- **Type imports** use JSDoc `@import` syntax at top of file:
  ```js
  /**
   * @import { X, Y } from './file.js'
   */
  ```
- Braces required for all control flow, even single-line bodies.
- Optional chaining only for intentionally nullable values.
- JSDoc on all functions: declare all `@param`, add `@returns` only when return type is not self-evident.

## Testing

Two vitest projects configured in `vitest.config.mjs`:
- **`node`**: `server/**/*.test.js` and other non-app tests — Node environment.
- **`jsdom`**: `app/**/*.test.js` — jsdom environment with `test/setup-vitest.js` (suppresses Lit dev-mode warnings).

Tests use active-verb naming (not "should ..."), follow setup → execution → assertion structure.

## Pre-commit Validation

Run before submitting changes:
1. `npm run tsc`
2. `npm test`
3. `npm run lint`
4. `npm run prettier:write`

## Debug Logging

Uses the `debug` package with `beads-ui:*` namespaces.
- Browser: `localStorage.debug = 'beads-ui:*'` then reload.
- Server/CLI: `DEBUG=beads-ui:* bdui start`

## Metaswarm

This project uses [metaswarm](https://github.com/dsifry/metaswarm) for multi-agent orchestration. Use `/start-task` to begin tracked work. See `AGENTS.md` for issue tracking conventions and the agent workflow.
