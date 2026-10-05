- The web app is one shell that loads small apps at run time. Each app has its own repository, CI and release. Screens and tokens: [[UI Design]]; images: `images/screens/`.
- ## 1. Repositories
	- | Repository | Owns | Exposes | Wave |
	  | --- | --- | --- | --- |
	  | `likho-web-shell` | Navigation, login, theme, Home, error boundaries, the remote loader | — (host) | 1 |
	  | `likho-mfe-library` | Recordings, upload queue, Jobs | `./routes` | 1 |
	  | `likho-mfe-transcript` | Transcript player, live lines, corrections (`/recordings/:id`); `./TranscriptPanel` = the player and the lines, read-only, for the shell's `/embed/recordings/:ref#token=…` page that another system shows in an iframe | `./App`, `./TranscriptPanel` | 1 - built; panel 5 October 2026 |
	  | `likho-mfe-vocabulary` | The glossary and the spellings with how often each is heard, CSV in and out (`/vocabulary`) | `./App` | 2 - built |
	  | `likho-mfe-insights` | The last two weeks in numbers and charts (calls and minutes a day, the QA score a day, speed, languages, moods, agents, campaigns), then the day's calls with what the model said about each one (`/insights`) | `./App` | 2 - built |
	  | `likho-mfe-admin` | People and roles, invitations, API keys, workspace settings, the audit log (`/admin`, admins only) | `./App` | 2 - built |
	  | `likho-ui` | Design system package `@likho/ui`: tokens, components, icons, mascot | npm package | 1 |
	  | `likho-web-sdk` | `@likho/web-sdk`: GraphQL client with hooks generated from likho-api's `schema.graphql` (GraphQL Code Generator + TanStack Query), session and permissions hooks, live-lines helper, event channel | npm package | 1 |
	- With one developer, Wave 1 builds the shell and two apps; the other three are split out when their screens are built. The split follows service ownership, so a backend change and its screen ship together.
- ## 2. How it is wired
	- **Module Federation** with Vite (`@module-federation/vite`). Each app builds a `remoteEntry.js` and is served by its own NGINX container under `/mfe/<name>/`.
	- **Manifest.** The shell fetches `/mfe/manifest.json` at start (`{ "library": "/mfe/library/remoteEntry.js", … }`). Changing one line releases or rolls back one app. The manifest is a file in `likho-infra` per environment.
	- **Shared singletons:** `react`, `react-dom`, `react-router`, `@tanstack/react-query`, `@likho/ui`, `@likho/web-sdk` (`singleton: true`, version ranges pinned by Renovate).
	- **Routing.** The shell owns the router and mounts each app's `routes` under its prefix: `/recordings`, `/jobs` → library; `/recordings/:id`, `/search` → transcript; `/vocabulary` → vocabulary; `/insights` → insights; `/settings` → admin. The URL is the contract between apps.
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
- ## Status
	- **Built (Wave 1):** likho-web-sdk (typed GraphQL operations, hooks, uploads, live updates), likho-web-shell (sign-in, navigation, theme, home, vocabulary, settings, the remote loader), likho-mfe-library (`/recordings`), likho-mfe-transcript (`/recordings/:id`). The manifest is `nginx/mfe/manifest.json` in likho-infra.
	- Two things learned while building it, now part of the design: each app ships its own stylesheet (the shell cannot know an app's classes) **scoped under its root element** (`[data-mfe="…"]`), so the same utility class in two apps never fights over an element; and packages are installed from the **tarball attached to a GitHub release** (likho-ui, likho-web-sdk, the contracts), because a git sub-folder dependency did not install reliably.
	- Since: the Search page in the shell (`/search`, a hit opens the line at its moment), "From the dialer" in the library (a call by its id), and corrections in place in the transcript app (click a line's text or press E; Enter saves; the new version is shown with the line marked, the versions panel says which line a version corrected).
	- Since 5 October 2026: `likho-mfe-admin` at `/admin` (people with roles changed in place, disable and enable; invitations by email with a role, the one-time link shown to copy; API keys; auto-transcribe and the models; the audit log by kind of change). The shell has `/invite/:token` (whom the link is for, a name, a password), "Forgotten your password?" → `/forgot` → `/reset/:token`, and a personal settings page (appearance, own password). A viewer sees no upload button, no Admin, no Transcribe, no delete, no correction.
	- Since 5 October 2026 as well: `likho-mfe-vocabulary` at `/vocabulary` (the shell's page moved out): names heard most first with their counts and when, word or phrase, switched on or off; spellings with how often they were applied and the last lines as before/after examples; CSV export and import of either table. Viewers read; members and admins change.
	- Since 5 October 2026, step 6: the search page narrows by campaign, agent, disposition and the days the calls were made (selects fed by `recordingFacets`, everything in the address) and keeps a search for later as a chip (`saveSearch`); the library shows campaign and disposition, agent and call time in its own columns and narrows by campaign, agent and the days from the address.
	- Since 5 October 2026, step 7: the transcript page has an Insights card beside the lines (the model's summary, the customer's mood, the products, the score with each point's reason, the checks with the line each rests on; Analyse / Again for members; a note when the insights are from an older version of the transcript; "off" when no model is configured), kept fresh by `useRecordingLive` (`GET /events/recordings/:id`). `likho-mfe-insights` at `/insights` (port 5178, `/mfe/insights/`) lists a day's calls with mood, score and summary, the day in numbers and the same by agent and by campaign; the day, campaign and agent live in the address.
	- Since 5 October 2026, step 8: the home page shows yesterday in numbers (calls, transcribed, minutes, analysed, average score) to whoever is signed in, with nothing typed (`useAnalyticsOverview(yesterday())`); the Insights page opens with the last two weeks - six numbers, three bar charts drawn as SVG (calls a day, minutes a day, QA score a day), the languages and moods as shares, agents and campaigns as tables - narrowed by the campaign and agent in the address.
	- Since 5 October 2026, step 9: the shell's `/embed/recordings/<rec_… or the dialer's id>#token=…` page (no sign-in, no navigation; `?t=`, `?theme=`) mounts likho-mfe-transcript's `./TranscriptPanel` with the token from the address fragment through a `LikhoProvider` of its own (`token` instead of the session cookie, SDK 0.9.0). The company's reports portal embeds it in an iframe beside the recordings; its backend holds the API key and exchanges it per page open.
	- Not built yet: Renovate.
