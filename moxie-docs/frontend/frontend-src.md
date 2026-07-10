# frontend/src: Feed orchestration, article loading, and helpers

This area provides the feed tab state (global/feed/tag) via context, a hook to fetch articles with auth awareness, and small helpers for date formatting and error normalization. It exists to centralize feed UI state and common utilities used across pages/components.

## Architecture

- Entry points
  - `frontend/src/context/FeedContext.jsx` exports `FeedProvider` (default) and `useFeedContext()` to supply and consume current feed tab and selected tag.
  - `frontend/src/hooks/useArticles.js` exports the `useArticles` hook that loads articles based on feed context, location, and optional username, with auth-aware behavior.
- Helpers
  - `frontend/src/helpers/dateFormatter.js` formats ISO-like date strings to a readable "Month Day, Year" using `toLocaleDateString` with the `en` locale.
  - `frontend/src/helpers/errorHandler.js` normalizes HTTP errors (expects Axios-like `error.response`), logs, and throws a user-facing string for certain statuses.
- Control/data flow
  - `FeedProvider` initializes `tabName` to `"feed"` when authenticated via `useAuth()`; otherwise `"global"`. It exposes `changeTab(e, tabName)` which also extracts a `tagName` from the event target text.
  - Consumers use `useFeedContext()` to read `tabName` and `tagName`, then pass them into `useArticles({ location, tabName, tagName, username })`.
  - `useArticles` gets auth `headers` via `useAuth()` and calls `getArticles({ headers, location, tabName, tagName, username })` to populate `{ articles, articlesCount }` and a `loading` flag.

## frontend/src/context/FeedContext.jsx

Responsibility
- Provides feed UI state (which tab is active and which tag is selected) and a handler to switch tabs/tags. It reacts to authentication state changes and updates the default tab accordingly.

Key behavior and state
- Imports `useAuth` from `./AuthContext` and reads `{ isAuth }`.
- Internal state: `[{ tabName, tagName }, setTab]` initialized to `{ tabName: isAuth ? "feed" : "global", tagName: "" }`.
- `useEffect([isAuth])` keeps `tabName` synced with auth: sets `tabName` to `"feed"` when authenticated, `"global"` otherwise, preserving the current `tagName`.
- `changeTab = async (e, tabName) => { const tagName = e.target.innerText.trim(); setTab({ tabName, tagName }); }`
  - Note: derives `tagName` from the event target’s inner text; consumers must ensure the clicked element’s text is the intended tag.
- Context value: `{ changeTab, tabName, tagName }`.

Usage and entry points
- Wrap feed pages/components with `<FeedProvider>` to access the context via `useFeedContext()`.
- Consumers read `tabName` to decide between personalized feed (`"feed"`) or global, and read `tagName` for tag-specific queries.

Implications
- Switching authentication state will automatically switch `tabName` between `"feed"` and `"global"` without clearing `tagName`.
- `changeTab` is async but does not await anything; it synchronously sets state.

## frontend/src/hooks/useArticles.js

Responsibility
- React hook that fetches article lists based on current location, tab, tag, and optional username, incorporating auth headers when available, and exposes loading state and setters.

Inputs
- Destructured parameter object: `{ location, tabName, tagName, username }`.

Auth and dependencies
- Imports `useAuth` from `"../context/AuthContext"` to obtain `{ headers }` (auth headers for API calls).
- Imports `getArticles` from `"../services/getArticles"` which performs the actual data fetch (not included here; this hook defines the contract it expects to call).

Behavior
- Local state: `[{ articles, articlesCount }, setArticlesData]` initialized to `{ articles: [], articlesCount: 0 }` and `loading` initialized to `true`.
- Effect dependencies: `[headers, location, tabName, tagName, username]`.
- Early return: `if (!headers && tabName === "feed") return;` — avoids fetching the personalized feed without auth headers.
- On effect run:
  - Sets `loading` to `true`.
  - Calls `getArticles({ headers, location, tabName, tagName, username })`.
    - On success: `then(setArticlesData)` expects the promise to resolve to an object with `articles` and `articlesCount` keys.
    - On failure: logs to console via `catch(console.error)`.
    - Always: `finally(() => setLoading(false))`.

Return value
- `{ articles, articlesCount, loading, setArticlesData }`.
  - Exposes `setArticlesData` for callers that may want to optimistically update the list/count.

Contracts
- Expects `getArticles` to accept an object with keys `{ headers, location, tabName, tagName, username }` and to resolve to `{ articles, articlesCount }`.
- `headers` is expected to be falsy when unauthenticated and truthy when authenticated; when `tabName === "feed"`, absence of `headers` short-circuits the fetch.

## frontend/src/helpers/dateFormatter.js

Responsibility
- Formats a date-like value into a long US English date string.

Behavior
- Default export: `function dateFormatter(date)`.
- Returns `new Date(date).toLocaleDateString("en", { month: "long", day: "numeric", year: "numeric" })`.
- Example output: "January 2, 2024".

Notes
- Relies on the environment’s `Intl` implementation. Invalid `date` inputs will result in `Invalid Date` and may produce `"Invalid Date"` or throw depending on environment usage.

## frontend/src/helpers/errorHandler.js

Responsibility
- Normalizes and surfaces API errors (designed for Axios-like errors) by logging and throwing a concise message for specific HTTP statuses.

Behavior
- Default export: `function errorHandler(error)`.
- If `!error.response`: `console.log(error)` and return.
- Otherwise destructures `{ status, data } = error.response`.
- If `status` is one of `[401, 403, 404, 422, 500]`:
  - Logs `console.log(error.response, data.errors.body[0])`.
  - Throws `data.errors.body[0]` (a string message from the server).
- For other statuses: `console.dir(error)`.

Contracts and assumptions
- Expects server error payloads in the shape `{ errors: { body: [string, ...] } }`.
- Meant to be used in `catch` or error boundaries to surface a single error string; thrown value is not an `Error` instance but a string.

## Gotchas

- FeedContext’s tag extraction: `changeTab(e, tabName)` derives `tagName` from `e.target.innerText.trim()`. If the clicked element contains nested elements or extra whitespace, the tag may be incorrect. Prefer binding `changeTab` to the element with the tag text or refactor to pass the tag explicitly if needed.
- Auth-driven tab switching: When `isAuth` changes, `tabName` flips between `"feed"` and `"global"` but `tagName` is preserved. Components that assume tag is cleared on tab change should handle this explicitly.
- useArticles short-circuit: If `tabName === "feed"` without `headers`, the effect returns early and keeps previous `articles` state while leaving `loading` unchanged from the prior run until the next effect. Ensure callers initialize UI appropriately on auth transitions.
- getArticles dependency: The hook assumes `getArticles` resolves to an object with `articles` and `articlesCount`. Diverging shapes will break rendering silently unless validated.
- errorHandler throws strings: Consumers using `try/catch` should be prepared to handle a thrown string. If you rethrow or display errors, avoid assuming an `Error` object.
- Error payload shape: `errorHandler` will throw if `data.errors.body[0]` is missing when status is one of `[401, 403, 404, 422, 500]`. If your backend deviates from this structure, guard or adapt before calling `errorHandler`.