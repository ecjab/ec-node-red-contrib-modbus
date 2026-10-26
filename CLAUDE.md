# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
npm install

# Lint (uses StandardJS with auto-fix)
npm run lint

# Run all tests
npm test

# Run a single test file
npx mocha ./test/units/modbus-client-test.js --reporter dot --timeout 8000

# Run only core tests
npm run test:core

# Run with coverage
npm run coverage

# Build (transpiles src/ → modbus/ via Babel + Uglify, minifies HTML)
npm run build

# Local dev link
npm run dev-link

# Clean build output
npm run clean
```

**Do not edit files in `modbus/`** — that directory is the Gulp build output of `src/`. All source changes belong in `src/`.

## Architecture

This is a Node-RED contribution package providing Modbus TCP, UDP, and Serial communication nodes. Node-RED requires nodes to be registered as pairs: a `.js` backend and an `.html` frontend, both named after the node.

### Source layout

```
src/
  modbus-basics.js          # Shared utility functions (time unit helpers, rate calc, status helpers)
  modbus-client.js          # Config node: manages Modbus connection lifecycle
  modbus-read.js            # Polling reader node (interval-based)
  modbus-getter.js          # One-shot reader node (input-triggered)
  modbus-flex-getter.js     # Getter with dynamic FC/address from msg.payload
  modbus-write.js           # Write node (input-triggered)
  modbus-flex-write.js      # Write with dynamic address/value from msg.payload
  modbus-flex-fc.js         # Arbitrary function code execution
  modbus-flex-connector.js  # Dynamic client switching
  modbus-flex-sequencer.js  # Sequential read/write operations
  modbus-server.js          # Built-in Modbus server (for testing/demo)
  modbus-queue-info.js      # Queue monitoring node
  modbus-response.js        # Response parsing/formatting node
  modbus-response-filter.js # Filter/transform Modbus responses
  modbus-io-config.js       # IO configuration node (maps CSV/file to addresses)
  core/
    modbus-core.js          # Shared helpers: FC mapping, ObjectId, message tracking
    modbus-client-core.js   # XState FSM for connection state, unit-id resolution
    modbus-queue-core.js    # Per-unit-id command queuing (Maps indexed 0–255)
    modbus-server-core.js   # Server-side helpers
    modbus-io-core.js       # IO file parsing helpers
  *.html                    # Node-RED editor UI (paired with each .js)
```

### Key architectural patterns

**Connection state machine** (`src/core/modbus-client-core.js`): The Modbus Client node uses an XState FSM with states: `new → init → connected → activated → queueing/sending/reading/writing → closed/failed/stopped`. State transitions drive all connection management and reconnect logic.

**Per-unit-id queuing** (`src/core/modbus-queue-core.js`): Commands are buffered in a `Map<unitId, command[]>` (256 slots, one per Modbus unit ID). When `parallelUnitIdsAllowed` is true, all unit queues drain concurrently; otherwise they drain sequentially via `unitSendingAllowed` round-robin.

**`// SOURCE-MAP-REQUIRED` anchor**: Every `src/**/*.js` file contains this comment. Gulp's `releaseJSContent` replaces it with `require('source-map-support').install()` before transpiling.

**Namespace pattern**: Core modules use a global namespace object (`de.biancoroyal.modbus.*`) with guard assignments (`de = de || {...}`) to allow safe re-use across Node-RED's runtime environment.

**Message tracking**: `modbus-core.js` generates BSON ObjectIds as `messageId` values. The client stores outgoing messages in a `Map` keyed by `messageId` so response nodes can correlate replies back to original messages.

### Build pipeline

`gulp` (default task) runs in sequence: clean → minify HTML → transpile+uglify JS (Babel ES2019→ES2015) → copy locales/icons → generate JSDoc → update CHANGELOG. The `code/` directory holds non-uglified transpiled output (for debugging). Published files are in `modbus/`.

### Testing structure

- `test/core/` — pure unit tests of core modules (no Node-RED runtime)
- `test/units/` — Node-RED node tests using `node-red-node-test-helper` with mock flows defined in `test/units/flows/*.js`
- `test/e2e/` — end-to-end tests that require a live Modbus server (long timeout ~100s)
- `test/helper/` — shared test utilities: `getPort()` finds a free TCP port; `cleanFlowPositionData()` strips x/y/z from exported flows

Tests import directly from `src/` (not `modbus/`), so no build step is needed to run them.

### Linting

StandardJS (`standard`) with auto-fix. Ignores `modbus/`, `code/`, `examples/`, and test flow JSON files. The `standard` config in `package.json` declares mocha globals (`chai`, `expect`, `should`, `assert`, `sinon`).

### Coverage thresholds (nyc)

Branches: 75%, Lines/Functions/Statements: 85%. Coverage is measured only over `src/**/*.js`.
