# frontend/src/components/Navbar

A small set of React components that render the authenticated user menu in the site’s top navigation bar. It provides a dropdown UI with profile, settings, and logout actions, and integrates with app auth state and routing.

## Architecture

- Entry points
  - `frontend/src/components/Navbar/DropdownMenu.jsx`: The concrete dropdown UI and behavior used inside a `<li>` in the navbar.
  - `frontend/src/components/Navbar/DropdownItem.jsx`: A presentational helper that renders a single dropdown item as a `react-router-dom` `<Link>` with optional icon and state.
  - `frontend/src/components/Navbar/index.js`: Barrel file re-exporting the `./Navbar` default (not included here) so consumers can `import Navbar from '.../Navbar'`.
- Data/control flow
  - `DropdownMenu` consumes auth context via `useAuth()` to get `loggedUser` and `setAuthState`.
  - Clicking the user avatar/name toggles `dropdown` local state to show/hide the menu.
  - Each action in the menu is a `DropdownItem` which either routes to a path (profile/settings) or invokes a handler (logout). The profile item passes `loggedUser` via router `state`.
  - Logout calls `setAuthState(userLogout)` to mutate global auth state; `userLogout` is provided by the app’s services.

## frontend/src/components/Navbar/DropdownItem.jsx

Renders one clickable item inside the dropdown as a `react-router-dom` `Link`.

- Responsibilities
  - Present an item with optional icon and text that navigates to a route or stays on `#` when `url` is omitted.
  - Optionally invoke a provided `handler` on click (e.g., for logout).
  - Optionally attach router `state` to the navigation.
- Props
  - `handler: function | undefined` — Passed to the `onClick` of the `Link`. If provided, it runs before navigation.
  - `icon: string | undefined` — CSS class string for an `<i>` tag (e.g., `ion-person`). When provided, an `<i className={icon}></i>` is rendered before the text.
  - `text: string` — Visible label of the item.
  - `url: string | undefined` — Route for the `to` prop. Defaults to `"#"` when not provided.
  - `state: any | undefined` — Passed to `Link`’s `state` for downstream route consumption.
- Render details
  - Root element: `<Link className="dropdown-item" ...>`.
  - When `icon` is truthy, an `<i>` icon is prepended, followed by a space and the `text`.
- Dependencies
  - `react-router-dom` `Link` for client-side navigation.
- Notable behavior
  - Because the root is a `Link`, providing only a `handler` without `url` will still render as a link to `#`. If `handler` triggers navigation or state changes, ensure it prevents default when necessary.

## frontend/src/components/Navbar/DropdownMenu.jsx

The user avatar/name dropdown that appears in the navbar and exposes Profile, Settings, and Logout actions.

- State and context
  - `const [dropdown, setDropdown] = useState(false);` — Controls menu visibility.
  - `const { loggedUser, setAuthState } = useAuth();` — Auth context hooks from `../../context/AuthContext`.
  - Derives `username` and `image` from `loggedUser || {}`.
- Event handlers
  - `handleClick`: `setDropdown(prev => !prev)` toggles the menu. Bound to the avatar/name container `onClick` and the menu `onMouseLeave`.
  - `logout`: Calls `setAuthState(userLogout)` to update auth state using the `userLogout` service.
- Rendered structure
  - Root: `<li className="nav-item dropdown">`.
  - Toggle: `<div className="nav-link dropdown-toggle cursor-pointer" onClick={handleClick}>` containing `<Avatar alt={username} className="user-pic" src={image} />` and `{username}`.
  - Menu: `<div className="dropdown-menu" style={{ display: dropdown ? "block" : "none" }} onMouseLeave={handleClick}>` containing items:
    - `<DropdownItem icon="ion-person" text="Profile" url={`/profile/${username}`} state={loggedUser} />`
    - `<DropdownItem icon="ion-gear-a" text="Settings" url="/settings" />`
    - `<div className="dropdown-divider"></div>`
    - `<DropdownItem icon="ion-log-out" text="Logout" handler={logout} />`
- Behavior notes
  - Menu visibility strictly controlled via inline `display` style toggled by local state; no Bootstrap JS is used despite Bootstrap-like class names.
  - `onMouseLeave` of the menu toggles visibility. If the user re-enters quickly, this can cause a flicker due to toggle semantics; consider explicit close instead of toggle when leaving if you change the behavior.
  - Profile item includes `state={loggedUser}` so the profile route can read router state for immediate render without refetching.
- Dependencies
  - `useAuth` from `../../context/AuthContext` for auth; assumes `loggedUser` shape has `username` and `image`.
  - `userLogout` from `../../services/userLogout` to produce new auth state.
  - `Avatar` from `../Avatar` to display the user image; passed `alt`, `className="user-pic"`, and `src`.
  - `DropdownItem` (described above) to render links.

## frontend/src/components/Navbar/index.js

A barrel re-export that exposes the default export from `./Navbar`.

- Purpose
  - Allows consumers to `import Navbar from '.../components/Navbar'` rather than from `./Navbar`. The actual `Navbar` component implementation lives in `frontend/src/components/Navbar/Navbar` (not included in this snippet) and is re-exported here.
- Code
  - `export { default } from "./Navbar";`

## Gotchas

- Auth assumptions: `DropdownMenu` destructures `username` and `image` from `loggedUser || {}`. If `loggedUser` is null and the menu is rendered, `username` will be `undefined`. This affects the displayed label and the profile URL (`/profile/undefined`). Ensure `DropdownMenu` is only rendered for authenticated users or guard the profile item when unauthenticated.
- Logout navigation: The logout item uses `DropdownItem` without a `url`, so it renders a `Link` to `#`. The `handler` triggers `setAuthState(userLogout)` only; it does not redirect. If your logout flow requires redirecting, handle navigation in `userLogout` or augment the handler accordingly.
- Toggle logic: Both clicking the avatar and leaving the menu call the same toggle function. Rapid sequences can desync expected open/close state. If you change triggers (e.g., add outside click handling), prefer explicit `setDropdown(false)` for closes.
- Router state: Passing `loggedUser` via `state` on the profile link makes the profile page dependent on router-provided state if it reads it. Changing the `loggedUser` shape or removing `state` may break consumers expecting this preloaded data.
- CSS/JS expectations: Classes like `dropdown`, `dropdown-menu`, `dropdown-item`, and `dropdown-toggle` resemble Bootstrap but behavior is implemented manually. Introducing Bootstrap’s JS or different CSS may conflict with the inline `display` control used here.