# Frontend components: Navbar, CommentList, and FeedToggler

This area contains shared UI components that compose page-level features: a site-wide navigation bar, a comment list with item rendering, and a feed toggler used to switch among feed sources. These components exist to centralize visual conventions and state-handling patterns so features remain consistent across pages.

## Architecture

- Entry points: These components are imported by pages/feature containers; there is no standalone route or server entry here.
- Composition: Page containers render `Navbar` at the app shell, `FeedToggler` within feed pages to select a feed scope, and `CommentList` to display comments for an article or entity.
- Data flow: Components are expected to receive data via props and raise minimal events upward (e.g., selection changes). They should remain presentational with optional lightweight internal state for UI concerns (active tab, pending fetch flags); data fetching should live in higher-level containers.

## frontend/src/components/Navbar

- Responsibility: Renders the global navigation/header, including brand link, primary navigation links, and user session controls (sign in/up vs. user dropdown) as appropriate for the app.
- Expected props and patterns:
  - `currentUser` or equivalent session shape from the app shell. When present, render authenticated links (e.g., New Post, Settings, Profile). When absent, render Sign in / Sign up links.
  - `onLogout?`: optional callback to trigger a sign-out flow from a user menu item.
  - Avoid fetching in this component; derive labels/links from props or a shared route config.
- Composition and state:
  - Stateless with respect to routing; uses link components from the app router (e.g., a `Link` primitive) and highlights the active route via router-provided helpers or a passed `activePath` prop.
  - Keep a small internal state only for UI affordances (dropdown open/close). Do not store user data here.
- Side effects and dependencies:
  - Relies on the app’s router/link component and icon set. No direct API calls.

## frontend/src/components/CommentList

- Responsibility: Displays a vertical list of comments for a given resource, supports empty/loading states, and exposes user actions like delete (owner/moderator) when applicable.
- Expected props and patterns:
  - `comments`: array of comment DTOs from the parent container. A typical item includes `id`, `body`, `author` (with `username`, `image`), `createdAt`, and optionally `canDelete` or a currentUser comparison to decide controls.
  - `onDelete?(id)`: optional async handler to delete a comment. The component should optimistically remove the comment only if the handler resolves.
  - `isLoading?`: show skeletons or a spinner when true; avoid initiating fetches here.
  - `emptyMessage?`: string to display when `comments.length === 0` and not loading.
- Composition and state:
  - Pure presentational mapping from `comments` to `CommentItem` rendering; minimal state for pending delete per-item (e.g., disable delete button while the promise resolves).
  - Do not mutate `comments` prop; request the parent to revalidate data after `onDelete`.
- Error handling:
  - If `onDelete` rejects, restore any optimistic UI and surface an inline error per-item.
- Accessibility:
  - Each comment should be a list item within an ordered/unordered list; buttons have labels such as “Delete comment by <username>”.

## frontend/src/components/FeedToggler

- Responsibility: Renders tabs/links to switch between different feed sources/scopes (e.g., Global Feed, Your Feed, Tag feed), delegating actual data fetching and routing to the parent container.
- Expected props and patterns:
  - `activeKey`: string identifying the current feed scope (e.g., `global`, `personal`, `tag:react`).
  - `items`: array of `{ key, label, href? | onSelect? }` where:
    - Provide `href` when feed switching is route-driven; component renders links.
    - Provide `onSelect(key)` for in-place toggling; component renders buttons.
  - `disabledKeys?`: optional set/array to disable unavailable feeds (e.g., “Your Feed” when logged out).
- Composition and state:
  - Stateless with respect to selection; highlights the `activeKey` item. No internal storage of feed data.
  - Supports tag feeds by rendering dynamic labels (e.g., `#react`) passed via `items`.
- Accessibility and behavior:
  - Render as a tablist or nav with appropriate aria roles; ensure focus styles and keyboard navigation.

## Gotchas

- Keep data fetching out of these components. They should be controlled by parent containers for predictable SSR/CSR behavior and testability.
- Avoid coupling to global singletons (stores, contexts) unless explicitly part of the app’s design; prefer props so these components remain reusable.
- Deletion in `CommentList` must be guarded: only show delete controls when authorized (parent should compute this) and handle async failures gracefully without leaving the UI inconsistent.
- In `FeedToggler`, treat keys as stable identifiers. Changing key semantics will break active-state logic and deep-linking if you expose `href`s.
- In `Navbar`, do not directly manipulate auth state. Always invoke the provided `onLogout` or route to sign-out; side effects belong in higher layers.