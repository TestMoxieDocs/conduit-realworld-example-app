# Testing: helpers and frontend test tooling

This area covers the current automated tests around helper utilities and the frontend’s test tooling configuration. It explains where tests live, how they run (based on what’s in package.json), and what the existing assertions verify.

## Architecture

- Test locations
  - Backend helper tests: `backend/helper/helpers.test.js` (Node + Jest-style globals).
  - Frontend helper tests: `frontend/src/helpers/*.test.js` (pattern only; used to organize tests colocated with helpers).
- Frontend tooling
  - `frontend/package.json` defines scripts for building and serving via Vite and declares test-related dev dependencies (`@testing-library/*`, `jest-dom`) that frontend tests can import.
- Execution model
  - Tests in `backend/helper/helpers.test.js` use Jest-style `describe`, `test.each`, and `expect`, implying a Jest-compatible runner is expected for backend helpers.
  - Frontend has Testing Library dependencies, enabling React component and DOM-centric tests colocated in `frontend/src/helpers/*.test.js`.

## backend/helper/helpers.test.js

This file implements unit tests for a helper function named `slugify` that is imported from a sibling module.

- Purpose
  - Verifies that various “Hello World” string variants normalize to the same slug.
- Imports
  - `const { slugify } = require("./helpers");`
    - Contract implied by this test: `./helpers` exports a `slugify` function.
- Test suite
  - `describe("Slugify", () => { ... })`
  - Defines an array `stringsArray` with representative inputs:
    - `"  Hello World  "`
    - `"  Hello WORLD  "`
    - `" HELLO WORLD"`
    - `"Hello World"`
    - `"Hello_world "`
    - `"Hello-world"`
  - Uses `test.each(stringsArray)("%p", (string) => { expect(slugify(string)).toBe("hello-world"); });`
    - For each variant, expects exactly the output string `"hello-world"`.
- Behavioral contract tested
  - Trimming leading/trailing whitespace.
  - Lowercasing.
  - Normalizing spaces/underscores/hyphens between words to a single hyphen (`-`).
  - Idempotency when input is already hyphenated.
- Expected outcome
  - All inputs produce exactly `hello-world`.
- Status and failures
  - A failing test would indicate a breaking change in `slugify` normalization semantics for at least one of the cases above.

## frontend/src/helpers/*.test.js

- Purpose
  - Pattern for colocated frontend helper tests under `frontend/src/helpers/` with filenames ending in `.test.js`.
- Expected tooling
  - Tests in this directory can rely on the dev dependencies declared in `frontend/package.json` (see below), notably `@testing-library/*` and `@testing-library/jest-dom`.
- Usage patterns supported by dependencies
  - DOM queries and assertions via Testing Library.
  - User interaction simulations via `@testing-library/user-event`.
  - Extended matchers such as `toBeInTheDocument` from `@testing-library/jest-dom`.

## frontend/package.json

This file defines the frontend app’s package metadata, build/dev scripts, and dependencies relevant to writing and running tests.

- Scripts
  - `build`: `vite build`
  - `dev`: `vite`
  - `preview`: `vite preview`
- Dependencies
  - Runtime: `axios`, `markdown-to-jsx`, `react`, `react-dom`, `react-paginate`, `react-router-dom`, `web-vitals`.
- Dev dependencies relevant to tests
  - `@testing-library/jest-dom`: `^6.9.1`
  - `@testing-library/react`: `^16.3.2`
  - `@testing-library/user-event`: `^14.6.1`
  - Type defs and Vite React plugin: `@types/react`, `@types/react-dom`, `@vitejs/plugin-react-swc`, `vite`
- Implications
  - Frontend tests can import Testing Library utilities and jest-dom matchers.
  - Test execution should use a runner that supports Jest-style globals or Testing Library’s typical Jest integration. The file does not define a `test` script; add one in this file when integrating a chosen runner.

## Gotchas

- Backend `slugify` contract is strict: it must return exactly `"hello-world"` for all the enumerated inputs in `backend/helper/helpers.test.js`. Changes to whitespace handling, case normalization, or underscore/hyphen collapsing will break these tests.
- Frontend `package.json` includes Testing Library but no `test` script. Ensure your CI and local workflow invoke your chosen test runner explicitly and initialize `@testing-library/jest-dom` if you rely on its matchers in `frontend/src/helpers/*.test.js`.
- The backend test uses CommonJS (`require("./helpers")`). Keep the backend helper module’s export shape compatible with `const { slugify } = require("./helpers");` or update both the implementation and tests together.