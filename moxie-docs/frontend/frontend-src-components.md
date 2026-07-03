# frontend/src/components: Shared UI Components

Reusable presentational and layout components used across the frontend. This area centralizes common UI patterns (auth page framing, avatars, container rows) and re-export barrels for feature components.

## Architecture

- Entry points are barrel files that re-export default components from same-named files (e.g., `ArticleMeta/index.js` → `./ArticleMeta`). This allows clean `import ArticleMeta from '.../components/ArticleMeta'` while keeping implementations colocated.
- Presentational components render markup only and depend on React Router and local assets. They do not fetch data or own business logic; parents provide props.
- Notable concrete implementations in this tree:
  - `AuthPageContainer/AuthPageContainer.jsx`: standard layout for login/register-like pages, including error display and a contextual link.
  - `Avatar/Avatar.jsx`: image component with a bundled default avatar asset fallback.
- All other files in this listing are barrels; their concrete implementations live next to them (not included in this excerpt) and are consumed via the barrels here.

## frontend/src/components/AuthPageContainer/AuthPageContainer.jsx

Responsibility
- Provides a consistent page layout for authentication-related pages (e.g., sign in/up), handling title, a secondary navigation link, optional error display, and content slot for forms.

Rendering and flow
- Wraps children inside a `div.auth-page` and a shared grid container via `ContainerRow` with `type="page"`.
- Centers content with `col-md-6 offset-md-3 col-xs-12` column classes.
- Displays the `title` in an `<h1>` and a centered `Link` to `path` with `text` for cross-navigation (e.g., “Need an account? Sign up”).
- If `error` is truthy, it renders an unordered list with class `error-messages` containing a single `<li>` of the error string.
- Renders `{children}` after the error block, intended for the actual auth form component.

Dependencies
- `react-router-dom` `Link` for internal navigation.
- `../ContainerRow` to provide a page-level container/row layout.

Props contract
- `title: string` – required page title text.
- `path: string` – route path for the contextual link target.
- `text: string` – link label to display.
- `error?: string` – when provided and truthy, displayed inside an error list.
- `children: ReactNode` – the auth form or related content to render inside.

Side effects and behavior
- Purely presentational; no data fetching, state, or side effects. Assumes CSS classes exist for styling.

Example usage pattern
- Wrap a login form inside `AuthPageContainer` and pass server/client validation errors via the `error` prop to surface them inline.

## frontend/src/components/AuthPageContainer/index.js

- Barrel re-export: `export { default } from "./AuthPageContainer";`
- Consumers import from the directory path to get `AuthPageContainer`.

## frontend/src/components/Avatar/Avatar.jsx

Responsibility
- Displays a user avatar image with graceful fallbacks for missing `alt`, `className`, or `src`.

Rendering and flow
- Renders a plain `<img>` tag.
- Computes props with fallbacks:
  - `alt` defaults to the literal string `"placeholder"` if not provided.
  - `className` defaults to an empty string when not provided.
  - `src` defaults to a bundled asset import `../../assets/smiley-cyrus.jpeg` when not provided.

Dependencies
- Local asset: `frontend/src/assets/smiley-cyrus.jpeg` (imported as `avatar`).

Props contract
- `alt?: string` – alternative text; ensure accessibility by passing a meaningful value when showing user photos.
- `className?: string` – for styling hooks; defaults to `""`.
- `src?: string` – the image URL; when falsy, falls back to the bundled placeholder.

Side effects and behavior
- No effects or state. Consumers should handle broken external URLs; the component will not retry or error-handle network failures.

## frontend/src/components/Avatar/index.js

- Barrel re-export: `export { default } from "./Avatar";`

## frontend/src/components/ContainerRow/index.js

- Barrel re-export: `export { default } from "./ContainerRow";`
- The `AuthPageContainer` depends on `ContainerRow` and passes `type="page"`. Ensure the sibling `ContainerRow` component supports this prop as used here.

## frontend/src/components/ArticleAuthorButtons/index.js

- Barrel re-export: `export { default } from "./ArticleAuthorButtons";`
- The implementation component is expected to live at `./ArticleAuthorButtons` and be the default export.

## frontend/src/components/ArticleEditorForm/index.js

- Barrel re-export: `export { default } from "./ArticleEditorForm";`

## frontend/src/components/ArticleMeta/index.js

- Barrel re-export: `export { default } from "./ArticleMeta";`

## frontend/src/components/ArticleTags/index.js

- Barrel re-export: `export { default } from "./ArticleTags";`

## frontend/src/components/ArticlesButtons/index.js

- Barrel re-export: `export { default } from "./ArticlesButtons";`

## frontend/src/components/ArticlesPagination/index.js

- Barrel re-export: `export { default } from "./ArticlesPagination";`

## frontend/src/components/ArticlesPreview/index.js

- Barrel re-export: `export { default } from "./ArticlesPreview";`

## frontend/src/components/AuthorInfo/index.js

- Barrel re-export: `export { default } from "./AuthorInfo";`

## frontend/src/components/BannerContainer/BannerContainer.jsx

- Not present in the provided source content; see its barrel below. If you modify its props contract, audit all imports of `BannerContainer` that rely on the barrel path.

## frontend/src/components/BannerContainer/index.js

- Barrel re-export: `export { default } from "./BannerContainer";`

## frontend/src/components/CommentEditor/index.js

- Barrel re-export: `export { default } from "./CommentEditor";`

## frontend/src/components/CommentList/CommentAuthor.jsx

- Not present in the provided source content; likely a presentational component for comment author details. Ensure any changes preserve how `CommentList` composes it via the barrel below.

## frontend/src/components/CommentList/index.js

- Barrel re-export: `export { default } from "./CommentList";`

## frontend/src/components/FavButton/FavButton.jsx

- Not present in the provided source content; see its barrel below. If it triggers side effects (e.g., favoriting), keep its external props API stable because it is consumed via the barrel path.

## frontend/src/components/FavButton/index.js

- Barrel re-export: `export { default } from "./FavButton";`

## Gotchas

- Barrels mask file moves: Changing the filename of an implementation without updating its sibling `index.js` will silently break imports at runtime. Keep the barrel path and the implementation filename in sync.
- AuthPageContainer error rendering: It renders a single `<li>` containing the entire `error` string. If you migrate to structured error arrays, update this component to map the list rather than passing a concatenated string.
- Avatar asset fallback: The placeholder image is bundled. If you rename or relocate `frontend/src/assets/smiley-cyrus.jpeg`, update the import or consumers will render a broken image.
- React Router dependency: `AuthPageContainer` requires `react-router-dom`’s `Link`. If you move this component into contexts without a Router, it will throw; wrap in a `Router` or abstract the link.