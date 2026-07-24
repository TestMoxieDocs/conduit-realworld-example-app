# Backend service runtime and dependencies

This area defines the backend service runtime, dependencies, and developer scripts used to run the API server and manage the database. It exists to provide the process entry point, dependency graph, and Sequelize tooling for models and migrations.

## Architecture

- Entry point: `index.js` (referenced by `main` and scripts in `backend/package.json`). This is where the Express app is created, middleware registered (e.g., CORS, JSON parsing), routes mounted, and the HTTP server started. The file is not shown here, but all control enters through this module when running `npm start` or `npm run dev`.
- HTTP framework: Express 5.x (`express` dependency). Any API routes are expected to be defined and mounted from `index.js` or modules it requires.
- AuthN/AuthZ libraries: `jsonwebtoken` for JWT handling and `bcrypt` for password hashing and verification. If present in code, JWT verification middleware would guard protected routes; `bcrypt` would be used in signup/login flows and password updates.
- Database access: Sequelize ORM (`sequelize`, `pg`, `pg-hstore`) with CLI support via `sequelize-cli`. PostgreSQL is the backing database driver.
- Configuration: `dotenv` to load environment variables (e.g., DB credentials, JWT secrets) early in process bootstrap.
- CORS: `cors` for cross-origin access control.
- Project structure surface: `backend/models/` and `backend/migrations/` directories are present for Sequelize model definitions and schema migrations, respectively. Their contents are not included in this snapshot but are integral to the data model and migration flow.

## backend/package.json

This file defines how the backend service is executed and which libraries it relies on.

- Scripts
  - `dev`: `node --watch index.js` launches the server with Node’s watch mode for hot-reload on file changes. Entry point is `index.js`.
  - `start`: `node index.js` starts the server in standard mode.
  - `sqlz`: `npx sequelize-cli` exposes the Sequelize CLI. Use this to run commands like `npx sequelize-cli db:migrate`, `db:seed:all`, or `model:generate`.
- Main: `index.js` is the module Node resolves when requiring this package and is the server entrypoint.
- Dependencies
  - `express@^5.2.1`: HTTP server/router used to define API endpoints.
  - `cors@^2.8.6`: CORS middleware likely configured in `index.js`.
  - `dotenv@^17.3.1`: Loads env vars from a `.env` file before configuring Sequelize and JWT.
  - `sequelize@^6.37.8`, `pg@^8.20.0`, `pg-hstore@^2.3.4`: ORM and Postgres driver for DB connectivity.
  - `sequelize-cli@^6.6.5` (devDependency): CLI to generate and run migrations, seeders, and models.
  - `bcrypt@^6.0.0`: Password hashing (e.g., `bcrypt.hash`, `bcrypt.compare`).
  - `jsonwebtoken@^9.0.3`: JWT signing/verification for stateless auth.

## backend/models/

Sequelize model definition directory. Typical responsibilities include:
- Defining model schemas (attributes, types, validations) and associations.
- Exporting initialized models via a central `index.js` that reads `process.env` for DB configuration (often using `dotenv`).
- These models are consumed by route handlers in `index.js` or in modular route/controller files to perform CRUD and business logic.

Because the concrete model files aren’t shown here, consult this directory in your local checkout for actual model names, fields, and associations. Those determine the request/response data contracts of any API routes built atop Sequelize queries.

## backend/migrations/

Sequelize migration directory. Typical responsibilities include:
- Versioning schema changes with up/down functions.
- Creating tables and columns referred to by models in `backend/models/`.

Run migrations with the provided CLI script, for example:
- `npm run sqlz db:migrate` to apply migrations to the database configured via environment variables.
- `npm run sqlz db:migrate:undo` to roll back the last migration.

The actual migration files will indicate the precise schema (tables, columns, constraints) that API routes expect to exist.

## Gotchas

- Entry point coupling: `backend/package.json` hardcodes `index.js` as the entry. If you reorganize server bootstrap, update both the `main` field and the `dev`/`start` scripts accordingly or your server will not start.
- Environment loading: Ensure `dotenv` is initialized before using any env-dependent modules (Sequelize config, JWT secret). A common pattern is `require('dotenv').config()` at the very top of `index.js` and the Sequelize config.
- Sequelize CLI context: `sequelize-cli` expects a config file (commonly `config/config.json` or `config/config.js`) and a conventional project layout. If you move `models/` or `migrations/`, update the CLI config or it won’t find your artifacts.
- Dependency expectations: The presence of `bcrypt` and `jsonwebtoken` implies password-based auth with JWTs. Rotate and protect the JWT secret and cost factors via env vars; mismatches between hashing cost or JWT algorithms on different environments can break login flows.
- Postgres compatibility: `pg-hstore` is often required by Sequelize for certain JSON/hstore column types. If your models use these types, ensure your database has the necessary extensions enabled and that the versions are compatible with `pg@8.x`.