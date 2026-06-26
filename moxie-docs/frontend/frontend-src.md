# Frontend routing shell and layout (frontend/src/App.jsx and frontend/src/routes)

This area defines the application’s top-level layout shell for all routed pages and the mounting point for nested routes. It exists to provide a consistent header/footer around whatever route content is currently active via React Router’s Outlet, and to serve as the parent element for any child route definitions located under `frontend/src/routes`.

## Architecture

- Entry: `frontend/src/App.jsx` is the root layout component rendered by the React Router configuration.
- Composition: It renders a `Navbar` in a `<header>`, an `<Outlet />` in `<main>` where matched route elements render, and a `Footer` in `<footer>`.
- Route mounting: Child route elements (pages, redirects, guards) are defined elsewhere (typically where the router is created) and render into `Outlet`. The directory `frontend/src/routes` is the conventional home for route components and utilities that plug into this shell.
- Data/control flow: React Router selects a route based on the current URL, creates the element tree for that match, and injects the matched child element under `Outlet`. `App` itself does not do data fetching or navigation; it is a pure layout wrapper.

## frontend/src/App.jsx

Responsibility
- Provides the global app frame: header with `Navbar`, main content region for active route via `Outlet`, and footer with `Footer`.
- Establishes a stable structure that all pages inherit without duplicating navigation or footer markup.

Key behavior from source
- Imports and renders `Navbar` from `./components/Navbar` and `Footer` from `./components/Footer`.
- Uses `Outlet` from `react-router-dom` inside `<main>` as the insertion point for the active child route element.
- No props, state, side effects, or conditional rendering; the component is deterministic and presentational.

Rendered structure
- `<header>`: always contains `<Navbar />`.
- `<main>`: always contains `<Outlet />`; the currently matched child route will render here.
- `<footer>`: always contains `<Footer />`.

Implications for routing and guards
- Because `App` is a parent route element, any route-level loaders, actions, or guards (e.g., components that navigate on auth checks) should be attached to the child routes whose elements appear in the `Outlet`. `App` itself performs no auth checks or redirects.

## frontend/src/routes

Responsibility
- Serves as the directory for route components, route trees, and any guard/redirect components that are mounted under `App`’s `Outlet`.

What’s defined
- The repository layout indicates the directory exists at `frontend/src/routes`, but no concrete files are shown here. The routing system relies on this directory to supply the child route elements that render in `App`’s `<Outlet />`.

How it fits into the shell
- Route elements defined in files under `frontend/src/routes` are referenced by the router configuration (commonly created alongside or above `App`) and, when matched, will render inside the `Outlet` provided by `App`.

## Gotchas

- No auth or redirects in App: `frontend/src/App.jsx` contains no logic for authentication, authorization, or redirect behavior. If your pages require guards, implement them in the route elements that render within `Outlet` or in the router configuration (e.g., wrapper components or route loaders/actions).
- Layout assumptions: Because `Navbar` and `Footer` are always rendered by `App`, any full-screen pages must account for the fixed header/footer layout when handling scroll/height. Changes to `App`’s semantic tags (`header`, `main`, `footer`) may affect global styles and accessibility.
- Outlet dependency: Removing or relocating `<Outlet />` will break rendering of all child routes. Ensure it remains within the layout where route content should appear.