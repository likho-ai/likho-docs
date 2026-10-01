- The web app is one shell that loads small apps at run time. Each app has its own repository, CI and release. Screens and tokens: [[UI Design]]; images: `images/screens/`.
- ## 1. Repositories
	- | Repository | Owns | Exposes | Wave |
	  | --- | --- | --- | --- |
	  | `likho-web-shell` | Navigation, login, theme, Home, error boundaries, the remote loader | — (host) | 1 |
	  | `likho-mfe-library` | Recordings, upload queue, Jobs | `./routes` | 1 |
	  | `likho-mfe-transcript` | Transcript player, live lines, corrections, Search | `./routes`, `./TranscriptPanel` (embeddable) | 1 |
	  | `likho-mfe-vocabulary` | Vocabulary, Models, language policy | `./routes` | 2 |
	  | `likho-mfe-insights` | Dashboard, call summaries and quality checks | `./routes` | 2–3 |
	  | `likho-mfe-admin` | Settings, users and roles, API keys | `./routes` | 2 |
	  | `likho-ui` | Design system package `@likho/ui`: tokens, components, icons, mascot | npm package | 1 |
	  | `likho-web-sdk` | `@likho/web-sdk`: GraphQL client with hooks generated from likho-api's `schema.graphql` (GraphQL Code Generator + TanStack Query), session and permissions hooks, live-lines helper, event channel | npm package | 1 |
	- With one developer, Wave 1 builds the shell and two apps; the other three are split out when their screens are built. The split follows service ownership, so a backend change and its screen ship together.
- ## 2. How it is wired
	- **Module Federation** with Vite (`@module-federation/vite`). Each app builds a `remoteEntry.js` and is served by its own NGINX container under `/mfe/<name>/`.
	- **Manifest.** The shell fetches `/mfe/manifest.json` at start (`{ "library": "/mfe/library/remoteEntry.js", … }`). Changing one line releases or rolls back one app. The manifest is a file in `likho-infra` per environment.
	- **Shared singletons:** `react`, `react-dom`, `react-router`, `@tanstack/react-query`, `@likho/ui`, `@likho/web-sdk` (`singleton: true`, version ranges pinned by Renovate).
	- **Routing.** The shell owns the router and mounts each app's `routes` under its prefix: `/recordings`, `/jobs` → library; `/recordings/:id`, `/search` → transcript; `/vocabulary`, `/models` → vocabulary; `/dashboard` → insights; `/settings` → admin. The URL is the contract between apps.
	- **Events between apps:** a typed channel in `@likho/web-sdk` (`emit('upload.finished', { recordingId })`); used sparingly.
	- **State.** No shared store. Session, workspace, permissions and theme come from the shell through `@likho/web-sdk` context; server data through TanStack Query per app.
	- **Styling.** Tailwind v4 in every app with the `@likho/ui` preset; tokens are CSS variables defined once by the shell, so light and dark switch everywhere at once. No app defines global CSS.
	- **Failure.** Each remote is mounted inside an error boundary with a retry; a missing app shows a message in its area and the rest keeps working.
- ## 3. Embedding in other apps
	- `likho-mfe-transcript` exposes `TranscriptPanel` (`{ dialerCrtObjectId | recordingId, leg?, theme? }`). The company's reports portal (React 19 + Vite) loads it as a federated remote beside its `RecordingButton`, authenticating with an API key exchanged by its backend. This is the main practical reason for micro-frontends here: the same panel serves two applications.
- ## 4. Standard app layout
	- ```
	  likho-mfe-<name>/
	  ├── vite.config.ts            # federation: name, exposes, shared
	  ├── src/
	  │   ├── routes.tsx            # exported route objects
	  │   ├── pages/  components/  features/  hooks/
	  │   └── dev/main.tsx          # standalone dev shell with mocked session
	  ├── tests/                    # Vitest + Testing Library; Playwright smoke in the shell repo
	  ├── Dockerfile                # build → nginx serving /mfe/<name>/
	  └── .github/workflows/ci.yml  # lint, tsc, test, build, publish image
	  ```
	- Local work: `pnpm dev` in one app starts it standalone with a mocked session, or `pnpm dev:shell` loads it into the shell while the other apps come from the Compose stack.
- ## 5. Costs and guards
	- Seven repositories to keep on the same React and `@likho/ui` versions: Renovate group updates and a nightly "all apps against latest shell" Playwright run in `likho-web-shell`.
	- A slower first load than one bundle: shared singletons, route-level loading and `modulepreload` for the app behind the current URL.
	- The fallback if this proves too heavy: the same code as packages in one Vite app. The folder layout and the URL contract stay identical, so the change is mechanical.
