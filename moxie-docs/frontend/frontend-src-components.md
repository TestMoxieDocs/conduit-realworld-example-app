# Navbar and Feed Toggling Components

These components drive primary navigation and browsing modes in the frontend. The Navbar gates available links by authentication state, while FeedToggler exposes the current feed scope (your/global/tag) and reflects the active tab via context.

## Architecture

- Entry points:
  - `frontend/src/components/Navbar/Navbar.jsx` renders the top navigation bar for the app.
  - `frontend/src/components/FeedToggler/FeedToggler.jsx` renders pill-style tabs used on feed pages.
- Shared state and contracts:
  - Both consume `useAuth` from `../../context/AuthContext` to branch on authentication (`isAuth`).
  - `FeedToggler` additionally consumes `useFeedContext` from `../../context/FeedContext` for active tab (`tabName`) and tag label (`tagName`).
- Composition:
  - Navbar composes `NavItem`, `SourceCodeLink`, and `DropdownMenu` along with `react-router-dom`’s `Link`.
  - FeedToggler composes `FeedNavLink` items that resolve routing and active state styling.
- Control/data flow:
  - On render, contexts provide state. Navbar/FeedToggler compute which links to display. User interactions (clicks) navigate via client-side routing. Active state styling and tab label rendering are delegated to `NavItem`/`FeedNavLink` and the contexts.

## frontend/src/components/Navbar/Navbar.jsx

Responsibility
- Renders the global top navigation bar, including the app brand, a left-aligned source link, and right-aligned navigation items. It shows different sets of links depending on whether the user is authenticated.

Key behavior
- Determines authentication via `const { isAuth } = useAuth();` from `../../context/AuthContext`.
- Always renders the brand link: `<Link className="navbar-brand" to="/">conduit</Link>` using `react-router-dom`’s `Link`.
- Always renders `<SourceCodeLink left />` which appears to align a source link to the left side of the navbar (specifics handled in `SourceCodeLink`).
- Right-side nav list (`<ul className="nav navbar-nav pull-xs-right">`) contains:
  - Always: `<NavItem text="Home" icon="ion-compose" url="/" />`
  - If `isAuth` is true:
    - `<NavItem text="New Article" icon="ion-compose" url="/editor" />`
    - `<DropdownMenu />` (likely user/profile menu; details in that component)
  - If `isAuth` is false:
    - `<NavItem text="Login" icon="ion-log-in" url="/login" />`
    - `<NavItem text="Sign up" url="/register" />`

Routing and state
- Uses `react-router-dom` for client-side routing of brand/home link. `NavItem` is responsible for linking and any active styling of individual nav entries.
- Authentication gating is purely presentational here; no redirects or guards are applied—click targets must be handled/guarded by routes.

Dependencies
- `useAuth` from `../../context/AuthContext` (must provide `isAuth` boolean).
- `NavItem` from `../NavItem` (must accept `text`, optional `icon`, and `url`).
- `SourceCodeLink` from `../SourceCodeLink` (accepts `left` prop).
- `DropdownMenu` from `./DropdownMenu` (shown only when authenticated).

Side effects
- None in this component; it renders based on context.

What’s not obvious
- The “Home” item uses `icon="ion-compose"`, which implies Ionic icon classes; ensure Ionicons CSS is loaded globally for icons to render.
- The presence of `DropdownMenu` suggests profile-related actions live there; removing it will hide access to those actions for authenticated users.

## frontend/src/components/FeedToggler/FeedToggler.jsx

Responsibility
- Renders the set of feed scope tabs (Your Feed, Global Feed, and a contextual Tag tab). It derives which tabs to display and the current active tab from contexts.

Key behavior
- Authentication check: `const { isAuth } = useAuth();` from `../../context/AuthContext`.
- Feed state: `const { tabName, tagName } = useFeedContext();` from `../../context/FeedContext`.
- Rendered structure:
  - Container: `<div className="feed-toggle">` with `<ul className="nav nav-pills outline-active">` for pill navigation styling.
  - If `isAuth` is true, renders: `<FeedNavLink name="feed" text="Your Feed" />`
  - Always renders: `<FeedNavLink name="global" text="Global Feed" />`
  - Conditionally renders a tag tab only when the current tab is tag-scoped: `{tabName === "tag" && <FeedNavLink icon name="tag" text={tagName} />}`

Active state and labeling
- Active tab determination is delegated to `FeedNavLink` in conjunction with `useFeedContext` state. `tabName` values used here are string literals: `"feed"`, `"global"`, and `"tag"`.
- The tag tab label is dynamic: `text={tagName}` and rendered only when `tabName === "tag"`. If `tagName` is empty or undefined in that state, the UI will render an empty label.

Dependencies
- `useAuth` from `../../context/AuthContext` (must provide `isAuth` boolean).
- `useFeedContext` from `../../context/FeedContext` (must provide `tabName` and `tagName`).
- `FeedNavLink` from `./FeedNavLink` (must accept `name`, `text`, and optional `icon` boolean prop; responsible for route and active class management).

Side effects
- None in this component; it renders based on context and delegates navigation to `FeedNavLink`.

What’s not obvious
- “Your Feed” is suppressed for unauthenticated users; only “Global Feed” is visible when `isAuth` is false.
- The tag pill is not a general tag browser; it only appears if the current tab context is explicitly set to `"tag"`.

## Gotchas
- Auth gating expectations: Navbar’s presence/absence of links (e.g., “New Article”, `DropdownMenu`) does not enforce backend or route-level authorization. Ensure routes like `/editor` and any profile actions in `DropdownMenu` are protected server/client-side as needed.
- Context contracts are strict: `useFeedContext` must provide consistent `tabName` values exactly matching `"feed" | "global" | "tag"`; introducing new values without updating `FeedToggler` will result in missing or incorrect tabs.
- Tag tab visibility: The tag pill only renders when `tabName === "tag"`. If you programmatically navigate to a tag route but forget to set `tabName` and `tagName` in `FeedContext`, users won’t see a tag pill and may lose visual context for the active tag.
- Icon dependencies: Navbar uses `icon="ion-compose"` and `ion-log-in`; `FeedToggler` can render a tag icon when `icon` is passed to `FeedNavLink`. Ensure Ionicons (or the expected icon set) CSS is available; otherwise icons will not display.
- Link ownership: `NavItem` and `FeedNavLink` encapsulate routing and active-state CSS. If you change their props or behavior, update all call sites here to keep labels, icons, and URLs consistent.