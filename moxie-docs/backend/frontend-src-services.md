# Backend API Surface and Contracts

This document defines the API contracts implied by the repository structure and intended frontend usage, so that maintainers can evolve endpoints safely and frontend developers can rely on stable shapes. It focuses on the service calls referenced by the frontend (`getArticle(s)`, `setArticle`, `deleteArticle`, `getComments`, `postComment`, `deleteComment`, `getProfile`).

## Architecture

- Entry points
  - Frontend service layer in `frontend/src/services` (e.g., modules that call the backend for articles, comments, and profiles). These functions encapsulate fetch logic and are the only callers of backend endpoints from the UI.
  - Backend HTTP API in `backend` (routes/controllers/handlers that serve JSON over HTTP). The exact framework and file layout are not visible in this snapshot, but the contracts below define the expected behaviors the frontend depends on.
- Flow
  - UI components call service functions → service functions make HTTP requests to backend routes → backend authenticates/authorizes (where required), performs CRUD on articles/comments/user profiles, and returns JSON responses and error codes.
- Shared concerns
  - Auth: endpoints that mutate data require authentication; read operations are largely public except where noted.
  - Error model: JSON error responses include an HTTP status code and a machine-readable error string; see each route for specifics.

## backend: Articles API

The frontend references three article-related service calls: `getArticle(s)`, `setArticle`, and `deleteArticle`. These imply the following REST endpoints and contracts.

### Route: GET /articles
- Purpose: List articles with optional filtering/pagination.
- Auth: Not required for public content.
- Query params
  - `page` (number, optional): 1-based page index.
  - `limit` (number, optional): page size.
  - Optional filters (if supported by backend): e.g., `author`, `tag`.
- Responses
  - 200 OK: `{ items: Article[], total: number, page: number, limit: number }`
  - 400 Bad Request: `{ error: "invalid_query", message: string }`
  - 500 Internal Server Error: `{ error: "internal", message: string }`

### Route: GET /articles/:id
- Purpose: Fetch a single article by ID.
- Auth: Not required for public content.
- Path params: `id` (string)
- Responses
  - 200 OK: `Article`
  - 404 Not Found: `{ error: "not_found", message: string }`
  - 500 Internal Server Error: `{ error: "internal", message: string }`

### Route: POST /articles
- Purpose: Create or update an article (maps to `setArticle`). If creating vs updating are separate in the backend, the service may call POST for create and PUT/PATCH for update; see Gotchas.
- Auth: Required. Caller must be an authenticated user with permission to create or edit the target article.
- Body (JSON)
  - For create: `{ title: string, body: string, tags?: string[] }`
  - For update: `{ id: string, title?: string, body?: string, tags?: string[] }`
- Responses
  - 201 Created (create): `Article`
  - 200 OK (update): `Article`
  - 400 Bad Request: `{ error: "invalid_input", message: string, details?: Record<string,string> }`
  - 401 Unauthorized: `{ error: "unauthorized", message: string }`
  - 403 Forbidden: `{ error: "forbidden", message: string }`
  - 404 Not Found (update target missing): `{ error: "not_found", message: string }`
  - 409 Conflict (unique constraint/title slug conflict): `{ error: "conflict", message: string }`
  - 500 Internal Server Error: `{ error: "internal", message: string }`

### Route: DELETE /articles/:id
- Purpose: Delete an article by ID (maps to `deleteArticle`).
- Auth: Required. Caller must own the article or have delete permission.
- Path params: `id` (string)
- Responses
  - 204 No Content
  - 401 Unauthorized: `{ error: "unauthorized", message: string }`
  - 403 Forbidden: `{ error: "forbidden", message: string }`
  - 404 Not Found: `{ error: "not_found", message: string }`
  - 409 Conflict (cannot delete due to constraints): `{ error: "conflict", message: string }`
  - 500 Internal Server Error: `{ error: "internal", message: string }`

## backend: Comments API

The frontend references three comment-related service calls: `getComments`, `postComment`, `deleteComment`.

### Route: GET /articles/:id/comments
- Purpose: List comments for a given article (maps to `getComments`).
- Auth: Not required for public content.
- Path params: `id` (string)
- Query params (optional): `page`, `limit`
- Responses
  - 200 OK: `{ items: Comment[], total: number, page: number, limit: number }`
  - 404 Not Found (article missing): `{ error: "not_found", message: string }`
  - 500 Internal Server Error: `{ error: "internal", message: string }`

### Route: POST /articles/:id/comments
- Purpose: Add a comment to an article (maps to `postComment`).
- Auth: Required.
- Path params: `id` (string)
- Body (JSON): `{ body: string }`
- Responses
  - 201 Created: `Comment`
  - 400 Bad Request: `{ error: "invalid_input", message: string }`
  - 401 Unauthorized: `{ error: "unauthorized", message: string }`
  - 404 Not Found (article missing): `{ error: "not_found", message: string }`
  - 409 Conflict (duplicate submission, if deduplication enabled): `{ error: "conflict", message: string }`
  - 500 Internal Server Error: `{ error: "internal", message: string }`

### Route: DELETE /articles/:articleId/comments/:commentId
- Purpose: Remove a comment (maps to `deleteComment`).
- Auth: Required. Caller must be the comment author or have moderation rights.
- Path params: `articleId` (string), `commentId` (string)
- Responses
  - 204 No Content
  - 401 Unauthorized: `{ error: "unauthorized", message: string }`
  - 403 Forbidden: `{ error: "forbidden", message: string }`
  - 404 Not Found (comment or article): `{ error: "not_found", message: string }`
  - 500 Internal Server Error: `{ error: "internal", message: string }`

## backend: Profile API

The frontend references one profile-related service call: `getProfile`.

### Route: GET /profile or GET /users/:id
- Purpose: Fetch the current user’s profile (if session-based) or another user by ID.
- Auth
  - If route is `/profile`: Required; uses the current session/auth token.
  - If route is `/users/:id`: Optional for public profile data; required for private fields.
- Path params: `id` (string) when using `/users/:id`.
- Responses
  - 200 OK: `Profile`
  - 401 Unauthorized (for `/profile` without auth): `{ error: "unauthorized", message: string }`
  - 404 Not Found: `{ error: "not_found", message: string }`
  - 500 Internal Server Error: `{ error: "internal", message: string }`

## frontend/src/services

The frontend service layer consolidates all client calls to the backend. While the specific file names are not listed in this snapshot, expect modules such as:
- `articles.ts`: exports `getArticles`, `getArticle`, `setArticle`, `deleteArticle`.
- `comments.ts`: exports `getComments`, `postComment`, `deleteComment`.
- `profile.ts`: exports `getProfile`.

Common behaviors and expectations:
- All functions return JSON-decoded data or throw on non-2xx responses. Callers should handle thrown errors that include the HTTP status and server-provided `{ error, message }` payload.
- Authenticated requests attach credentials (cookie or `Authorization: Bearer <token>`), depending on app-wide auth strategy.
- Pagination helpers normalize `{ items, total, page, limit }` responses.

## Data models

- Article
  - Example: `{ id: string, title: string, body: string, tags: string[], authorId: string, createdAt: string, updatedAt: string, slug?: string }`
- Comment
  - Example: `{ id: string, articleId: string, authorId: string, body: string, createdAt: string, updatedAt?: string }`
- Profile
  - Example: `{ id: string, username: string, displayName?: string, avatarUrl?: string, bio?: string, createdAt: string }`

These shapes reflect typical usage the frontend will need; the exact fields should be verified against the backend implementation before making breaking changes.

## Gotchas

- Creating vs updating articles: If the backend splits create (POST /articles) and update (PUT or PATCH /articles/:id), ensure the `setArticle` service dispatches to the correct method/path based on presence of `id`. Changing this behavior is a breaking change for the frontend.
- Error shapes: Keep the `{ error: string, message: string }` structure stable. Frontend error handling often keys on `error` and HTTP status; adding nested wrappers will break consumers.
- Auth propagation: If you change from cookie-based to bearer-token auth (or vice versa), update all service calls and CORS settings consistently. Mixed strategies can cause elusive 401/403s in production.
- Pagination defaults: Changing default `limit` or response envelope keys (`items`, `total`, `page`, `limit`) will break lists and infinite scroll components. Introduce versioned endpoints or negotiate via query params.
- Deletion responses: The frontend may rely on 204 No Content for optimistic UI updates. Returning 200 with a body can cause unexpected JSON parsing attempts; keep 204 for deletes.
- Slugs vs IDs: If articles are addressed by `slug` in URLs but `id` in APIs, ensure the backend unambiguously resolves one or the other and the services pass the correct identifier.
- Comment moderation: Tighten authorization carefully. Changing delete rules (e.g., from author-only to moderator-only) without frontend updates will expose 403s not currently handled.