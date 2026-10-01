- Companion to the architecture overview. One section per repo: purpose, stack, folder tree, ports, environment, data it owns, gRPC methods, events. Versions are majors; exact versions are pinned when each repo is created and kept current by Renovate.
- ## Shared conventions
	- **Ports (local).** HTTP 40x0, gRPC 50x0, one decade per service.
	- | Service | HTTP | gRPC | | Infra | Port |
	  | --- | --- | --- | --- | --- | --- |
	  | likho-web-shell (Vite dev; apps on 5174–5178) | 5173 | — | | NGINX gateway | 80 |
	  | likho-api | 4000 | — | | PostgreSQL | 5432 |
	  | likho-media | 4010 | 5010 | | MongoDB | 27017 |
	  | likho-transcription | 4020 | 5020 | | Redis | 6379 |
	  | likho-language | 4030 | 5030 | | NATS (clients / monitoring) | 4222 / 8222 |
	  | likho-search | 4040 | 5040 | | Object store (SeaweedFS, S3 API) | 9000 |
	  | likho-analytics | 4050 | 5050 | | Meilisearch | 7700 |
	  | likho-insights | 4060 | 5060 | | Grafana / OTLP gRPC / OTLP HTTP | 3000 / 4317 / 4318 |
	  | likho-ml | 4070 | 5070 | | ClickHouse (W3) / Keycloak (W3) | 8123 / 8180 |
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
	- Node 24 LTS, TypeScript, **NestJS 11** on the Fastify adapter, **GraphQL** (code-first, `@nestjs/graphql`) for the web apps, REST controllers with `@nestjs/swagger` for other apps, Drizzle ORM + drizzle-kit (PostgreSQL), ioredis, `nats` (nats.js) for JetStream, `jose` for JWTs, connect-es gRPC clients from `@likho-ai/contracts`, nestjs-pino, OpenTelemetry. pnpm, Vitest, Testcontainers for integration tests. The module layout keeps the vocabulary of the existing Express apps (controllers, services, repositories, guards instead of middlewares).
	- ```
	  likho-api/
	  ├── src/
	  │   ├── main.ts                  # Nest app on Fastify: GraphQL, REST, SSE, health, Swagger
	  │   ├── app.module.ts
	  │   ├── config/                  # validated environment (ConfigModule)
	  │   ├── common/                  # guards (session, permission), filters (one error shape), interceptors, request id
	  │   ├── modules/
	  │   │   ├── auth/                # login, session cookie, lockout, API keys, internal JWT + JWKS
	  │   │   ├── users/               # users, roles, permission catalogue, audit log
	  │   │   ├── recordings/          # resolver + REST controller + service + repository
	  │   │   ├── jobs/                # selection policy, queueing, progress, cancel
	  │   │   ├── transcripts/         # reads via gRPC, corrections, exports
	  │   │   ├── search/  vocabulary/  models/  settings/  stats/
	  │   │   └── live/                # SSE + GraphQL subscription fed by Redis pub/sub
	  │   ├── events/                  # NATS JetStream publisher and consumers (media.ready, transcription.*)
	  │   ├── clients/                 # media, transcription, language, search (gRPC)
	  │   ├── db/
	  │   │   ├── schema/              # users, roles, permissions, workspaces, recordings, jobs, corrections, settings, api_keys, audit_logs
	  │   │   └── migrations/
	  │   └── telemetry.ts
	  ├── schema.graphql               # generated and committed: the web apps' contract
	  ├── openapi.json                 # generated and committed: source of the Postman collection
	  ├── test/
	  ├── drizzle.config.ts
	  └── Dockerfile
	  ```
	- **Owns (PostgreSQL `likho_api`)**
	- ```
	  users(id, email, name, status, failed_login_count, locked_until, created_at)
	  roles, permissions, role_permissions, user_roles      sessions(id hash, user_id, expires_at, revoked_at)
	  audit_logs(id, user_id, action, entity_type, entity_id, request_id, changes jsonb, created_at)
	  workspaces(id, name, created_at)      workspace_members(workspace_id, user_id, role)
	  recordings(id, workspace_id, original_name, media_id, size_bytes, duration_s, sha256,
	             source[upload|folder|api], status[uploaded|ready|queued|transcribing|done|failed],
	             latest_transcript_id, detected_language, created_by, created_at, updated_at)
	  jobs(id, recording_id, model_id, language_policy, force, status[queued|running|done|failed|cancelled],
	       progress_s, total_s, error, created_by, created_at, started_at, finished_at)
	  segment_corrections(id, transcript_id, segment_idx, layer[script|roman], before, after, user_id, created_at)
	  settings(workspace_id, key, value jsonb)         feature_flags(key, enabled, rules jsonb)
	  api_keys(id, workspace_id, name, hash, last_used_at)
	  ```
	- **Redis**: `job:{id}:events` pub/sub channel (live segments → SSE), `job:{id}:tail` list (last 200 events so a browser that connects late catches up), sessions, rate limits.
	- **Env**: `DATABASE_URL`, `REDIS_URL`, `MEDIA_GRPC_ADDR`, `TRANSCRIPTION_GRPC_ADDR`, `LANGUAGE_GRPC_ADDR`, `SEARCH_GRPC_ADDR`, `AUTH_SECRET`, `PUBLIC_ORIGIN`.
	- **Produces** `likho.transcription.requested`, `likho.transcript.corrected`. **Consumes** `likho.media.ready` (mark recording ready, auto-queue if the setting is on), `likho.transcription.segment` (→ Redis → SSE), `.completed` / `.failed` (update job + recording).
	- **GraphQL surface (what the web apps call)**: queries `recordings`, `recording`, `jobs`, `transcript`, `transcriptVersions`, `search`, `glossary`, `spellings`, `models`, `settings`, `statsOverview`, `me`; mutations `requestUpload`, `createJob`, `cancelJob`, `correctSegment`, `retransliterate`, `upsertGlossaryTerm`, `upsertSpelling`, `setDefaultModel`, `updateSettings`, `login`, `logout`; subscription `jobEvents(jobId)` for live lines. Each micro-frontend asks only for the fields its screen shows; types and hooks are generated from `schema.graphql`.
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
	- **Produces** `likho.vocabulary.updated`.
	- ---
- ## likho-search
	- Go, connect-go, nats.go, meilisearch-go.
	- ```
	  likho-search/
	  ├── cmd/search/main.go
	  └── internal/  config/  indexer/ (consume → fetch transcript via gRPC → index)  grpcapi/  meili/
	  ```
	- **Owns** Meilisearch index `segments`: one document per segment `{id, transcript_id, recording_id, workspace_id, idx, start, end, text_roman, text_script, language, recording_name, created_at}`; searchable `text_roman`, `text_script`, `recording_name`; filterable `workspace_id`, `language`, `created_at`. Typo tolerance on (helps with variable Hinglish spellings).
	- **gRPC `likho.search.v1.SearchService`**: `Search(query, filters, page) → hits with highlights + timestamps`, `Reindex(transcript_id)`, `DeleteRecording(recording_id)`.
	- **Consumes** `likho.transcription.completed`, `likho.transcript.corrected`.
	- ---
- ## Wave 3 services (outline)
	- | Service | Stack | Owns | Interface |
	  | --- | --- | --- | --- |
	  | **likho-analytics** | Go, clickhouse-go, nats.go | ClickHouse `events` (raw CloudEvents) + materialized views: `calls_daily`, `speed_daily`, `language_mix` | `AnalyticsService.GetOverview / GetTimeseries / GetLanguageMix`; consumes every subject |
	  | **likho-insights** | Python, Anthropic SDK | MongoDB `insights` (summary, products[], sentiment, qa_score, model, cost) | `InsightsService.GetInsights / Analyze`; consumes `transcription.completed`; produces `insights.completed` |
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
