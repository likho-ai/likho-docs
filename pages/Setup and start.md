- How to set up Likho on one machine and start it, from nothing to the first transcribed call. Every repository's README says more about that repository; this page is the order.
- ## 1. What you need
	- A machine with about **16 GB of memory** and 20 GB of free disk. The speech model runs on the CPU; a GPU is not needed. Windows 11 with PowerShell is what the scripts are written for; on Linux or macOS the same `bash` scripts exist beside them.
	- | Tool | Version | Used by |
	  | --- | --- | --- |
	  | Docker Desktop | current | the local stack (databases, event bus, object store, search, ClickHouse, the gateway) |
	  | Git | current | every repository |
	  | Node.js | 24 | likho-api, the connector, the web apps; `pnpm` through `npx pnpm@10.34.6` or Corepack |
	  | Python | 3.12 with [uv](https://docs.astral.sh/uv/) | likho-transcription, likho-language, likho-insights |
	  | Go | 1.27 | likho-media, likho-search, likho-analytics |
	  | FFmpeg | current, on the PATH | likho-media (waveforms, playable copies) |
	- Docker Desktop keeps its disk on `C:` by default; give it room (an image pull on a nearly full disk hangs the engine).
- ## 2. Get the code
	- Clone every repository of the organisation into **one folder**, side by side; the scripts assume this layout (`likho-infra/scripts/make-env-secrets.py` and `likho-deploy` write and read the sibling folders):
	- ```
	  D:\likho\
	    likho-infra        likho-contracts     likho-api           likho-media
	    likho-transcription likho-language     likho-search        likho-insights
	    likho-analytics    likho-connector-ameyo
	    likho-web-sdk      likho-ui            likho-web-shell
	    likho-mfe-library  likho-mfe-transcript likho-mfe-admin    likho-mfe-vocabulary  likho-mfe-insights
	    likho-deploy       likho-docs
	  ```
	- `git clone https://github.com/likho-ai/<repository>.git` for each. The contracts are consumed as released packages (a Git tag for Python and Go, a tarball attached to the GitHub release for TypeScript), so nothing has to be built from likho-contracts to run the rest.
- ## 3. Start the stack
	- In `likho-infra`, with Docker Desktop running:
	- ```powershell
	  .\stack.ps1 doctor     # Docker, ports, tools, free memory
	  .\stack.ps1 up         # PostgreSQL, MongoDB, Redis, NATS JetStream, the S3 store, Meilisearch, ClickHouse, the gateway; waits until healthy
	  .\stack.ps1 smoke      # the checks that prove it works
	  ```
	- `.\stack.ps1 obs` also starts Grafana with logs, traces and metrics (about 1 GB of memory); `.\stack.ps1 down` stops everything and keeps the data.
	- The gateway listens on **http://localhost:8080** and routes to the services and the web apps running on the host, so the browser has one origin.
- ## 4. Start the services
	- **The quick way (Windows):** in likho-infra, `.\dev.ps1` starts every service and web app below that is not running yet, each in a PowerShell window of its own (close a window to stop that part; `.\dev.ps1 status` and `.\dev.ps1 stop`). The tables below are what it runs, for starting a part by hand.
	- Each service reads its settings from `.env.development` in its folder (committed, no secrets; the defaults match the stack). Start each in its own terminal, in this order; the first two take a minute.
	- | Repository | Command | Ports (HTTP health and metrics / gRPC) |
	  | --- | --- | --- |
	  | likho-media | `go run ./cmd/likho-media` | 4010 / 5010 |
	  | likho-language | `uv sync` then `uv run likho-language` | 4030 / 5030 |
	  | likho-transcription | `uv sync` then `uv run likho-transcription` | 4020 / 5020 - the speech model (1.6 GB) downloads on the first job |
	  | likho-search | `go run ./cmd/likho-search` | 4040 / 5040 |
	  | likho-insights | `uv sync` then `uv run likho-insights` | 4050 / 5050 - analyses nothing until a model key is given, see §7 |
	  | likho-analytics | `go run ./cmd/likho-analytics` | 4070 / 5070 - takes every event the bus holds into ClickHouse |
	  | likho-api | `npx pnpm@10.34.6 install`, `npx pnpm@10.34.6 build`, `node dist/main.js` | 4000 - makes its tables and the first admin on start |
	  | likho-connector-ameyo | `npx pnpm@10.34.6 install`, `npx pnpm@10.34.6 dev` | 4060 - only with the dialer's settings, see §7 |
	- Every service answers `GET /healthz` (alive) and `GET /readyz` (its databases and the bus answer) on its HTTP port, and `GET /metrics`. The order matters only for the first start: likho-api asks likho-media for upload links and likho-transcription for transcripts.
- ## 5. Start the web app
	- The shell and the apps are separate Vite dev servers; the gateway serves them all at http://localhost:8080. In each folder: `npx pnpm@10.34.6 install` once, then `npx pnpm@10.34.6 dev`.
	- | Repository | Port | What it is |
	  | --- | --- | --- |
	  | likho-web-shell | 5173 | sign-in, navigation, home, search, settings; loads the apps below |
	  | likho-mfe-library | 5174 | the recordings |
	  | likho-mfe-transcript | 5175 | the transcript page, and the panel other systems embed |
	  | likho-mfe-admin | 5176 | people, keys, settings, the audit log |
	  | likho-mfe-vocabulary | 5177 | the glossary and the spellings |
	  | likho-mfe-insights | 5178 | the numbers and what the model says |
	- Open **http://localhost:8080** and sign in with the development admin from likho-api's `.env.development`: `admin@example.com` / `admin-password-1`. (A real installation sets `BOOTSTRAP_ADMIN_EMAIL` and `BOOTSTRAP_ADMIN_PASSWORD` in `.env.<environment>.local` instead.)
	- The apps are shared with the shell at run time (Module Federation), so after a change of `@likho-ai/web-sdk` the shell's dev server must be restarted with `npx pnpm@10.34.6 exec vite --port 5173 --strictPort --force`, else the apps see the old package.
- ## 6. The first call
	- **Upload call** on the home page or the Recordings page: any audio format. The file goes to likho-media, is found to be audio, and a job is queued; likho-transcription takes it and the lines appear on the transcript page as they are written. The first job also downloads the speech model, so it takes a few minutes; a two-minute call then takes about a minute and a half.
	- Open the call: play it, read it in Hinglish, Devanagari or both, correct a line, download `.txt` or `.srt`. **Search** finds any word of any call. **Vocabulary** holds the names the model listens for and how words are written. **Insights** shows the day's calls and the last two weeks in numbers; the home page shows yesterday.
	- A script does the same with an API key (Admin → API keys): `POST /api/v1/recordings`, `PUT` the file to the link, then `GET /api/v1/recordings/<id>/transcript`. The REST reference is at http://localhost:8080/api/docs.
- ## 7. Settings and secrets
	- Every repository reads `.env`, `.env.local`, `.env.<LIKHO_ENV>`, `.env.<LIKHO_ENV>.local`, in that order, each overriding the one before; a real environment variable wins over all. The `.env.<environment>` files are committed and hold no secrets; the `.local` files are ignored by git and hold the secrets of that environment on that machine. `python likho-infra/scripts/make-env-secrets.py staging` (or `production`) writes them for every repository with fresh passwords, leaving the public domain and the first admin's email to fill in.
	- **The model for insights.** Nothing is sent to a language model until `ANTHROPIC_API_KEY` is set in `likho-insights/.env.<environment>.local`, and the company has said yes to transcript text leaving its machines. The auditor's own form goes in `likho-insights/config/qa.local.json` (the repository ships an example).
	- **The dialer.** The connector needs the dialer's voice-log API values and, for the schedule, the reporting database, in `likho-connector-ameyo/.env.<environment>.local`; the README there lists them. Without them, calls still arrive by upload and by script.
	- **The company's zone.** Set `TZ` (likho-api) and `TIMEZONE` (likho-analytics) to the company's zone so that a dialer's bare call times and the days in the numbers are read right.
	- **Another system's page.** A portal that shows a call's transcript beside its own recording button keeps an API key on its server and exchanges it at `POST /api/v1/tokens/exchange` for a short-lived viewer token; its page embeds `http://<likho>/embed/recordings/<the dialer's id>#token=…` in an iframe.
- ## 8. Stop
	- Stop each service and dev server with Ctrl+C; `.\stack.ps1 down` in likho-infra stops the stack and keeps the data (`.\stack.ps1 reset` removes the data too).
- ## 9. In a cluster
	- `likho-deploy` runs the same thing in Kubernetes: `.\scripts\local.ps1 up` starts a local minikube cluster, makes the Secrets from the `.env.staging.local` files, installs the backing services with Helm and builds and deploys every service and app with Skaffold, at http://localhost:8080 (stop the stack's gateway first, it uses the same port). Staging and production are Helm releases of the same two charts: `.\scripts\deploy.ps1 <environment>`, with the images CI publishes to GitHub Container Registry. The settings keep their one source: each repository's `.env.<environment>`, synced into the chart by `scripts/sync-env.py`.
- ## 10. When something is off
	- `.\stack.ps1 doctor` and `.\stack.ps1 ps` say what is running and what is not; `.\stack.ps1 logs <service>` follows one container.
	- A service that cannot reach NATS at start keeps trying for two minutes (`NATS_CONNECT_TIMEOUT_SECONDS`), then stops; start the stack first.
	- A job that stays "queued" means no worker took it: likho-transcription is not running, or is still downloading the model (its log says).
	- The web app shows "could not be loaded" for an app whose dev server is not running; the shell keeps working.
	- Tests: every repository's `pnpm test`, `uv run pytest` or `go test ./...` runs against the local stack with the other services faked. Stop your own likho-transcription first: a running worker takes the tests' jobs.
