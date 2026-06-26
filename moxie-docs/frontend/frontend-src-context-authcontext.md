# AuthContext and useAuth (frontend/src/context/AuthContext.jsx)

This area centralizes authentication state for the frontend, exposing a React context and hook that components consume for auth headers, current user info, and a way to mutate auth state.

## Architecture

- Entry point: `frontend/src/context/AuthContext.jsx` defines the context provider and `useAuth` hook.
- Persisted bootstrap: initial state is sourced from `localStorage.getItem("loggedUser")` if present, otherwise falls back to an unauthenticated default.
- User sync: when `headers` is present, the provider fetches the current user via `getUser({ headers })` and merges it into state.
- Consumer access: components call `useAuth()` to get `{ headers, isAuth, loggedUser, setAuthState }`.

Data/control flow
- Render tree wraps in `AuthProvider` → initial state pulled from localStorage or default.
- If `headers` exists, an effect calls `getUser` → on success, `loggedUser` is updated; on failure, error is logged to console only.
- Components read `isAuth`, `headers`, and `loggedUser` and may call `setAuthState` to log in/out or update headers.

## frontend/src/context/AuthContext.jsx

Responsibility
- Declares the auth context, provides the `useAuth` hook, stores and updates auth state, and synchronizes `loggedUser` details from the backend when headers are set.

Key exports
- `useAuth()`: a hook returning the context value.
- Default export: `AuthProvider`, a component that must wrap any subtree needing auth state.

State shape and initialization
- Reads `const loggedIn = JSON.parse(localStorage.getItem("loggedUser"));` once at module load. If present, it seeds the provider state; otherwise uses:
  - `authState = { headers: null, isAuth: false, loggedUser: { bio: null, email: "", image: null, token: "", username: "" } }`.
- Provider state: `[{ headers, isAuth, loggedUser }, setAuthState] = useState(loggedIn || authState)`.

Headers-driven user fetch
- Effect runs whenever `headers` changes:
  - Guard: `if (!headers) return;`
  - Calls `getUser({ headers })`.
  - On success: `setAuthState((prev) => ({ ...prev, loggedUser }))` where `loggedUser` is returned by `getUser`.
  - On failure: logs the error via `console.error` and leaves state unchanged.

Context value
- The provider exposes: `{ headers, isAuth, loggedUser, setAuthState }`.
  - `headers`: initially `null` unless restored from localStorage; components are expected to set this to trigger user fetching.
  - `isAuth`: boolean flag in state; not automatically inferred from `headers` or `loggedUser` within this module.
  - `loggedUser`: object containing user profile fields; initially blank unless restored or fetched.
  - `setAuthState`: raw React state setter; consumers can set headers, flip `isAuth`, and update `loggedUser`.

External dependencies
- `getUser` from `../services/getUser`: invoked with an object `{ headers }`. The module assumes that passing `headers` is sufficient to retrieve the current user. Any token handling/validation happens outside this file.

Typical usage pattern
- Wrap your app:
  - `<AuthProvider><App /></AuthProvider>`
- In a component:
  - `const { headers, isAuth, loggedUser, setAuthState } = useAuth();`
  - On login: set `headers` (and likely `isAuth: true`) via `setAuthState(prev => ({ ...prev, headers: myHeaders, isAuth: true }))` to trigger `getUser`.
  - On logout: clear `headers`, `isAuth`, `loggedUser`, and optionally localStorage to fully reset auth state.

## Contracts and side effects

- Persistence contract
  - This file reads from `localStorage.getItem("loggedUser")` once at module import time. It does not write to localStorage. If your login/logout flows rely on persistence, you must manage `localStorage` writes/clears externally to match the state you set via `setAuthState`.

- User fetch contract
  - Setting a non-null `headers` value is the only trigger that starts a user fetch. The content/shape of `headers` is not enforced here; it must be whatever `getUser` expects. If `headers` is unset or cleared, no fetch runs.

- Error handling
  - Failures from `getUser` are swallowed after `console.error`. There is no automatic logout or `isAuth` toggle on error.

- Authorization state
  - `isAuth` is not derived automatically; you must explicitly set it when you consider the user authenticated or logged out.

## Gotchas

- Module-load snapshot of localStorage
  - `loggedIn` is read at import time, not when `AuthProvider` mounts. If you update `localStorage` before the provider first renders in the same session, but after the module has been imported, those changes will not affect the initial state unless you also call `setAuthState` to mirror them.

- No automatic persistence
  - Because the provider never writes to localStorage, state changes from `setAuthState` are ephemeral across reloads unless you handle persistence yourself. Conversely, if `localStorage` contains a stale `loggedUser`, it will be used as the initial state regardless of token validity.

- `isAuth` is decoupled from `headers` and `loggedUser`
  - You must manage this flag consistently. For example, setting `headers` without flipping `isAuth` leaves consumers seeing `isAuth === false` even though a user fetch may succeed.

- Partial state updates
  - The effect only updates `loggedUser`, preserving previous `headers` and `isAuth`. If your `getUser` response includes updated token or username, those will be reflected only within `loggedUser`. If you store tokens in `headers`, you must update them yourself.

- Headers semantics are opaque here
  - This module treats `headers` as an opaque blob handed to `getUser`. Any changes to `../services/getUser`'s expectations (e.g., header names) must be coordinated with the code that populates `headers` via `setAuthState`.

- Logout flow is manual
  - Clearing `headers` stops further user fetches but does not clear `loggedUser` or reset `isAuth` automatically. Implement a logout helper that updates state and clears related localStorage keys to avoid leaking prior user info in the UI.