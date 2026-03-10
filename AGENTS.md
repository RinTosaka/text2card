# Repository Guidelines

## Project Structure & Module Organization
- `src/` contains the Vue 3 application (`App.vue`, `main.js`) and styles (`style.css`, `fonts.css`).
- `public/` holds static assets served as-is.
- `dist/` is the Vite build output (generated).
- `docs/` contains project documentation artifacts.

## Build, Test, and Development Commands
- `npm run dev` runs the local Vite dev server with hot reload.
- `npm run build` builds the production bundle into `dist/`.
- `npm run preview` serves the production build locally for verification.

## Coding Style & Naming Conventions
- Indentation: 2 spaces in Vue SFCs and CSS.
- Prefer `camelCase` for JS/TS variables and functions, `kebab-case` for CSS class names.
- Keep Vue `<script setup>` logic in `App.vue` lean; extract helpers only when they are reused.
- No explicit lint/format scripts are defined; follow the existing style in `src/App.vue` and `src/style.css`.

## Testing Guidelines
- No automated test framework is configured in this repo.
- If you add tests, document the toolchain and add a script in `package.json`.

## Commit & Pull Request Guidelines
- Commit style in history follows `feat: <summary>` (e.g., `feat: 新增README展示区并接入GitHub图标入口`).
- Keep commit messages short and action-oriented.
- PRs should include a concise summary and screenshots or GIFs for UI changes.
- After every code/file modification, the agent must ask the user whether to create a commit before taking any commit action.
- When the user asks to create a commit, the agent must first review all current uncommitted changes (`git status`/`git diff`) and write a commit message that summarizes the full change set since the last commit, not only the most recent edit.

## Security & Configuration Tips
- Avoid committing large binary assets; use `public/` for small static files only.
- User-generated file uploads (fonts/images) are handled client-side; keep processing in the browser.
