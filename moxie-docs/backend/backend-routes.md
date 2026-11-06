# Backend Express API Routes

This area defines the Express routers that make up the backend HTTP API. It wires URL paths to controller functions, enforces authentication where required, and composes nested routers for articles, comments, favorites, profiles, tags, and users.

## Architecture

- Entry points: Each file under `backend/routes/` exports an Express `router` that is mounted by the main server (not shown here). These define the request paths and apply middleware such as `verifyToken`.
- Auth: Most routes require `verifyToken` from `backend/middleware/authentication`. Only `backend/routes/users.js` (registration/login) and `backend/routes/tags.js` are public in this set.
- Controllers: Business logic is delegated to controller functions imported from `backend/controllers/*` (e.g., `../controllers/articles`). These handle reading request params/body, authorization, DB operations, and response formatting.
- Composition: `backend/routes/articles.js` mounts additional sub-routers for comments and favorites so that all article-related endpoints live under the same base.

High-level map of routers in this area:
- `backend/routes/articles.js`
  - GET `/` → `allArticles`
  - POST `/` → `createArticle`
  - GET `/feed` → `articlesFeed`
  - GET `/:slug` → `singleArticle`
  - PUT `/:slug` → `updateArticle`
  - DELETE `/:slug` → `deleteArticle`
  - mounts `./articles/favorites.js` and `./articles/comments.js`
- `backend/routes/articles/comments.js`
  - GET `/:slug/comments` → `allComments`
  - POST `/:slug/comments` → `createComment`
  - DELETE `/:slug/comments/:commentId` → `deleteComment`
- `backend/routes/articles/favorites.js`
  - POST `/:slug/favorite` → `favoriteToggler`
  - DELETE `/:slug/favorite` → `favoriteToggler`
- `backend/routes/profiles.js`
  - GET `/:username` → `getProfile`
  - POST `/:username/follow` → `followToggler`
  - DELETE `/:username/follow` → `followToggler`
- `backend/routes/tags.js`
  - GET `/` → list all tags
- `backend/routes/user.js`
  - GET `/` → `currentUser`
  - PUT `/` → `updateUser`
- `backend/routes/users.js`
  - POST `/` → `signUp`
  - POST `/login` → `signIn`

## backend/routes/articles.js

Responsible for article collection and single-article CRUD endpoints, plus composition of article sub-resources (comments and favorites).

- Base path: mounted by the app as the articles resource root (commonly `/api/articles`), though the exact mount path is defined elsewhere.
- Middleware: All routes use `verifyToken` from `../middleware/authentication`.
- Controllers: Imported from `../controllers/articles`.

Endpoints:
- GET `/` (auth: required) → `allArticles`
  - Supports listing all articles, and according to comments, filtering by author, tag, or favorited-by user. Query parameters are handled by the controller.
  - Responses: JSON result from controller; status codes determined by controller.
- POST `/` (auth: required) → `createArticle`
  - Creates a new article.
  - Responses: JSON created article; on validation/auth failure the controller sets appropriate error status.
- GET `/feed` (auth: required) → `articlesFeed`
  - Returns the authenticated user’s personalized feed.
- GET `/:slug` (auth: required) → `singleArticle`
  - Fetches a single article by `slug` path parameter.
  - Error cases: not found handled by controller.
- PUT `/:slug` (auth: required) → `updateArticle`
  - Updates an article by `slug`. Authorization (ownership) is enforced in the controller.
- DELETE `/:slug` (auth: required) → `deleteArticle`
  - Deletes an article by `slug`.

Sub-routers:
- Favorites routes from `./articles/favorites` mounted at `/`.
- Comments routes from `./articles/comments` mounted at `/`.

Data/control flow:
- Express parses the route and applies `verifyToken` to all incoming requests, which should attach authenticated user info to `req` or reject unauthorized requests.
- Controllers handle DB work and send JSON responses.

## backend/routes/articles/comments.js

Defines comment collection endpoints for a specific article.

- Base path: mounted under the articles base path by `backend/routes/articles.js`.
- Middleware: All routes require `verifyToken` from `../../middleware/authentication`.
- Controllers: `allComments`, `createComment`, `deleteComment` from `../../controllers/comments`.

Endpoints:
- GET `/:slug/comments` (auth: required) → `allComments`
  - Inputs: `slug` path param selects the article.
  - Responses: JSON list of comments for the article; status codes set by the controller (e.g., 200 on success, 404 if article missing).
- POST `/:slug/comments` (auth: required) → `createComment`
  - Inputs: `slug` path param; request body fields handled by controller.
  - Responses: JSON new comment; controller handles validation errors.
- DELETE `/:slug/comments/:commentId` (auth: required) → `deleteComment`
  - Inputs: `slug` path param; `commentId` path param.
  - Behavior: Deletes a specific comment; authorization enforced in the controller.

## backend/routes/articles/favorites.js

Toggles article favorite state for the authenticated user using a single controller for both actions.

- Base path: mounted under the articles base path by `backend/routes/articles.js`.
- Middleware: Requires `verifyToken` from `../../middleware/authentication`.
- Controller: `favoriteToggler` from `../../controllers/favorites`.

Endpoints:
- POST `/:slug/favorite` (auth: required) → `favoriteToggler`
  - Behavior: Marks the article identified by `slug` as favorited by the current user.
- DELETE `/:slug/favorite` (auth: required) → `favoriteToggler`
  - Behavior: Removes the favorite.

Notes:
- The `favoriteToggler` must internally branch based on the HTTP method (`POST` vs `DELETE`). Ensure it checks `req.method`.

## backend/routes/profiles.js

Profile retrieval and follow/unfollow for a given username.

- Middleware: All routes require `verifyToken` from `../middleware/authentication`.
- Controllers: `getProfile`, `followToggler` from `../controllers/profiles`.

Endpoints:
- GET `/:username` (auth: required) → `getProfile`
  - Fetches public profile data for `username` with respect to the current user (e.g., following status handled by controller).
- POST `/:username/follow` (auth: required) → `followToggler`
  - Follows the profile.
- DELETE `/:username/follow` (auth: required) → `followToggler`
  - Unfollows the profile.

Notes:
- As with favorites, the controller must branch based on method to follow/unfollow.

## backend/routes/tags.js

Public endpoint returning all tags. This route does not require authentication.

- Dependencies: `Tag` model from `../models`, helper `appendTagList` from `../helper/helpers`.
- Error handling: Uses `try/catch` and delegates to `next(error)` on failure.

Endpoint:
- GET `/` (auth: none)
  - Behavior: `Tag.findAll()` loads all tags; `appendTagList(tagList)` transforms the result; response is `res.json({ tags })`.
  - Success responses: 200 implicit with body `{ tags }`.
  - Error responses: Propagated via `next(error)` to the app-wide error handler.

## backend/routes/user.js

Authenticated current user lookup and update.

- Middleware: Requires `verifyToken` from `../middleware/authentication` for both routes.
- Controllers: `currentUser`, `updateUser` from `../controllers/user`.

Endpoints:
- GET `/` (auth: required) → `currentUser`
  - Returns the authenticated user’s details.
- PUT `/` (auth: required) → `updateUser`
  - Updates the authenticated user’s details. Input validation and error handling occur in the controller.

## backend/routes/users.js

Public authentication endpoints for registration and login.

- Controllers: `signUp`, `signIn` from `../controllers/users`.
- Middleware: No `verifyToken` applied here.

Endpoints:
- POST `/` (auth: none) → `signUp`
  - Registers a new user. Request body schema and error codes are defined in the controller.
- POST `/login` (auth: none) → `signIn`
  - Authenticates a user and returns credentials (e.g., token) as defined by the controller.

## Gotchas

- Authentication coverage: All article, comment, favorite, profile, and current-user routes are protected by `verifyToken`. Only `users` (sign up/login) and `tags` are public. Mounting order in the main app must ensure `verifyToken` isn’t globally applied in a way that breaks public routes.
- Method-toggling controllers: Both `favoriteToggler` and `followToggler` are used for POST and DELETE on the same path. They should branch on `req.method`. Changing them to separate controllers requires updating these routers.
- Path composition: `articles.js` mounts `./articles/comments` and `./articles/favorites` at `/`. If you change base paths here, you must update documentation and any clients that call `/:slug/comments` or `/:slug/favorite` under the articles base.
- Error propagation: `tags.js` explicitly calls `next(error)`; other routes rely on controllers to handle errors and send responses. Maintain consistent error semantics in controllers to avoid mixed response shapes or status codes.