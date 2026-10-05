- Companion to the architecture overview. One section per repo: purpose, stack, folder tree, ports, environment, data it owns, gRPC methods, events. Versions are majors; exact versions are pinned when each repo is created and kept current by Renovate.
- ## Shared conventions
	- **Metrics and a patient start (5 October 2026):** every service serves Prometheus text at `GET /metrics` on its HTTP port (OpenTelemetry instruments, names `likho_*`) and pushes the same over OTLP/HTTP when `OTEL_EXPORTER_OTLP_ENDPOINT` is set (locally `http://localhost:4318`, the `obs` profile with Grafana and the "Likho - jobs and services" dashboard). Every service keeps trying to reach NATS at start for `NATS_CONNECT_TIMEOUT_SECONDS` (120) instead of exiting; once connected, the clients reconnect on their own.
	- **Ports (local).** HTTP 40x0, gRPC 50x0, one decade per service.
	- | Service | HTTP | gRPC | | Infra | Port |
	  | --- | --- | --- | --- | --- | --- |
	  | likho-web-shell (Vite dev; apps on 5174–5178) | 5173 | — | | NGINX gateway | 80 |
	  | likho-api | 4000 | — | | PostgreSQL | 5432 |
	  | likho-media | 4010 | 5010 | | MongoDB | 27017 |
	  | likho-transcription | 4020 | 5020 | | Redis | 6379 |
	  | likho-language | 4030 | 5030 | | NATS (clients / monitoring) | 4222 / 8222 |
	  | likho-search | 4040 | 5040 | | Object store (SeaweedFS, S3 API) | 9000 |
	  | likho-insights | 4050 | 5050 | | Meilisearch | 7700 |
	  | likho-connector-ameyo | 4060 | - | | Grafana / OTLP gRPC / OTLP HTTP | 3000 / 4317 / 4318 |
	  | likho-analytics | 4070 | 5070 | | ClickHouse (HTTP / native) / Keycloak (W3) | 8123 / 9100 / 8180 |
	  | likho-ml | 4080 | 5080 | | | |
	- Through the NGINX gateway everything is one origin: `http://likho.localhost` → web, `/graphql` `/api` `/events` → likho-api, `/media` → likho-media.
	- **Environment variables common to every service**
	- ```
	  LIKHO_ENV=development            # development | test | production
	  LOG_LEVEL=info
	  HTTP_PORT=40x0   GRPC_PORT=50x0
	  NATS_URL=nats://localhost:4222
	  NATS_DURABLE=<service-name>      # durable consumer name
	  JWKS_URL=http://localhost:4000/.well-known/jwks.json   # verifies internal JWTs
	  OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
	  OTEL_SERVICE_NAME=<service-name>
	  ```
	- **Every service exposes** `GET /healthz` (process alive), `GET /readyz` (dependencies reachable), `GET /metrics` (Prometheus), the standard gRPC health service, JSON logs with `trace_id`, and shuts down gracefully on SIGTERM (finish the in-flight message, then acknowledge it).
	- **Events** use the CloudEvents JSON envelope; `type` is the versioned name, the NATS subject is the type without its version (`likho.media.ready`), and the CloudEvent `id` is sent as `Nats-Msg-Id` so a retried publish is not stored twice:
	- ```json
	  {"specversion": "1.0", "id": "01J…", "source": "likho-media", "type": "likho.media.ready.v1",
	   "time": "2026-10-01T07:00:00Z", "subject": "rec_01J…", "data": { … }}
	  ```
	- **IDs** are prefixed ULIDs (`rec_…`, `job_…`, `trn_…`, `usr_…`): sortable by time and readable in logs.
	- **Databases** — one PostgreSQL server locally, but a separate database and login per service (`likho_api`, `likho_media`, `likho_language`, `likho_ml`). No service reads another's tables.
	- ---
- ## likho-contracts
	- Single source of truth for every interface. Nothing is hand-written twice.
	- ```
	  likho-contracts/
	  ├── buf.yaml                     # lint + breaking-change rules
	  ├── buf.gen.yaml                 # codegen: TS (connect-es), Python (grpcio + mypy stubs), Go (connect-go)
	  ├── proto/likho/
	  │   ├── common/v1/common.proto   # Language, Script, Segment, Page, Error
	  │   ├── media/v1/media.proto
	  │   ├── transcription/v1/transcription.proto
	  │   ├── language/v1/language.proto
	  │   ├── search/v1/search.proto
	  │   ├── ml/v1/registry.proto            (W3)
	  │   ├── insights/v1/insights.proto      (W3)
	  │   └── analytics/v1/analytics.proto    (W3)
	  ├── events/                      # JSON Schema per event type + examples/
	  │   ├── likho.media.uploaded.v1.schema.json
	  │   ├── likho.media.ready.v1.schema.json
	  │   ├── likho.transcription.requested.v1.schema.json
	  │   ├── likho.transcription.segment.v1.schema.json
	  │   ├── likho.transcription.completed.v1.schema.json
	  │   ├── likho.transcription.failed.v1.schema.json
	  │   ├── likho.vocabulary.updated.v1.schema.json
	  │   └── likho.transcript.corrected.v1.schema.json
	  ├── streams.yaml                 # JetStream stream, subjects, retention, durable consumers
	  ├── gen/                         # generated, committed for Go (module path), built for TS/Python
	  │   ├── go/  ts/  python/
	  ├── packages/
	  │   ├── ts/package.json          # @likho-ai/contracts  → GitHub Packages (npm)
	  │   └── python/pyproject.toml    # likho-contracts      → installed from git tag by uv
	  ├── Makefile                     # make lint | breaking | generate
	  └── .github/workflows/ci.yml     # buf lint, buf breaking (against main), validate schemas, publish on tag
	  ```
	- Rules: packages are versioned `v1`, `v2`; a field is never removed or renumbered inside a version (`buf breaking` enforces it); events only gain optional fields.
	- ---
- ## likho-infra
	- ```
	  likho-infra/
	  ├── compose.yaml                 # profiles: infra | core | platform
	  ├── compose.override.example.yaml
	  ├── .env.example
	  ├── nginx/nginx.conf                    # routes /, /graphql, /api, /events (no buffering), /media
	  ├── postgres/init/00-databases.sql      # CREATE DATABASE/ROLE per service
	  ├── nats/streams.sh                     # creates the stream and consumers from likho-contracts/streams.yaml
	  ├── objectstore/buckets.sh                     # buckets: likho-audio, likho-normalized, likho-peaks, likho-models
	  ├── grafana/dashboards/                 # service overview, consumer lag, transcription speed
	  ├── k8s/
	  │   ├── charts/likho-service/           # one generic Helm chart reused by every service
	  │   ├── values/<service>.yaml
	  │   ├── gateway/                        # Gateway + HTTPRoutes (NGINX Gateway Fabric)
	  │   └── minikube/README.md              # profile, memory, addons
	  ├── skaffold.yaml                       # builds local repos, deploys charts to minikube
	  ├── .github/workflows/                  # REUSABLE workflows, called from each repo's ci.yml
	  │   ├── node-ci.yml  python-ci.yml  go-ci.yml
	  │   ├── docker-publish.yml              # build multi-stage image → ghcr.io/likho-ai/<repo>
	  │   └── release.yml                     # release-please
	  ├── scripts/doctor.ps1                  # checks Docker, ports, memory, tools on Windows
	  └── Makefile                            # make up | down | logs | reset | streams | doctor
	  ```
	- Profiles: `infra` = NGINX, Postgres, MongoDB, Redis, NATS, object store, Meilisearch; `obs` = otel-lgtm. `core` = infra + the five Wave 1 services. `platform` = core + ClickHouse, Keycloak and the Wave 3 services.
	- ---
- ## likho-web (now split into micro-frontends)
	- The web app is built as a shell plus small apps; repositories, wiring and the embeddable transcript panel are in [[Frontend]]. The stack and the folder vocabulary below still apply inside the shell and each app.
	- React 19, Vite, TypeScript, Tailwind CSS v4, shadcn/ui (Radix), TanStack Query + generated GraphQL hooks, React Router, motion, wavesurfer.js, lucide-react, Zod. pnpm, ESLint (flat) + Prettier, Vitest + Testing Library, Playwright (smoke).
	- ```
	  likho-web/
	  ├── index.html
	  ├── vite.config.ts               # proxy /graphql /api /events /media → the gateway
	  ├── components.json              # shadcn
	  ├── public/                      # favicon.svg, og.png, mascot/*.svg
	  └── src/
	      ├── main.tsx  App.tsx        # providers: GraphQL + query client, theme, router, toasts
	      ├── routes/
	      │   ├── home.tsx  dashboard.tsx  recordings.tsx  transcript.tsx
	      │   ├── jobs.tsx  search.tsx  vocabulary.tsx  models.tsx  settings.tsx
	      │   └── not-found.tsx
	      ├── components/
	      │   ├── ui/                  # shadcn primitives
	      │   ├── layout/              # AppShell, NavBar, MobileNav, Footer, ThemeToggle
	      │   ├── brand/               # Logo, Mascot (idle/listening/done), HeroArt, EmptyState
	      │   ├── upload/              # Dropzone, UploadQueue
	      │   ├── player/              # WaveformPlayer, TransportBar, SpeedMenu
	      │   ├── transcript/          # TranscriptView, SegmentLine, LayerToggle, LiveBadge, EditSegment
	      │   └── data/                # StatTile, StatusChip, DataTable, LanguageBadge
	      ├── features/                # one folder per domain: hooks + small view-models
	      │   ├── recordings/  transcripts/  jobs/  search/  vocabulary/  models/  settings/
	      ├── lib/                     # graphql.ts, events.ts (SSE), format.ts (clock, bytes), cn.ts
	      ├── hooks/                   # useTheme, useHotkeys, useJobEvents, usePlayer
	      └── styles/globals.css       # design tokens, light + dark
	  ```
	- Env: `VITE_API_ORIGIN` (empty in dev = same origin through the proxy). Docker image: static build served by NGINX; the same image runs behind the gateway in every environment.
	- ---
- ## likho-api
	- **A token for another system's browser (v0.10.0, 5 October 2026):** `POST /api/v1/tokens/exchange` with an API key answers a viewer token (`lt_…`, 60 to 3600 s, table `exchanged_tokens`, hashed like keys and sessions); the guard reads `Bearer lt_…` as a viewer of the key's workspace (`exchangedTokenId` on the principal, so it can neither change anything nor exchange again). `recordings(filter: { externalId })` is exact.
	- **The numbers behind the calls (v0.9.0, 5 October 2026):** `analyticsOverview(since, until, facts)`, `analyticsTimeseries(metric, bucket, since, until, facts)`, `analyticsBreakdown(by, since, until, facts, limit)` and `GET /api/v1/analytics/{overview,timeseries,breakdown}`, asked of likho-analytics (`ANALYTICS_GRPC_ADDR`) for the workspace; a window that ends before it starts or spans more than 400 days is refused by the API.
	- **Insights (v0.8.0, 5 October 2026):** `recording { insights }`, `insights(recordingId)` (null until analysed), `analyseRecording(id, force)` (members; audited `insights.requested`), `insightsStatus`; REST `GET` / `POST /api/v1/recordings/:id/insights` and `GET /api/v1/insights/status`. The insights live in likho-insights (Connect client, `INSIGHTS_GRPC_ADDR`); the API checks the recording is the workspace's. `likho.insights.completed` / `failed` are relayed to browsers as `insights` live events; `GET /events/recordings/:id` opens with an `open` event and streams one recording's `job`, `recording` and `insights` events. `CONSUMER_START` (`all` | `new`) says where a consumer group new to the bus starts; the tests use `new`.
	- **The library and the search by the facts of a call (v0.7.0, 5 October 2026):** every recording has a `call_time` (a connector's `callTime` attribute, read in the process's `TZ` when it carries no zone, else when the recording was made) and its facts go out as `likho.recording.updated` the moment it exists. `recordings(filter)` takes `campaign`, `agent`, `disposition`, `source`, `since`, `until` (on the call time); `recordingFacets(key, filter)` lists the values a fact takes with their counts (what a filter dropdown shows); `search(filter)` passes the same facts to likho-search. Saved searches (`saveSearch`, `savedSearches`, `deleteSavedSearch`; table `saved_searches` with the filter as JSON) are shared by the workspace and removed by who saved them or an admin; both changes are audited (`search.saved`, `search.deleted`).
	- **No job waits forever (v0.5):** a sweeper runs with the consumers every `JOB_SWEEP_SECONDS` (60). A job still queued after `JOB_QUEUED_MAX_MINUTES` (15) is asked for again once (`jobs.asked`), then failed as `no_worker`. A running job with no line for `JOB_STALL_MAX_MINUTES` (10) is stopped at the worker (`CancelJob`), failed as `stalled`, and a fresh job with `attempt` 2 is queued (`JOB_MAX_ATTEMPTS`), which gets its own second ask. The worker says at once that it took a job (`likho.transcription.started`, contracts v0.8.1), so a job is `running` while the model loads and the sweeper leaves it alone; a worker that starts a job already given up on (the stalled job's request, delivered once more to the worker that comes back) is told to drop it, and the worker drops a job it was told to cancel before it had it in hand. A transcript that arrives for a job given up on is kept: the job is done after all. `jobs.last_progress_at` is set when queued, asked again, started and on every line; `Job.attempt` and `Job.lastProgressAt` are in GraphQL and REST. Metrics: `likho_jobs` by status, `likho_jobs_queue_oldest_seconds`, `likho_jobs_finished_total`, `likho_job_realtime_factor`, `likho_events_handled_total`, `likho_job_sweeps_total`.
	- **Built.** Repository: https://github.com/likho-ai/likho-api
	- NestJS 12, Apollo Server 5 (GraphQL), REST with OpenAPI (`/api/docs`, `openapi.json`, a Postman collection), server-sent events, PostgreSQL (Drizzle), Redis (live updates between instances), NATS JetStream, Connect clients to the three services. oxlint, prettier, vitest (24 tests against the local stack; the other services faked in the test process).
	- ```
	  likho-api/
	  ├── src/
	  │   ├── auth/          # users, sessions (cookie), API keys, the guard
	  │   ├── recordings/    # recordings, jobs, the event consumers
	  │   ├── transcripts/   # transcripts over gRPC from likho-transcription
	  │   ├── vocabulary/    # glossary and spellings over gRPC from likho-language
	  │   ├── settings/      # workspace settings, API keys
	  │   ├── rest/          # /api/v1 for scripts and connectors
	  │   ├── live/          # Redis pub/sub and /events (SSE)
	  │   ├── bus/ clients/ db/ config/
	  │   └── tools/         # export (schema, OpenAPI, Postman), users:add
	  ├── schema.graphql  openapi.json  postman/
	  └── Dockerfile
	  ```
	- **Owns (PostgreSQL `likho_api`)**: `users`, `sessions`, `workspaces`, `workspace_members`, `api_keys`, `recordings` (status: uploading → ready | failed → queued → transcribing → done), `jobs`, `settings`, `handled_events`.
	- **GraphQL**: queries `me`, `recordings` (filter, search, pages), `recording` (with `playbackUrl`, `peaksUrl`, `jobs`, `latestTranscript`), `recordingCounts`, `jobs`, `job`, `transcript`, `transcriptVersions`, `engines`, `glossary`, `spellings`, `settings`, `apiKeys`; mutations `login`, `logout`, `requestUpload`, `deleteRecording`, `createJob`, `cancelJob`, `retransliterate`, `upsertGlossaryTerm`, `deleteGlossaryTerm`, `upsertSpelling`, `deleteSpelling`, `updateSettings`, `createApiKey`, `revokeApiKey`. The schema is committed as `schema.graphql`; the web client is generated from it.
	- **REST** (`Authorization: Bearer lk_...`): `POST /api/v1/recordings` (upload link), `GET /api/v1/recordings`, `GET /api/v1/recordings/{id}`, `/audio`, `/transcript`, `/jobs`, `DELETE`, `GET /api/v1/jobs/{id}`, `POST /api/v1/jobs/{id}/cancel`. Errors: `{"error": {"code", "message"}}`.
	- **Live**: `GET /events/jobs/{id}` streams each line (both layers) and the job's end; `GET /events/recordings` streams every change in the workspace. The gateway keeps `/events/` unbuffered.
	- **Consumes** `likho.media.ready`, `likho.media.failed`, `likho.live.segment`, `likho.transcription.completed`, `likho.transcription.failed` (durable consumers named `<group>-<event>`, applied once by event id). **Produces** `likho.transcription.requested`.
	- **Env**: `DATABASE_URL`, `REDIS_URL`, `NATS_URL`, `MEDIA_GRPC_ADDR`, `TRANSCRIPTION_GRPC_ADDR`, `LANGUAGE_GRPC_ADDR`, `PUBLIC_ORIGIN`, `SESSION_SECRET`, `BOOTSTRAP_ADMIN_EMAIL/PASSWORD` (the first admin and workspace), `CONSUMERS_ENABLED`, `CONSUMER_GROUP`.
	- Sign-in is a session cookie backed by a row (revocation is immediate); passwords are scrypt; API keys are shown once and stored hashed. Keycloak and short-lived JWTs between services remain Wave 3.
	- Built since v0.1: `search`, `requestImport` / `imports`, recording `attributes` and `source`, `likho.recording.deleted` (v0.2); `correctSegment(input)` and `corrections(recordingId)`, with `POST/GET /api/v1/recordings/:id/transcript/corrections` (v0.3); roles admin / member / viewer, invitations (`inviteUser`, `invitation(token)`, `acceptInvitation`, `revokeInvitation`), `users`, `setUserRole`, `disableUser` / `enableUser`, `changePassword`, `requestPasswordReset` / `resetPassword`, and `auditLog(filter, first, after)` (v0.4, 5 October 2026).
	- **People (v0.4):** three roles - an admin manages people, keys and settings; a member works with recordings; a viewer reads, plays and searches (a viewer's mutation is `forbidden`; API keys act as members). People join by invitation: a one-time link, seven days, token stored hashed, mailed through `SMTP_URL` when set and shown to the admin either way. Password resets go by mail only (two hours) and end every other session; disabling a person ends their sessions at once. Tables `invitations`, `password_resets`, `audit_log`.
	- **Audit log (v0.4):** every mutation a person or an API key makes writes one row - who (kind, id, name, address), what (`recording.deleted`, `user.invited`, `transcript.corrected`, ...), to what (kind and id), the details that matter and never a secret. Written after the change; a failure to write is logged, not thrown. Read with `auditLog`, newest first, by action, target or actor, page by page.
	- ---
- ## likho-media
	- **Built.** Repository: https://github.com/likho-ai/likho-media
	- Go, Connect (answers plain gRPC too), pgx (PostgreSQL), minio-go (S3), nats.go (JetStream), FFmpeg in the image. golangci-lint, `go test` (73 tests against the local stack).
	- ```
	  likho-media/
	  ├── cmd/likho-media/main.go
	  ├── internal/
	  │   ├── config/      # environment → settings
	  │   ├── links/       # signed upload and download links (HMAC)
	  │   ├── httpapi/     # PUT /media/uploads/{id}, GET /media/{id}/original|audio|peaks (Range)
	  │   ├── rpc/         # MediaService
	  │   ├── ingest/      # workers: probe, waveform, playback copy, events
	  │   ├── audio/       # ffprobe / ffmpeg, peaks
	  │   ├── store/       # PostgreSQL: the media table is also the work queue
	  │   ├── objects/     # S3
	  │   └── events/      # uploaded / ready / failed
	  └── Dockerfile
	  ```
	- **The original is never changed.** The speech model reads a recording exactly as it was uploaded. Measured on real calls: transcribing a converted 16 kHz copy changed about one word in seven, and a lossless copy was twelve times the size of a telephone MP3.
	- **Playback.** MP3, AAC, FLAC and plain WAV are played by the browser as they are, so nothing is stored twice. For telephone codecs (A-law, mu-law, GSM, AMR) an MP3 copy is made once.
	- **Owns** buckets `likho-audio` (originals), `likho-normalized` (playback copies, only where needed), `likho-peaks` (waveforms); PostgreSQL `likho_media.media`.
	- **gRPC `likho.media.v1.MediaService`**: `CreateUpload(workspace, recording, name, sha256?) → upload link`, `GetMedia(id)`, `GetDownloadUrl(id, kind[original|normalized|peaks]) → signed link`, `DeleteMedia(id)`.
	- **Links.** The object store is never reachable from outside. A link points at likho-media through the gateway, is valid for one thing and expires (uploads 1 hour, downloads 15 minutes).
	- **Env**: `DATABASE_URL`, `NATS_URL`, `S3_ENDPOINT`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `PUBLIC_URL`, `LINK_SECRET`, `MAX_UPLOAD_MB`, `WORKERS`.
	- **Produces** `likho.media.uploaded`, then `likho.media.ready` or `likho.media.failed`.
	- What a caller can rely on:
		- A file is not lost when the service dies: a worker claims the oldest waiting file, and a file whose worker died is claimed again. Several instances share the work.
		- An event is not lost when the bus is down: the result is stored first and the event is sent until the bus confirms it.
		- The same content uploaded again is reported (`duplicate_of`); a caller that knows the checksum is spared the upload.
	- Not built yet: resumable uploads for very large files, and the watched folder (the command line tool `likho-transcribe --watch` covers local folders today).
	- ---
- ## likho-transcription
	- **Built.** Repository: https://github.com/likho-ai/likho-transcription
	- Python 3.12, uv, faster-whisper / CTranslate2, grpcio, nats-py, pymongo, httpx, pydantic-settings. ruff, mypy, pytest (99 tests; the service tests run against MongoDB and NATS from likho-infra).
	- ```
	  likho-transcription/
	  ├── pyproject.toml  uv.lock
	  ├── src/
	  │   ├── likho_engine/            # the speech engine: no network, no database
	  │   │   ├── audio.py  chunking.py  cleanup.py  engine.py
	  │   │   ├── transcript.py        #   both layers of every line; asks a Language how to write them
	  │   │   ├── writer.py  files.py  watch.py
	  │   │   └── cli.py               #   the likho-transcribe command
	  │   └── likho_transcription/     # the service around it
	  │       ├── __main__.py          #   starts worker + gRPC + health
	  │       ├── settings.py
	  │       ├── consumer.py          #   likho.transcription.requested → one job at a time
	  │       ├── runner.py            #   fetch audio → detect → policy → decode → Hinglish → store
	  │       ├── gateways.py          #   gRPC to likho-language (built-in rules when it is down) and likho-media
	  │       ├── grpc_server.py       #   TranscriptionService
	  │       ├── store.py             #   MongoDB transcripts, one document per version
	  │       └── events.py            #   live line / completed / failed
	  ├── tests/                       # engine tests + service tests against the local stack
	  └── Dockerfile                   # CPU image; the model cache is a volume
	  ```
	- The Hinglish rules are not in this repository: it installs the `likho-hinglish` package from likho-language, so both services write Hinglish with the same code.
	- **Owns (MongoDB `likho_transcription.transcripts`)**: the two-layer document. Language detection block (`detected`, `probability`, `candidates[]`, `decoded_as`, `policy`), `segments[]` each with `text_script` (layer 1) and `text_roman` (layer 2), model, stats, `vocabulary_version`, `version`. A version is never changed; a re-transliteration is a new version. Indexes: `recording_id + version` (unique), `job_id` (unique).
	- **gRPC `likho.transcription.v1.TranscriptionService`**: `GetTranscript(id)`, `ListTranscripts(recording_id)`, `Transcribe(recording_id, media_id, workspace_id, model, policy) → stream` (started, each line, the stored transcript), `Retransliterate(transcript_id) → new version`, `ListEngines()`, `CancelJob(job_id)`.
	- **Env**: `MONGO_URL`, `NATS_URL`, `LANGUAGE_GRPC_ADDR`, `MEDIA_GRPC_ADDR`, `DEFAULT_MODEL=turbo`, `DEVICE=auto`, `COMPUTE_TYPE=auto`, `WORKER_ENABLED=true`, `JOB_MAX_DELIVER=3`. The audio is downloaded through a link from likho-media, so this service holds no object-store keys.
	- **Consumes** `likho.transcription.requested`. Downloads the original file through a link from likho-media. **Produces** `likho.live.segment` (each line as it is written), `likho.transcription.completed`, `likho.transcription.failed`, and `likho.dead` for a request that could not be done.
	- What a caller can rely on:
		- A job is not lost: it is acknowledged only when the transcript is stored. While it runs the worker reports "in progress"; a crashed worker's job is handed to another one.
		- A job is not done twice: a recording that already has a transcript reports that transcript unless the job says `force`, and event ids are derived from the job.
		- A service that is down means a retry (3 attempts). Audio that cannot be read fails at once.
		- When likho-language is down, lines are written with the built-in rules and the transcript is stored with `vocabulary_version` 0, to be re-transliterated later.
	- Scaling: workers share one durable consumer, so starting a second worker (another machine, a GPU box) splits the queue with no code change.
	- ---
	- **Corrections (v0.2, 5 October 2026):** `CorrectSegment(transcript, index, layer, text, user, workspace)` makes the next version with that one line as the person wrote it, keeps the change in the `corrections` collection (recording, the version looked at, the version made, index, layer, before, after, user, time - training data, never expired) and publishes `likho.transcript.corrected` with the new version's id, which likho-search reindexes. The Hinglish of a corrected script line is derived again through likho-language; a corrected Hinglish stands as written. Only the latest version can be corrected. `ListCorrections(recording)` returns them newest first.
- ## likho-language
	- Python 3.12, uv, grpcio, SQLAlchemy 2 + Alembic (PostgreSQL), nats-py. The transliteration rules move here from `transcriber/hinglish/` as an installable package (`likho-hinglish`) that likho-transcription also depends on for its offline fallback.
	- ```
	  likho-language/
	  ├── src/
	  │   ├── likho_hinglish/          # rules.py, words.py — the pure library (published)
	  │   └── likho_language/
	  │       ├── grpc_server.py       # LanguageService
	  │       ├── transliterate.py     # rules + DB spelling table + phrase replacements, cached in memory
	  │       ├── policy.py            # which language to decode as, per detected language
	  │       ├── models.py  repo.py   # glossary_terms, spellings, language_policies
	  │       ├── seed.py              # imports glossary.txt / custom_words.json
	  │       └── events.py            # vocabulary.updated
	  ├── alembic/
	  └── tests/
	  ```
	- **Owns (PostgreSQL `likho_language`)**
	- ```
	  glossary_terms(id, workspace_id, term, language, enabled, note, created_at)
	  spellings(id, workspace_id, source, target, is_phrase, enabled, created_at)      -- "त्रिफला" → "triphala"
	  language_policies(id, workspace_id, detected_language, min_probability, decode_as, target_layer)
	          -- seed: hi → hi → hinglish;  ur → hi → hinglish;  en ≥ 0.80 → en → as-is;  default → hi
	  vocabulary_versions(workspace_id, kind, version, updated_at)
	  ```
	- **gRPC `likho.language.v1.LanguageService`**: `Transliterate(text, script, workspace)`, `TransliterateBatch(segments[])`, `GetHotwords(workspace, language)`, `ResolveDecodePolicy(detected, probability, candidates[])`, glossary CRUD (`ListGlossary`, `UpsertGlossaryTerm`, `DeleteGlossaryTerm`), spellings CRUD, policy CRUD.
	- **Produces** `likho.vocabulary.updated`. **Consumes** `likho.live.segment` (since v0.3.0, 5 October 2026): every transcript line, with the recording's workspace, raises `heard` on each glossary term found in it (whole word or phrase, either layer, any case) and `applied` on each spelling whose source is in it; the last three such lines are kept per spelling as before/after examples (`spelling_examples`). Counts are lines heard: a recording transcribed twice counts twice. `ImportGlossaryTerms` and `ImportSpellings` load many entries in one transaction (one version, one event); the CSV itself is read and written by likho-api.
	- ---
- ## likho-search
	- **The facts of a call beside its lines (v0.3.0, 5 October 2026):** `likho.recording.updated` (likho-api, on creation and on every change of a recording's facts) is kept in a second index `<INDEX_NAME>_recordings` and copied onto every line of the recording - whichever arrives first, the lines or the facts - so `Search` narrows by `campaign`, `agent`, `disposition`, `source` and a window on `call_time` besides language, recording and the transcript's time. A deleted recording loses its facts too.
	- Built (v0.1, 3 October 2026): Go, Connect, meilisearch-go, nats.go. `cmd/likho-search`, `internal/{config, index, indexer, events, rpc, app}`.
	- **Owns** the Meilisearch index `segments`: one document per transcript line `{id, transcript_id, recording_id, workspace_id, idx, start, end, text_roman, text_script, language, created_at}`; `text_roman` and `text_script` searchable (typo tolerance on, since Hinglish is spelled many ways), the rest filterable. One transcript per recording is indexed: the latest. Names and the other facts about a recording are likho-api's, which decorates the hits.
	- **gRPC `likho.search.v1.SearchService`**: `Search(workspace, query, language?, recording?, since?, until?, page, page_size)` → hits with both texts, the matches wrapped in `<mark>`, timestamps; `Reindex(transcript, workspace)`; `DeleteRecording(recording)`.
	- **Consumes** `likho.transcription.completed` and `likho.transcript.corrected` (fetches the transcript from likho-transcription and replaces the recording's lines), `likho.recording.deleted`.
	- In the product: likho-api's `search` query and `GET /api/v1/search`, the shell's Search page (`/search?q=…&lang=…`), a hit opens `/recordings/:id?t=<seconds>` where the transcript app marks the line and stands the player there. HTTP 4040, gRPC 5040.
	- ---
- ## likho-connector-ameyo
	- Built (v0.1, 3 October 2026): Node 24, TypeScript, `nats`, `pg`, `mssql`. `src/{config, ameyo, dialer, likho, state, policy, importer, schedule, bus, writeback, app, cli}`.
	- **Owns** PostgreSQL `likho_connector`: `calls` (which calls were fetched, with what result), `cursors` (where the schedule is), `handled_events`.
	- **Three ways a call comes in.** By **id**: a person pastes a `crt_object_id` in the library's "From the dialer" panel (likho-api's `requestImport`, event `likho.import.requested`), or `likho-connector-ameyo import <id>` on the command line. By **schedule**: every few minutes the new calls since the cursor are read from the dialer's reporting database (read only), judged by the policy - campaigns, shortest talk time - and fetched within a daily budget (one CPU transcribes a small share of a day). By **backfill**: a window of call times, from the command line.
	- **How a call travels:** the details from the dialer's database (campaign, agent, disposition, call time, talk time, phone masked to its last digits) → the audio from the dialer's voice-log API (`downloadVoiceLog` by `crtObjectId`) → `POST /api/v1/recordings` with `source: ameyo`, `externalId` and the details as attributes, the file PUT to likho-media → the workspace's auto-transcribe (or a job when the recording was already there) → `likho.import.completed` or `likho.import.failed` with a reason and a code (`not_found`, `no_recording`, `unavailable`, `rejected`, `error`); a dialer that does not answer makes the request come back later.
	- **Write-back** (off until allowed): on `likho.transcription.completed`, the Hinglish text into the CRM (MS SQL Server) through one statement with `@externalId` and `@transcript`.
	- **Nothing of the company is in the repository.** The dialer's address and credentials, the API key, and the SQL that names tables and campaigns are `.env.<environment>.local` and `queries/*.local.sql`, both ignored by git; the repository holds the column contract and an example of each query.
	- **First real run (5 October 2026, v0.3.0):** the reporting database and both recording servers answer from the development machine; one call imported by id and three by backfill appeared in the library with their attributes and were transcribed. The live voice-log server keeps the last week or two; older recordings come from the archiver by the leg's `call_id` (`AMEYO_ARCHIVAL_URL`, a plain GET of `dacxURI=dacx://voicelog-archiver-storage-path/<call_id>`), tried when the live server has nothing; the recording's `audioFrom` attribute says `live` or `archive`. The reporting table is large (tens of millions of rows): the real queries keep every read inside a `call_time` window and order by `call_time` alone, and `check` tries the calls query only from `SCHEDULE_START`.
	- Still to agree with the company: which campaigns and how many calls a day (the policy, then the schedule switched on), and whether and where the CRM may be written.
	- ---
- ## likho-insights
	- **Built 5 October 2026 (v0.1.0, Python, Anthropic SDK behind a small `Model` interface).** On `likho.transcription.completed` (or `Analyse` over gRPC) it sends one transcript's Hinglish lines and the auditor's form to the model and keeps the answer in MongoDB `likho_insights.insights`, one document per transcript: `summary`, `intent`, `products[]`, `sentiment`, `checks[] {key, label, answer yes/no/na, evidence}`, `scores[] {key, label, score, max, reason}`, `score_total`, `score_max`, `model`, `input_tokens`, `output_tokens`, `form_version`, `transcript_cut`. The answer is read strictly against the form: a skipped check is `na`, a skipped score is 0 with a note, a score above its maximum is clamped. `GetInsights` (by recording or by transcript), `Analyse` (again with `force`; the id stays), `GetStatus` (`enabled`, `model`, `form_version`). Events `likho.insights.completed.v1` (ids, model, sentiment, score, tokens - no text) and `likho.insights.failed.v1`; a deleted recording loses its insights.
	- **The form** is a JSON file (`QA_FORM_FILE`): a version, up to 40 checks and 40 scored points with `snake_case` keys. The company's own form is `config/qa.local.json`, ignored by git; the repository ships `config/qa.example.json`.
	- **No text leaves without a key.** `ANTHROPIC_API_KEY` empty (the default) means the service starts, answers `GetStatus` with `enabled: false`, serves the insights it has and analyses nothing. What is sent with a key: the lines (time code and Hinglish words), the language, the form. Not sent: audio, attributes, names beyond the words. Transcripts longer than `MAX_TRANSCRIPT_CHARS` (24 000) are cut with a note.
	- Settings: `GRPC_PORT` 5050, `HTTP_PORT` 4050, `MONGO_URL` / `MONGO_DATABASE`, `NATS_URL`, `TRANSCRIPTION_GRPC_ADDR`, `ANTHROPIC_MODEL` (claude-sonnet-5-5), `MODEL_TIMEOUT_SECONDS`, `MODEL_MAX_OUTPUT_TOKENS`, `REPORT_LANGUAGE` (English), `CONSUMERS_ENABLED`, `CONSUMER_GROUP`, `CONSUMER_START`. Tests run the real service against MongoDB and NATS with a fake model and a stand-in likho-transcription; CI uses `python-ci.yml` with `stack-services: "mongo nats"`.
- ## likho-analytics
	- **Built 5 October 2026 (v0.1.0, Go, clickhouse-go over HTTP, nats.go).** Two durable consumers (`likho-analytics-events` on `likho.>` of LIKHO, `likho-analytics-corrections` on LIKHO_KEEP) store every CloudEvent in ClickHouse `events` (id, type, source, subject, time, workspace_id, recording_id, data; ReplacingMergeTree by id, partitioned by month, kept `RETENTION_DAYS`) and every `likho.recording.updated` / `deleted` in `recordings` (the facts as they change: source, name, call time, campaign, agent, disposition; the latest row wins). `GetOverview`, `GetTimeseries` (day or hour, a day beginning in `TIMEZONE`) and `GetBreakdown` (agent, campaign, disposition, language, sentiment, source) take one workspace, a window of call time (at most 400 days) and optional facts; they join the two tables into one row per call and group - fast enough for years of calls at this size. Views `calls_daily`, `language_mix`, `agent_daily`, `insights_daily` for ad-hoc use. Refusals: `INVALID_ARGUMENT` for a missing workspace, window, metric or dimension, hours over more than a month; `UNAVAILABLE` when ClickHouse does not answer.
	- Settings: `GRPC_PORT` 5070, `HTTP_PORT` 4070, `CLICKHOUSE_URL` (`http://user:password@host:8123/database`; the database is created when the login may), `MIGRATE_ON_START`, `RETENTION_DAYS` 400, `TIMEZONE` (else `TZ`, else UTC), `NATS_URL`, `CONSUMERS_ENABLED`, `CONSUMER_GROUP`, `CONSUMER_START` (`all` | `new`). Tests run the real service against ClickHouse (a database of its own) and NATS; CI uses `go-ci.yml` with `stack-services: "nats clickhouse"`.
- ## Wave 3 services (outline)
	- | Service | Stack | Owns | Interface |
	  | --- | --- | --- | --- |
	  | **likho-ml** | Python, SQLAlchemy, boto3 | PostgreSQL `likho_ml`: `models`, `evaluations`, `training_examples`, `training_runs`; bucket `likho-models` | `ModelRegistry.ListModels / RegisterModel / SetDefault / ListEvaluations / ExportDataset / StartTrainingRun`; consumes `transcript.corrected` |
	  | **likho-cli** | Python, SQLite | local `likho.db` | today's CLI; optional `--server` to push results to the platform |
	- Fine-tuning needs a GPU. likho-ml collects and exports the data and launches the run on a GPU machine or a cloud job; it does not train on this CPU box.
	- ---
- ## likho-docs (Logseq)
	- ```
	  likho-docs/
	  ├── logseq/config.edn            # :publishing/all-pages-public? true, default home page
	  ├── logseq/custom.css            # brand colours
	  ├── pages/
	  │   ├── Introduction.md  Architecture.md  Getting Started.md  Glossary.md
	  │   ├── services___likho-api.md  services___likho-media.md  …      (namespace pages)
	  │   ├── adr___0001 Microservices and separate repos.md  adr___0002 NATS JetStream …
	  │   └── runbooks___Local stack.md  runbooks___Reset data.md
	  ├── assets/                      # diagrams, screenshots
	  └── .github/workflows/publish.yml   # logseq/publish-spa → GitHub Pages (docs.<domain> via CNAME)
	  ```
	- Pages are plain Markdown, so they can be edited in the Logseq desktop app or any editor.
