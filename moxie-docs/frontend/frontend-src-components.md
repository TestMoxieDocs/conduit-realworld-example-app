# UI composition: Navbar, CommentList, and FeedToggler components

These React components implement the site navigation, comment rendering with author affordances, and the home feed tab toggling. They are thin, composable building blocks that rely on Auth and Feed contexts and a few service/helpers for side-effects and formatting.

## Architecture

- Entry points/components
  - `frontend/src/components/Navbar/Navbar.jsx`: Top nav bar that conditionally renders navigation for authenticated/anonymous users and mounts the profile `DropdownMenu`.
  - `frontend/src/components/CommentList/CommentList.jsx`: Fetches and renders comments for an article based on the URL `slug`. Manages deletion for the logged-in author.
  - `frontend/src/components/FeedToggler/FeedToggler.jsx`: Renders the tab navigation for feeds and tag view, delegating tab changes to `FeedContext`.
- Supporting components
  - `frontend/src/components/Navbar/DropdownMenu.jsx`: Avatar/username trigger and dropdown with profile, settings, and logout.
  - `frontend/src/components/Navbar/DropdownItem.jsx`: Single dropdown item, implemented as a `Link` with optional icon, handler, and navigation state.
  - `frontend/src/components/CommentList/CommentAuthor.jsx`: Renders avatar and username links to the author’s profile, passing profile details via location state.
  - `frontend/src/components/FeedToggler/FeedNavLink.jsx`: Single tab button that calls `changeTab` from `FeedContext`.
- Contexts and services relied upon
  - `useAuth` (from `../../context/AuthContext`) supplies `isAuth`, `loggedUser`, `headers`, and `setAuthState` used for conditional UI and authenticated side effects.
  - `useFeedContext` (from `../../context/FeedContext`) supplies `tabName`, `tagName`, and `changeTab` to coordinate feed selection.
  - Service/helper functions used: `getComments`, `deleteComment`, `userLogout`, and `dateFormatter`.
- Control/data flow
  - Navbar: `Navbar.jsx` reads `isAuth` to choose which links to show; when authenticated it mounts `DropdownMenu.jsx`, which uses `useAuth` and `userLogout` to update auth state.
  - Comment list: `CommentList.jsx` reads the article `slug` from `react-router-dom` params, fetches comments via `getComments({ slug })`, and conditionally enables delete via `deleteComment({ commentId, headers, slug })` for the matching `loggedUser.username`.
  - Feed toggler: `FeedToggler.jsx` reads current `tabName`/`tagName` and renders `FeedNavLink.jsx` buttons that call `changeTab(e, name)`.

## frontend/src/components/Navbar/Navbar.jsx

Renders the top navigation bar and conditionally shows links based on authentication state from `useAuth`.

- Dependencies: `react-router-dom/Link`, `useAuth` from `../../context/AuthContext`, `NavItem`, `SourceCodeLink`, `./DropdownMenu`.
- Behavior
  - Always renders brand link to `/` and a `SourceCodeLink` with `left` prop.
  - Always renders `Home` as `<NavItem text="Home" icon="ion-compose" url="/" />`.
  - If `isAuth` is true:
    - Renders `<NavItem text="New Article" icon="ion-compose" url="/editor" />` and `<DropdownMenu />`.
  - If `isAuth` is false:
    - Renders `<NavItem text="Login" icon="ion-log-in" url="/login" />` and `<NavItem text="Sign up" url="/register" />`.
- Side effects: None directly; defers logout and auth mutation to `DropdownMenu`.
- Styling/classes: Uses the Conduit example’s bootstrap-like classes such as `navbar`, `navbar-light`, `nav navbar-nav pull-xs-right`.

## frontend/src/components/Navbar/DropdownMenu.jsx

Provides the authenticated user’s avatar/username toggle and menu for Profile, Settings, and Logout.

- Local state: `dropdown` boolean via `useState(false)` for showing/hiding the menu.
- Auth context: `const { loggedUser, setAuthState } = useAuth();` and destructures `{ username, image }` from `loggedUser || {}`.
- Logout behavior
  - `logout` calls `setAuthState(userLogout);` (note: it passes the imported `userLogout` function to `setAuthState`). Any consumer of `useAuth` must accept functions to update state in this manner.
- Toggle behavior
  - `handleClick` flips `dropdown`.
  - The menu `<div className="dropdown-menu">` uses inline style `display: dropdown ? "block" : "none"`.
  - `onMouseLeave={handleClick}` collapses the menu when pointer leaves the menu container.
- Rendered items (via `DropdownItem`)
  - Profile: `<DropdownItem icon="ion-person" text="Profile" url={`/profile/${username}`} state={loggedUser} />`.
    - Passes the entire `loggedUser` as navigation `state` to the profile route.
  - Settings: `<DropdownItem icon="ion-gear-a" text="Settings" url="/settings" />`.
  - Divider: `<div className="dropdown-divider"></div>`.
  - Logout: `<DropdownItem icon="ion-log-out" text="Logout" handler={logout} />` with no `url` so it will use `#`.
- Trigger
  - A `div.nav-link.dropdown-toggle` with `onClick={handleClick}` that shows `<Avatar alt={username} className="user-pic" src={image} />` and the `username`.
- Assumptions
  - `loggedUser` can be nullish; avatar and username will render with `undefined` if not present. This component is only mounted when `isAuth` is true in `Navbar.jsx`.

## frontend/src/components/Navbar/DropdownItem.jsx

Single dropdown item element used within `DropdownMenu`.

- Props: `{ handler, icon, text, url, state }`.
- Renders a `react-router-dom` `<Link>` with:
  - `className="dropdown-item"`.
  - `onClick={handler}` (if provided).
  - `to={url || "#"}`; when no `url` is provided, navigates to `#`.
  - `state={state}` to pass location state to the target route.
  - Inner content: `{icon && <i className={icon}></i>} {text}`.
- Side effects: Any click handler runs before navigation. If `url` is omitted and the handler mutates state (e.g., logout), the link will not navigate away from the current page.

## frontend/src/components/CommentList/CommentList.jsx

Fetches and renders comments for the current article `slug`, guards deletion to the author, and exposes a trigger mechanism to refresh from parent.

- Props: `{ triggerUpdate, updateComments }`.
  - `triggerUpdate`: included in the `useEffect` dependency array to re-fetch when it changes (pattern for parent-driven refreshes).
  - `updateComments`: callback invoked after successful delete to let parent update comment state.
- Context/params
  - `const { headers, isAuth, loggedUser } = useAuth();`
  - `const { slug } = useParams();`
- Data loading
  - `useEffect(() => { getComments({ slug }).then(setComments).catch(console.error); }, [slug, triggerUpdate]);`
  - Expects `getComments({ slug })` to resolve to an array; sets local `comments` state (initialized as `[]`).
- Delete workflow (`handleClick(commentId)`) and controls
  - If not authenticated: `alert("You need to login first");` (continues and will still ask for confirmation; early return is not present for this case — see Gotchas).
  - Confirmation: `window.confirm("Want to delete the comment?")`; aborts if user cancels.
  - Calls `deleteComment({ commentId, headers, slug }).then(updateComments).catch(console.error);`.
  - The delete button is only rendered for authenticated users who are the comment author: `isAuth && loggedUser.username === username`.
- Render
  - When `comments?.length > 0`:
    - Maps each comment `{ author, author: { username }, body, createdAt, id }` to a `.card` with body text, footer containing `CommentAuthor` and a `.date-posted` (via `dateFormatter(createdAt)`).
    - If the logged-in user matches, shows a delete button: `<button className="btn btn-sm btn-outline-secondary pull-xs-right" onClick={() => handleClick(id)}><i className="ion-trash-a"></i></button>`.
  - Else: renders `<div>There are no comments yet...</div>`.

## frontend/src/components/CommentList/CommentAuthor.jsx

Renders the author’s avatar and username as links to their profile, passing basic profile details in route state.

- Props: `{ bio, followersCount, following, image, username }`.
- Render
  - First link wraps the avatar: `<Link className="comment-author" state={{ bio, followersCount, following, image }} to={`/profile/${username}`}>` with `<Avatar alt={username} className="comment-author-img" src={image} />`.
  - Second link shows the `username` text with the same `className`, `to`, and `state` payload.
- Notes
  - Both links pass the same `state` object, allowing the profile page to render immediately from navigation state while optionally fetching fresh data.

## frontend/src/components/FeedToggler/FeedToggler.jsx

Renders the pill-style feed tabs and the active tag tab (when in tag mode), wired to `FeedContext`.

- Context: `const { isAuth } = useAuth(); const { tabName, tagName } = useFeedContext();`
- Rendered tabs
  - If authenticated: `<FeedNavLink name="feed" text="Your Feed" />`.
  - Always: `<FeedNavLink name="global" text="Global Feed" />`.
  - If the current tab is a tag: `{tabName === "tag" && <FeedNavLink icon name="tag" text={tagName} />}`. The `icon` prop tells `FeedNavLink` to render the pound icon.
- Styling: `feed-toggle` container with `nav nav-pills outline-active` list.

## frontend/src/components/FeedToggler/FeedNavLink.jsx

Button-rendered nav item that indicates active tab and invokes `changeTab` on click.

- Props: `{ icon, name, text }`.
- Context: `const { tabName, changeTab } = useFeedContext();`
- Behavior
  - Click handler: `handleClick = (e) => { changeTab(e, name); }` — passes the DOM event and the tab `name` to the context.
  - Active class: `className={`nav-link ${tabName === name ? "active" : ""}`}` applied to a `<button>` inside `<li className="nav-item">`.
  - Icon: If `icon` is truthy, renders `<i className="ion-pound"></i>` before the `text`.

## Gotchas

- Comment deletion flow alert does not short-circuit
  - In `CommentList.jsx`, if `!isAuth`, the code calls `alert("You need to login first")` but does not immediately `return`. The next statement still prompts for confirmation and, if confirmed, proceeds to call `deleteComment` with `headers`. Ensure `useAuth().headers` is meaningful when unauthenticated or add an early return for safety if altering behavior.
- Logout `DropdownItem` has no `url`
  - In `DropdownMenu.jsx`, the logout item omits `url`, so `DropdownItem` navigates to `#`. If `setAuthState(userLogout)` is asynchronous and triggers a redirect elsewhere, this is fine; otherwise the current view remains. If you change logout flow, consider whether a route change is desired.
- Menu visibility on mouse leave
  - `onMouseLeave={handleClick}` is attached to the dropdown menu container. Moving the pointer from the toggle to the menu will not close it; only leaving the menu area will. If you change structure/styles, ensure the event still fires as intended.
- Profile state shape
  - `CommentAuthor.jsx` and `DropdownMenu.jsx` pass user/profile data via `Link state`. Any profile page expecting these should be defensive about missing fields and should not assume presence beyond `{ bio, followersCount, following, image }` or the full `loggedUser` object.
- Feed tab names are stringly-typed
  - `FeedToggler.jsx` and `FeedNavLink.jsx` use literal names: `"feed"`, `"global"`, and `"tag"`. Downstream logic in `FeedContext` should handle only these values. Renaming requires coordinated changes in context and any fetch logic.
- Icon classes are hard-coded
  - Icons rely on classes like `ion-compose`, `ion-person`, `ion-gear-a`, `ion-log-out`, `ion-pound`. Ensure the icon font CSS is loaded; otherwise, icons will not render.