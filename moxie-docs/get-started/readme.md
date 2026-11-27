# RealWorld Routes and UI Entry Points (Backend/Frontend)

This area wires the Express API routes for users, profiles, articles, comments, and tags, and the key React UI entry components that consume them. It exists to expose the RealWorld spec endpoints behind token‑protected routers and render the corresponding client features (home feed, comments, navbar) on the frontend.

## Architecture

- Express backend
  - Entry to each feature is via routers in `backend/routes/*` that delegate to controller functions and, where applicable, require `verifyToken` authentication.
  - Data for tags is fetched directly from Sequelize `Tag` model in `backend/routes/tags.js` and prepared by `appendTagList` helper.
  - Articles’ nested resources (favorites, comments) are composed by sub‑routers under `backend/routes/articles/*`.
- React frontend
  - `frontend/src/routes/Home.jsx` is the home route shell that conditionally renders a banner for guests, sets up `FeedProvider`, and displays feed tabs via `FeedToggler` with nested content in an `Outlet`.
  - `Navbar` reflects auth state via `useAuth`, toggling links and account menu.
  - `CommentList` fetches and renders comments for the current article slug, supports authenticated deletion with confirmation, and formats dates using `dateFormatter`.

Flow overview
- Requests enter Express under their respective routers (e.g., `articles.js`, `users.js`). Most article/profile/comment routes pass through `verifyToken` before invoking controllers. Tags are public and stream `res.json({ tags })` directly.
- On the client, auth state from `useAuth` gates UI (banner, feed tabs, nav items) and provides headers for comment actions. The `FeedProvider` context makes tab/tag state available to `FeedToggler` and nested routes.

## backend/routes/articles.js

Responsibility
- Declares the article collection and item routes, applies authentication to all of them, and mounts nested favorites and comments routers.

Routes
- GET `/` — requires `verifyToken`, handled by `allArticles`. Comment in source says “All Articles - by Author/by Tag/Favorited by user,” implying support for filters is implemented in the controller.
- POST `/` — requires `verifyToken`, handled by `createArticle`.
- GET `/feed` — requires `verifyToken`, handled by `articlesFeed`.
- GET `/:slug` — requires `verifyToken`, handled by `singleArticle`.
- PUT `/:slug` — requires `verifyToken`, handled by `updateArticle`.
- DELETE `/:slug` — requires `verifyToken`, handled by `deleteArticle`.

Nested routers
- `router.use("/", require("./articles/favorites"))` — mounts favorites sub‑routes at the same base.
- `router.use("/", require("./articles/comments"))` — mounts comment sub‑routes for articles (see below).

Auth and side effects
- Every route in this file is protected by `verifyToken` from `../middleware/authentication`. Any request without valid auth should be rejected by the middleware before controllers run.

Change impact
- Altering base paths or removing `verifyToken` will cascade across favorites/comments since they are mounted at `/`. Keep slug param names stable to avoid breaking sub‑routers and controllers.

## backend/routes/articles/comments.js

Responsibility
- Declares nested comment routes for a specific article identified by `:slug`.

Routes
- GET `/:slug/comments` — requires `verifyToken`, handled by `allComments`.
- POST `/:slug/comments` — requires `verifyToken`, handled by `createComment`.
- DELETE `/:slug/comments/:commentId` — requires `verifyToken`, handled by `deleteComment`.

Inputs and identifiers
- Path params: `slug` for the article, `commentId` for deletion.
- No request body schema is enforced here; bodies/queries are interpreted by controllers.

Responses and errors
- Responses are determined in `../../controllers/comments`; this router ensures auth and routing. Any 401/403 outcomes originate from `verifyToken` or controller authorization checks.

Change impact
- Renaming `:slug` or `:commentId` or moving these under a different mount path requires updating client services that call these endpoints (e.g., `getComments`, `deleteComment` on the frontend).

## backend/routes/users.js

Responsibility
- Public authentication endpoints for registration and login.

Routes
- POST `/` — handled by `signUp`.
- POST `/login` — handled by `signIn`.

Auth
- No `verifyToken` on either route; these are public entry points to obtain credentials.

Change impact
- Changing the base or method will break any client login/register flows expecting these exact paths.

## backend/routes/user.js

Responsibility
- Authenticated current user endpoint and profile update.

Routes
- GET `/` — requires `verifyToken`, handled by `currentUser`.
- PUT `/` — requires `verifyToken`, handled by `updateUser`.

Contracts
- Expects valid auth header processed by `../middleware/authentication`. The request body for updates is interpreted by `../controllers/user`.

Change impact
- Removing auth would expose PII updates publicly; keep `verifyToken` in place.

## backend/routes/profiles.js

Responsibility
- Fetch a profile by username and toggle follow/unfollow.

Routes
- GET `/:username` — requires `verifyToken`, handled by `getProfile`.
- POST `/:username/follow` — requires `verifyToken`, handled by `followToggler`.
- DELETE `/:username/follow` — requires `verifyToken`, handled by `followToggler`.

Behavior notes
- Both follow and unfollow delegate to the same `followToggler` controller; the HTTP method distinguishes action semantics.

Change impact
- Be careful changing the param name `:username` or removing `verifyToken`. Any consumer reading follow state or mutating it depends on these stable routes.

## backend/routes/tags.js

Responsibility
- Public endpoint to enumerate all tags directly via the ORM.

Route
- GET `/` — async handler:
  - Loads all tags with `const tagList = await Tag.findAll()` using `Tag` from `../models`.
  - Uses `appendTagList(tagList)` from `../helper/helpers` to build the response array.
  - Responds `res.json({ tags })`.
  - Errors are delegated to Express error middleware via `next(error)`.

Contracts
- No authentication. Output shape is exactly `{ tags }` (property named `tags`).

Change impact
- Any change to the property name or the use of `appendTagList` will break clients expecting the RealWorld spec tag list.

## frontend/src/routes/Home.jsx

Responsibility
- Home route shell that manages the banner for guests, sets up the feed context, and renders feed tabs with nested route content.

Key behavior
- Reads `{ isAuth }` from `useAuth()` to decide whether to show the banner.
- When not authenticated, renders a `BannerContainer` with:
  - `<h1 className="logo-font">conduit</h1>`
  - `<p>A place to share your knowledge.</p>`
- Always renders a `ContainerRow type="page"` that wraps:
  - `FeedProvider` providing context for feed tabs and tag filters.
  - Left column `<div className="col-md-9">` containing `FeedToggler` and a nested `Outlet` (for child routes to render the current feed list).
  - `PopularTags` alongside the feed (sidebar).

Change impact
- Removing `FeedProvider` will break `FeedToggler` and any nested components reliant on `useFeedContext`.
- Changing the column structure can affect layout assumptions of nested routes.

## frontend/src/components/Navbar/Navbar.jsx

Responsibility
- Top navigation bar that reflects authentication state and provides links to core areas.

Behavior
- Always renders a brand link `<Link className="navbar-brand" to="/">conduit</Link>` and a `SourceCodeLink` with `left` prop.
- Renders `NavItem` entries under `<ul className="nav navbar-nav pull-xs-right">`:
  - Always: `Home` (`icon="ion-compose"`, `url="/"`).
  - If authenticated (`isAuth` true):
    - `New Article` (`icon="ion-compose"`, `url="/editor"`).
    - `DropdownMenu` (account actions UI).
  - If not authenticated: `Login` (`icon="ion-log-in"`, `url="/login"`) and `Sign up` (`url="/register").

Change impact
- Toggling the `isAuth` logic will directly affect discoverability of editor/profile routes. Keep `useAuth` contract stable.

## frontend/src/components/CommentList/CommentList.jsx

Responsibility
- Fetch and display the list of comments for the current article slug; allow the author of a comment to delete it when authenticated.

Inputs
- Props: `triggerUpdate`, `updateComments` — external signals/callbacks to refetch or update after delete.
- Context: from `useAuth()` — `headers`, `isAuth`, `loggedUser`.
- Router: `slug` from `useParams()`.

Data loading
- `useEffect` with `[slug, triggerUpdate]` dependency:
  - Calls `getComments({ slug }).then(setComments).catch(console.error)`.

Rendering
- If `comments?.length > 0`, maps each comment object destructured as `{ author, author: { username }, body, createdAt, id }` to a card:
  - Body text in `.card-text`.
  - Footer renders `<CommentAuthor {...author} />` and date as `dateFormatter(createdAt)`.
  - Shows a delete button `<i className="ion-trash-a"></i>` only when `isAuth && loggedUser.username === username`.
- Else renders `There are no comments yet...`.

Delete flow
- `handleClick(commentId)`:
  - If not authenticated, alerts `"You need to login first"`.
  - Prompts `window.confirm("Want to delete the comment?")`; aborts if canceled.
  - Invokes `deleteComment({ commentId, headers, slug })`
    - On success, calls `updateComments`.
    - Errors are logged to console.

Change impact
- The component assumes comment objects include `author`, `author.username`, `body`, `createdAt`, `id`. Changing API shape will break rendering and authorization check.
- `deleteComment` requires `headers` (likely including auth token) and `slug`; do not remove these from `useAuth` or route params.

## frontend/src/components/FeedToggler/FeedToggler.jsx

Responsibility
- Render the set of feed navigation tabs based on auth and current feed context.

Behavior
- Reads `isAuth` from `useAuth()` and `{ tabName, tagName }` from `useFeedContext()`.
- Renders within `<div className="feed-toggle">` and `<ul className="nav nav-pills outline-active">`:
  - If authenticated: `<FeedNavLink name="feed" text="Your Feed" />`.
  - Always: `<FeedNavLink name="global" text="Global Feed" />`.
  - If current tab is a tag: `<FeedNavLink icon name="tag" text={tagName} />`.

Change impact
- Removing `FeedProvider` upstream breaks `useFeedContext` here. Keep `name` values stable if downstream logic relies on them for fetching.

## frontend/src/helpers/dateFormatter.test.js

Responsibility
- Unit test specifying expected behavior of `dateFormatter` helper.

Behavior under test
- Given `ISOString = "2020-01-01T12:11:08.212Z"`, `dateFormatter(ISOString)` must equal `"January 1, 2020"`.

Change impact
- Any change to date formatting must preserve this output or update tests and all dependent UI (`CommentList` uses `dateFormatter(createdAt)`).

## backend/helper/helpers.test.js

Note
- This path is referenced in the request but not included among the provided file contents. Other code in this area (e.g., `backend/routes/tags.js`) depends on `appendTagList` from `../helper/helpers`. Tests in this directory are intended to validate helper behavior used by routers like Tags. If you change `appendTagList`, ensure corresponding tests in `backend/helper/helpers.test.js` cover the output consumed by `tags.js`.

## Gotchas

- Authentication middleware coverage
  - All article, profile, comment, and current user routes are protected by `verifyToken`. If you add new routes in these areas, remember to apply `verifyToken` consistently to avoid exposing private data.
- Nested router mount points
  - `backend/routes/articles.js` mounts both favorites and comments routers at `/`. Changing this mount point or the `:slug` parameter name will break the composed paths like `/:slug/comments`.
- Response shape for tags
  - `backend/routes/tags.js` responds with `res.json({ tags })`. Frontend components and RealWorld clients expect the `tags` key specifically.
- Comment deletion authorization
  - `CommentList` authorizes delete locally by comparing `loggedUser.username` to the comment author’s `username`. If the API changes to allow moderators/admins to delete others’ comments, update the UI logic accordingly to avoid hiding valid actions.
- Date formatting contract
  - `dateFormatter` must output `January 1, 2020` for the given ISO string to satisfy tests and UI expectations. Locale changes or library swaps can cause subtle failures.