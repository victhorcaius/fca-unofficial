# AGENTS.md

## 1. Project Overview

**fca-unofficial** is a Node.js (≥22.0.0) Facebook Chat API library that emulates browser traffic to automate Messenger. It supports standard HTTP-based messaging, MQTT real-time transport, and E2EE messaging via the optional `meta-messenger.js` bridge. The public API surface is a collection of controller factories wired together at login time.

---

## 2. Project Map

```
fca-unofficial/
├── src/
│   ├── index.js                  # Entry point — login, buildAPI, controller wiring
│   ├── controllers/
│   │   ├── index.js              # Registry: loads every controller by name
│   │   └── *.js                  # One file per API method (factory pattern)
│   ├── e2ee/
│   │   ├── bridge.js             # E2EE bridge to meta-messenger.js
│   │   └── thread.js             # isE2EEChatJid() helper
│   ├── models/index.js           # Placeholder
│   ├── views/index.js            # Placeholder
│   ├── utils.js                  # Re-exports ./utils/index.js
│   └── utils/
│       ├── index.js              # Auto-merges all util modules
│       ├── base.js               # Aggregates all base-parts
│       ├── base-parts/
│       │   ├── auth.js           # makeDefaults, parseAndCheckLogin, saveCookies
│       │   ├── formatters.js     # All format* functions (_formatAttachment, etc.)
│       │   ├── identity.js       # ID/token generation helpers
│       │   ├── network.js        # get/post/postFormData, setProxy
│       │   ├── parsing.js        # getFrom, makeParsable, decodeClientPayload
│       │   └── type.js           # getType(obj)
│       └── format-*.js           # Thin re-export shims
├── test/
│   ├── unit/                     # Jest unit tests (run in CI)
│   ├── integration/              # Skipped unless appstate.json present
│   └── data/                     # Fixtures
├── scripts/                      # Dev/smoke scripts (not tests)
├── package.json
├── pnpm-lock.yaml / package-lock.json
└── jest.config.js
```

---

## 3. Build / Test Scripts

```bash
# Unit tests only (CI default — integration excluded)
npm test

# All unit tests
npm run test:unit

# Integration tests (requires test/appstate.json)
npm run test:integration

# Lint
npm run lint

# Format
npm run prettier
```

- Test runner: **Jest 27**, `--runInBand`, `testEnvironment: "node"`
- Setup file: `test/jest.setup.js` — polyfills Web Streams globals for ESM deps
- Integration tests skip automatically when `test/appstate.json` is absent

---

## 4. Code Style

### General
- **"use strict"** at the top of every file — no exceptions
- CommonJS (`require`/`module.exports`) throughout; no ESM (`import/export`)
- Node ≥22 features are allowed; avoid anything that breaks on Node 22

### Controller pattern
Every controller is a **factory function**:
```js
module.exports = function(defaultFuncs, api, ctx) {
  return function myMethod(arg, callback) { … };
};
```
- Always create a `Promise` + `resolveFunc`/`rejectFunc` pair and return it
- Always honour a callback when provided, falling back to the Promise
- Errors go to `log.error("myMethod", err)` then `return callback(err)`

### Naming
- `camelCase` for functions, variables, module exports
- File names match the exported method name exactly (e.g. `sendMessage.js` → `api.sendMessage`)
- Constants: no all-caps convention; just plain `var`/`const` camelCase

### Linting (ESLint 7)
- Rules: `semi: error`, `linebreak-style: unix`
- `no-unused-vars` is a **warning** — unused params must be prefixed `_`
- `no-empty` allows empty catch blocks

### Logging
- Use `npmlog` — `log.info`, `log.warn`, `log.error`, `log.verbose`
- Never use `console.log` in library code (scripts are exempt)

---

## 5. Guardrails — Never Modify

| Path / Pattern | Reason |
|---|---|
| `pnpm-lock.yaml` / `package-lock.json` | Lockfiles — managed by the package manager only |
| `src/e2ee/bridge.js` | Complex async E2EE bridge; changes break integration tests |
| `src/controllers/index.js` | Central registry; add entries only, never reorder or remove |
| `src/utils/base-parts/auth.js` | Auth/cookie logic shared by all requests; regressions are hard to detect |
| `test/jest.setup.js` | Global Jest polyfills; changing breaks the entire test suite |
| `jest.config.js` | Test runner config; do not add `--forceExit` or change `testEnvironment` |
| `.github/workflows/` | CI definitions; do not alter node version matrix or lint step |
| Any `test/integration/` file | Integration tests depend on live credentials; do not mock or delete |
| `src/utils/base-parts/formatters.js` | 600+ line formatter — changes break message parsing across the board |
