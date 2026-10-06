# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Small Express REST API (users + health check) used as the starter project for the Claude Code course. Course task: set up Claude on the project (`CLAUDE.md`, `.claude/settings.json`, `NOTES.md`) — app code is not meant to change.

## Commands

- `npm run dev` — start the API with `node --watch` on `http://localhost:3000` (port from `PORT`)
- `npm test` — run all tests (built-in `node --test` runner, no Jest/Mocha)
- `node --test tests/users.test.js` — run one test file
- `node --test --test-name-pattern="404" tests/` — run tests whose name matches
- `npm run lint` — ESLint (`eslint:recommended`, `.eslintrc.json`)

CI (`.github/workflows/ci.yml`, Node 22) runs `npm run lint` then `npm test` on every push/PR — both must pass.

## Architecture

- `server.js` builds the Express app, mounts one router per resource, and exports `app`. It only calls `listen()` when run directly (`require.main === module`), so tests import `app` without opening a port. Keep that guard.
- `routes/<resource>.js` — one `express.Router()` per resource, mounted in `server.js` (`/users`, `/health`). Routes do validation and HTTP status codes only.
- `db/store.js` — in-memory data access (module-level array, resets on restart). Routes never touch the data directly; all reads/writes go through store functions.
- `tests/*.test.js` — `node:test` + `node:assert` + `supertest` against the exported `app`. The store is shared module state within a test run, so tests that create data affect later tests.

## Conventions

- CommonJS only: `require` / `module.exports`, not `import`/`export` (ESLint is configured with `sourceType: "script"`).
- Errors are JSON `{ "error": "<message>" }` with the matching status (400 for missing/invalid input, 404 for not found); successful creates return 201 with the created object.
- Route `:id` params are converted with `Number(...)` before calling the store (store compares with `===`).
- New resource = new file in `routes/` + new functions in `db/store.js` + `app.use()` in `server.js` + tests in `tests/`.
- Unused `req`, `res`, `next`, `_` args are allowed by the lint config; don't rename them to silence warnings.
- Config comes from `.env` (see `.env.example`); never read or commit `.env`.

## Secrets

- Never commit credentials of any kind: API keys, secret keys, tokens, passwords, private keys. This includes tests, examples and comments.
- Secrets live only in `.env` / environment variables and are read via `process.env`; `.env.example` holds names with empty or placeholder values.
- Stage files by explicit path (`git add -A`, `git add .`, `git add .env*` are denied) and review `git diff --staged` before committing.
- `.claude/hooks/check-secrets.sh` (PreToolUse hook) blocks `git commit` if staged changes contain `.env*` files or secret-like strings. If it fires, remove the secret — don't work around the hook.
