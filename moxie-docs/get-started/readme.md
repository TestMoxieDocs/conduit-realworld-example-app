# Repository contributor onboarding docs area

This area provides the top-level contributor documentation and governance files. It exists to help engineers install, run, and contribute to the project consistently, and to set expectations for behavior and pull request quality.

## Architecture

Control and information flow for contributor interactions:
- README.md: Primary entry point for new users and contributors. Covers setup, configuration, running, and testing. It links to backend-specific files in `backend/`.
- .github/pull_request_template.md: GitHub UI entry that pre-populates PR descriptions. Lives under `.github/` so GitHub automatically loads it on new PRs. (Note: currently empty in this repo.)
- CODE_OF_CONDUCT.md: Community standards and enforcement guidelines referenced implicitly for all interactions (issues, PRs, discussions). No code-level coupling, but it governs behavior across the repo.

Data/behavior touchpoints:
- Local environment: README directs creation of `.env` per `backend/.env.example` and optional edits to `backend/config/config.js`.
- Package manager scripts: README relies on `npm` workspace commands (notably `-w backend`) to run Sequelize CLI and install DB drivers scoped to the backend workspace.
- Database: README documents creating/seeding a SQL database through Sequelize CLI.

## README.md

Path: `README.md`

Responsibility
- Serves as the definitive setup and run book for the fullstack app (React / Vite + SWC frontend; Express.js / Sequelize backend; PostgreSQL by default). Includes links to a live demo and an alternate CRA branch.

Key flows and contracts
- Prerequisites: Requires Git, Node.js `v18.11.0+`, npm, a SQL database, and an editor. Node version is explicitly constrained.
- Installation:
  - Clone repo, `cd conduit-realworld-example-app`, `npm install` at the workspace root.
- Configuration:
  - Create a `.env` at repo root.
  - Populate it using `backend/.env.example`.
  - (Optional) tweak Sequelize in `backend/config/config.js`.
  - If not on PostgreSQL, install the appropriate DB driver into the backend workspace using `npm install -w backend <driver>` (examples provided for `mysql2`, `mariadb`, `sqlite3`, `tedious`, `oracledb`). Notes that `pg`/`pg-hstore` are already installed.
  - Uses Sequelize v6 docs for installation reference.
- Database lifecycle:
  - Create DB with `npm run sqlz -- db:create`.
  - Seed with `npm run sqlz -- db:seed:all`.
  - Documents that `npm run sqlz` is an alias for `npx -w backend sequelize-cli` and shows `-- --help` to discover commands.
- Development usage:
  - Start dev with `npm run dev`.
  - Exposes two local endpoints post-start: web at `http://localhost:3000/` and API at `http://localhost:3001/api`.
- Testing:
  - Run `npm run test`.
- Production:
  - Build/start with `npm run start`.
- Licensing and acknowledgments: MIT; credits RealWorld resources and a guide on writing READMEs.

Important dependencies and assumptions
- Assumes npm workspaces and a `backend` workspace exist and that `sequelize-cli` is installed/configured such that `npx -w backend sequelize-cli` works via the `npm run sqlz` alias.
- Assumes environment variables defined in `backend/.env.example` are respected by the backend at runtime and by Sequelize CLI for DB connectivity.

What would bite you if you change it
- Changing the documented Node.js version may confuse contributors using toolchains pinned to `v18.11.0+`.
- Modifying command examples (e.g., dropping `-w backend`) will break workspace-scoped installs and Sequelize CLI execution if the monorepo layout is preserved.
- Editing port expectations without aligning actual dev server configs will cause confusion between the documented `3000` (web) and `3001` (API) endpoints.

## .github/pull_request_template.md

Path: `.github/pull_request_template.md`

Responsibility
- Provides the default pull request body template when contributors open PRs on GitHub. It is the control point for standardizing PR context, test plans, screenshots, and checklists.

Current behavior
- The file is present but empty (`null` content in the repository snapshot). As a result, GitHub will not prefill any template when opening a PR, and contributors must format PRs manually.

What would bite you if you change it
- Adding a template immediately affects all new PRs. Keep the content concise and action-oriented to avoid template fatigue.
- Using org-level `.github` repos or branch protection rules may interact with this file; ensure alignment to avoid conflicting checklists.

## CODE_OF_CONDUCT.md

Path: `CODE_OF_CONDUCT.md`

Responsibility
- Defines community standards using the Contributor Covenant v2.0. Applies to all project spaces and representation contexts.

Key sections and enforcement
- Our Pledge/Standards: Enumerates expected and unacceptable behaviors (e.g., no harassment, no doxxing, respectful feedback).
- Enforcement Responsibilities: Community leaders can remove/edit contributions and are expected to communicate moderation reasons where appropriate.
- Scope: Applies across community spaces and when representing the community.
- Enforcement: Reports directed to “the maintainer team” for handling; privacy of reporters is protected.
- Enforcement Guidelines: Four-tier consequence model (Correction, Warning, Temporary Ban, Permanent Ban) with community impact definitions.
- Attribution: References Contributor Covenant homepage and FAQ, with translations available.

What would bite you if you change it
- Altering enforcement language without defining clear contacts or processes can create ambiguity and reduce reporter safety.
- Removing scope or guideline clarity can hinder consistent moderation and expose the project to governance risks.

## Gotchas

- Workspace-scoped installs: The README’s DB driver install commands rely on `npm -w backend`. Running them without `-w backend` may install drivers at the root instead of the backend, breaking backend runtime or bloating the root workspace.
- Sequelize CLI aliasing: `npm run sqlz` is an alias for `npx -w backend sequelize-cli`. If you rename or remove this script, all documented DB lifecycle commands will fail as written.
- Database superuser requirement: Database creation requires an existing superuser. Without it, `db:create` will fail even if environment variables are present.
- Pull request template is empty: Expect inconsistent PR formatting until a proper template is added; reviewers should be prepared to request missing context.
- Code of Conduct reporting contact: The enforcement section references contacting “the maintainer team” but does not list a specific address or handle. Contributors may need to check repository maintainers for contact information.