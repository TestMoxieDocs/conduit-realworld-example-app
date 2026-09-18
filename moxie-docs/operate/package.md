# Monorepo run/build/test commands and local dev wiring

This area documents how to develop, build, run, and test the app from the package.json scripts present in the root, backend, and frontend workspaces. It exists to make new contributors productive quickly by showing the exact entry points, how the frontend proxies to the backend in dev, and how database lifecycle commands are invoked.

## Architecture

- Root workspace manager (`package.json`):
  - Declares npm workspaces `backend` and `frontend`.
  - Provides composite scripts for dev, DB tooling, tests, and prod start.
- Backend workspace (`backend/package.json`):
  - Node/Express entry via `index.js` with watch and start scripts.
  - Exposes `sequelize-cli` through an `sqlz` script for DB create/seed/etc.
- Frontend workspace (`frontend/package.json` + `frontend/vite.config.js`):
  - Vite-based React app with dev server on port 3000.
  - Dev-time proxy from `/api` to the backend at `http://localhost:3001`.
- Public assets (`frontend/public/robots.txt`):
  - Allow all crawling in all environments unless overridden at deploy.
- Root `README.md`:
  - End-to-end setup flow, environment, and the canonical commands you’ll run.

Control/data flow in local dev:
- You run `npm run dev` at repo root → concurrently starts `backend` (Node on 3001) and `frontend` (Vite on 3000).
- Browser hits `http://localhost:3000` (Vite). Any XHR/fetch to `/api/...` is proxied by Vite to `http://localhost:3001` per `frontend/vite.config.js`.
- DB management is invoked from root via `npm run sqlz -- <sequelize-cli subcommand>` which shells into the backend workspace.

## Root: package.json

Path: `package.json`

- Workspaces:
  - `"workspaces": ["backend", "frontend"]` — enables `-w <workspace>` addressing used in scripts.
- Scripts:
  - `dev`: `concurrently --names 'Node,Vite' -c 'green,blue' 'npm run dev -w backend' 'npm run dev -w frontend'`
    - Entry point for local development. Runs both servers with colored, named logs.
  - `sqlz`: `npx -w backend sequelize-cli`
    - Workspace-scoped shim to `sequelize-cli` installed in `backend`. Use with `--` to pass subcommands, e.g. `npm run sqlz -- db:create`.
  - `start`: `npm run build -w frontend && npm run start -w backend`
    - Production-style start: build the frontend bundle, then start the backend server.
    - Note: this does not serve the built frontend itself from the backend; delivering the built assets is deployment/environment-specific.
  - `test`: `vitest`
    - Runs `vitest` at the repo root. There is no root-level vitest config; it will auto-discover tests if present.
- Dev dependencies:
  - `concurrently` used by `dev`.
  - `@vitejs/plugin-react-swc` and `vitest` available at root for tooling consistency.

Common root workflows (from `README.md`):
- Install deps: `npm install`
- Create DB: `npm run sqlz -- db:create`
- Seed DB (optional): `npm run sqlz -- db:seed:all`
- Dev: `npm run dev` → visit `http://localhost:3000` (frontend) and `http://localhost:3001/api` (backend API)
- Tests: `npm run test`
- Production build/start: `npm run start`

## Backend workspace: backend/package.json

Path: `backend/package.json`

- Entry point: `"main": "index.js"`. All start/watch scripts run `index.js`.
- Scripts:
  - `dev`: `node --watch index.js`
    - Starts the backend with Node’s file watching for live-reload on changes to `index.js` and imported modules (Node’s watch semantics apply).
  - `sqlz`: `npx sequelize-cli`
    - Direct alias to `sequelize-cli`. From root prefer `npm run sqlz -- <cmd>` to ensure correct workspace context.
  - `start`: `node index.js`
    - Starts the backend server without watch. Intended for production.
- Dependencies of note:
  - `express` (server), `cors`, `dotenv` (env loading), `sequelize` (+ `pg`, `pg-hstore`), `bcrypt`, `jsonwebtoken`.
  - `sequelize-cli` as a devDependency enabling migration/seed commands.

Operational expectations reflected in `README.md`:
- Environment variables come from a backend `.env` (see `README.md` “Configuration” referencing `backend/.env.example`).
- Sequelize config is in `backend/config/config.js` (referenced by README), and DB user must exist before `db:create`.

## Frontend workspace: frontend/package.json

Path: `frontend/package.json`

- Type: `"type": "module"` — ESM throughout the frontend.
- Scripts:
  - `dev`: `vite`
    - Runs Vite dev server. Port configured in `frontend/vite.config.js` as 3000.
  - `build`: `vite build`
    - Produces a production bundle in `frontend/dist` (default Vite output path).
  - `preview`: `vite preview`
    - Serves the built bundle locally to validate a production build.
- Dependencies:
  - React 19 (`react`, `react-dom`), routing via `react-router-dom`, HTTP via `axios`, pagination, markdown rendering, and `web-vitals`.
- Dev dependencies:
  - `vite` and `@vitejs/plugin-react-swc` for SWC-based React transforms, plus Testing Library types/utilities.

## Frontend dev server and proxy: frontend/vite.config.js

Path: `frontend/vite.config.js`

- Exports Vite config with React SWC plugin: `plugins: [react()]`.
- Server configuration:
  - `port: 3000` — Vite serves the app at `http://localhost:3000` in dev.
  - `proxy: { '/api': { target: 'http://localhost:3001' } }`
    - Any frontend request to a path beginning with `/api` is proxied to the backend at `http://localhost:3001`.
    - This allows frontend code to use relative `/api/...` URLs without CORS issues in development.

Implications:
- Backend must listen on port 3001 for API requests during `npm run dev` (per README expectations).
- If your backend pathing changes, update the proxy to match, or adjust frontend API base URLs.

## Public robots: frontend/public/robots.txt

Path: `frontend/public/robots.txt`

- Contents:
  - `User-agent: *` then `Disallow:` with no paths — explicitly allows all crawlers to access all paths.
- Vite serves files under `frontend/public` as-is at the site root in dev and in builds, so this becomes `/robots.txt`.

## README.md: operational guidance

Path: `README.md`

- Establishes versions and prerequisites:
  - Node.js `v18.11.0+`, NPM, Git, a SQL database.
- Installation flow:
  - `git clone … && cd conduit-realworld-example-app && npm install`
- Configuration:
  - Create a `.env` at the repo root (loaded by backend via `dotenv`; variables specified in `backend/.env.example`).
  - Optional: adjust Sequelize settings in `backend/config/config.js`.
  - DB driver install guidance for non-PostgreSQL databases using `-w backend` scoped installs.
- Database lifecycle:
  - Create DB: `npm run sqlz -- db:create`
  - Help: `npm run sqlz -- --help`
  - Seed: `npm run sqlz -- db:seed:all`
- Running:
  - Dev: `npm run dev` → Frontend on `http://localhost:3000/`, API at `http://localhost:3001/api`.
  - Tests: `npm run test` (root `vitest`).
  - Production: `npm run start` builds frontend then starts backend.

## Gotchas

- Port assumptions in dev:
  - The Vite proxy in `frontend/vite.config.js` targets `http://localhost:3001`. Ensure the backend listens on 3001; otherwise, API calls from the frontend will fail. The README references API at `http://localhost:3001/api`.
- Passing arguments to sequelize-cli:
  - When running from root, always use `npm run sqlz -- <subcommand>` so arguments after `--` are forwarded to `sequelize-cli` (e.g., `db:migrate`, `db:seed:all`). Omitting `--` won’t pass the subcommand.
- Production `npm run start` does not serve the built frontend:
  - The root `start` script builds the frontend and starts the backend, but no script in these files serves the `frontend/dist` via the backend. Your deployment must serve the static assets separately or add static serving to the backend.
- Environment configuration lives in backend:
  - `.env` is referenced by the backend and `sequelize-cli`. Mismatched DB credentials or missing superuser will cause `npm run sqlz -- db:create` to fail (see README warnings).
- Test runner scope:
  - `npm run test` runs `vitest` at root. If you add tests under frontend/backend, ensure vitest is configured or colocate tests discoverable by the root runner, or add workspace-specific test scripts if needed.