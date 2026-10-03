- The development plan from 3 October 2026: what is built, how the data is laid out, and the steps that remain - backend, frontend and database - in the order to do them, each with what it changes and when it is done. The earlier plan ([[Introduction]], [[Services]], [[Frontend]]) stays the design; this page is the work list that follows from it.
- ## 1. Where we are
	- | Area | Built (repository, version) | What a person can do today |
	  | --- | --- | --- |
	  | Platform | likho-infra, likho-contracts v0.6.1, likho-deploy v0.1 | One command brings up the stack; the same charts run on a laptop cluster, staging and production |
	  | Backend | likho-api v0.2, likho-media v0.1, likho-transcription v0.1, likho-language v0.1, likho-search v0.1, likho-connector-ameyo v0.1 | Sign in, upload or fetch a call from the dialer, watch the lines arrive, read both layers, search every line, keep a vocabulary |
	  | Frontend | likho-ui, likho-web-sdk v0.2, likho-web-shell, likho-mfe-library, likho-mfe-transcript (v0.1) | The web app: library, transcript with player, search, vocabulary, settings, light and dark |
	  | Proof | A browser test drives the whole product through the gateway, on Compose and on Kubernetes | |
	- Open with the company (nothing below needs them to start, two steps need them to finish): the dialer's API credentials and reporting-database access, the campaigns and the daily budget, whether the CRM may be written, whether transcript text may go to a cloud model, the public domain, the licence.
- ## 2. The database architecture
	- **One database per service, never shared.** A service owns its tables and is the only one that reads or writes them; everyone else asks it (gRPC) or listens to it (events). likho-api decorates what it gets from the others with its own facts (names, dialer ids, attributes) instead of joining across databases. This is what makes each service replaceable and each database small.
	- | Service | Store | Owns today | Grows in |
	  | --- | --- | --- | --- |
	  | likho-api | PostgreSQL `likho_api` | `users`, `sessions`, `workspaces`, `workspace_members`, `api_keys`, `recordings` (with `attributes` jsonb, `source`, `external_id`), `jobs`, `imports`, `settings`, `handled_events` | step 2 (`invitations`, `audit_log`, `password_resets`), step 3 (`jobs.last_progress_at`) |
	  | likho-media | PostgreSQL `likho_media` + S3 buckets `likho-audio`, `likho-normalized`, `likho-peaks` | `media` (the work queue: status, attempts, codec, keys) | - |
	  | likho-transcription | MongoDB `likho_transcription` | `transcripts` (versioned documents with both layers, language, stats) | step 1 (`corrections`) |
	  | likho-language | PostgreSQL `likho_language` | glossary terms, spellings, decode policies, per workspace | step 5 (phrase lists) |
	  | likho-search | Meilisearch index `segments` | one document per transcript line, both layers, filters | step 6 (attributes as filters) |
	  | likho-connector-ameyo | PostgreSQL `likho_connector` | `calls` (what was fetched and the outcome), `cursors`, `handled_events` | - |
	  | likho-insights (step 7) | MongoDB `likho_insights` | `insights` per transcript: summary, products, sentiment, QA pre-fill, model, cost | new |
	  | likho-analytics (step 8) | ClickHouse | `events` (every CloudEvent) and materialised views `calls_daily`, `minutes_daily`, `language_mix`, `agent_daily` | new |
	  | likho-ml (step 10) | PostgreSQL `likho_ml` + bucket `likho-models` | `models`, `evaluations`, `training_examples`, `training_runs` | new |
	  | Shared | Redis (live fan-out, nothing persisted), NATS JetStream streams `LIKHO` (30 days), `LIKHO_LIVE` (1 day), `LIKHO_KEEP` (corrections, forever) | | |
	- **Rules that hold everywhere**
		- Ids are a prefix and a ULID (`rec_…`, `job_…`, `trn_…`, `imp_…`): they sort by time and say what they are.
		- Every consumer is idempotent: a `handled_events` table (or the event id in the document) makes a redelivered event change nothing twice.
		- Transcripts are **versioned, never edited in place**: a correction or a re-applied spelling is a new version; the search index holds the latest; `LIKHO_KEEP` holds every correction as training data.
		- Facts that vary by installation (campaign, agent, disposition, call time) are `attributes` on the recording, not columns: the connector fills them, the UI shows them, search filters by them (step 6).
		- Migrations live with the service (Drizzle in likho-api, SQL files in Go and Python services, a migrations collection in Mongo) and run at start; no shared migration tool.
		- Secrets never sit in a database row: API keys and sessions are stored hashed; the dialer's credentials are Kubernetes Secrets from `.env.<environment>.local`.
	- ```
	  browser ──▶ likho-api (PostgreSQL) ──gRPC──▶ likho-media (PostgreSQL + S3)
	                 │                    ──gRPC──▶ likho-transcription (MongoDB) ──gRPC──▶ likho-language (PostgreSQL)
	                 │                    ──gRPC──▶ likho-search (Meilisearch)
	                 │ events on NATS ◀──────────── every service; Redis fans live lines out to browsers
	  dialer ──▶ likho-connector-ameyo (PostgreSQL) ──REST──▶ likho-api
	  ```
- ## 3. The steps, in order
	- Each step is done the same way: contract first (likho-contracts), then the service with its tests against the real stack, then likho-api, then the SDK, then the screen, then the browser test, then the docs - CI green at every push. S = a day or two, M = about a week, L = two weeks or more.
	- ### Step 1 - Corrections (M)
		- **Why first:** people will fix lines; every fix is training data and must reach search.
		- **Contract:** `TranscriptionService.CorrectSegment(transcript_id, index, layer, text, user_id)` → a new transcript version; event `likho.transcript.corrected.v1` (already defined) carries before/after.
		- **Backend:** likho-transcription stores `corrections {transcript_id, version, index, layer, before, after, user_id, at}` and writes version n+1 with the line replaced (the other layer re-derived when the script layer changed); likho-api mutation `correctSegment` and `GET/POST /api/v1/recordings/:id/transcript/corrections`; likho-search already reindexes on the event.
		- **Database:** Mongo `corrections` collection (indexed by transcript and by user); `transcripts.version` chain unchanged.
		- **Frontend:** inline edit of a line in likho-mfe-transcript (click the text or press E, Enter saves, Esc cancels), a "corrected" mark, the version list shows who changed what; SDK `useCorrectSegment`.
		- **Done when:** a corrected word is found by search within a second, the old version is still readable, and the browser test corrects a line.
	- ### Step 2 - Users, roles and the admin screens (M)
		- **Backend:** likho-api: `users` list/create/disable, roles `admin` / `member` / `viewer` (viewer: read and search only), invitations by email with a one-time link (`invitations` table, token hashed, 7 days), password change and reset (`password_resets`), an `audit_log` (who did what, when, from where - sign-ins, key creation, deletions, role changes, corrections); every mutation writes to it. GraphQL and REST, with the same permission checks in one place (a guard that reads the role).
		- **Database:** `invitations {id, workspace_id, email, role, token_hash, invited_by, expires_at, accepted_at}`, `password_resets {id, user_id, token_hash, expires_at, used_at}`, `audit_log {id, workspace_id, user_id, action, subject_type, subject_id, details jsonb, ip, at}` with an index on (workspace, at).
		- **Frontend:** `likho-mfe-admin` (new app, loaded at `/admin`): people, invitations, roles, API keys, workspace settings, the audit log; the settings page moves there from the shell. Email sending through one small `mailer` module (SMTP settings in `.env.<environment>.local`).
		- **Done when:** an admin invites a person who signs in through the link, a viewer cannot delete, and every change is in the audit log.
	- ### Step 3 - Job robustness (S)
		- **Backend:** likho-api sweeps jobs: `queued` for longer than a limit with no worker, or `running` with no progress for N minutes, become `failed` with a reason and are queued once more (a `jobs.last_progress_at` column, written on every live segment). likho-api and likho-media retry the first connection to NATS instead of exiting. Metrics (`/metrics`, OpenTelemetry) on every service: jobs by state, queue age, realtime factor.
		- **Done when:** restarting NATS and the worker mid-job leaves no job "queued" forever, and Grafana shows the queue.
	- ### Step 4 - The connector against the real dialer (S, needs the company)
		- With the credentials in `.env.development.local`: `likho-connector-ameyo check`, one `import <crt_object_id>`, then a `backfill` of one day; set the campaigns and the daily budget; agree the CRM write-back and switch it on; the policy's first numbers from a week of running.
		- **Done when:** a call fetched by the schedule appears in the library with its attributes, and its transcript is in the CRM row.
	- ### Step 5 - Vocabulary as its own app, with usage (S)
		- **Backend:** likho-language: phrase lists (multi-word hotwords), how often each term was heard (from the transcripts), import/export as CSV.
		- **Frontend:** `likho-mfe-vocabulary` at `/vocabulary` (moved out of the shell): terms with counts, spellings with before/after examples from real lines, CSV import.
		- **Done when:** a term added here is heard in the next transcription and its count rises.
	- ### Step 6 - Search and library by attributes (S)
		- **Backend:** likho-search indexes the recording's attributes (campaign, agent, disposition, date) as filterable fields (likho-api passes them on completion through a `likho.recording.updated` event or the reindex call); likho-api `recordings` filter by attribute and `recordingCounts` by campaign.
		- **Frontend:** filters on the search and library pages (campaign, agent, date range), attribute columns in the library, saved searches (a `saved_searches` table in likho-api).
		- **Done when:** "every sale call of agent X last week with the word 'refund'" is one search.
	- ### Step 7 - Insights (L, needs the company's yes on sending text to a cloud model)
		- **Backend:** `likho-insights` (Python): on `likho.transcription.completed`, a summary, the products mentioned, the customer's sentiment, and the pre-filled QA checks (the ten yes/no observations and the scored points the auditors fill today - the mapping is the company's and lives in configuration, not in the public repository); the Claude API as the first model, with a local model behind the same interface for the day text may not leave. MongoDB `insights {transcript_id, recording_id, workspace_id, version, summary, products[], sentiment, qa{...}, model, tokens, cost, created_at}`; event `likho.insights.completed.v1`; likho-api `insights(recordingId)` and `POST /api/v1/recordings/:id/insights`.
		- **Frontend:** an Insights panel on the transcript page and `likho-mfe-insights` at `/insights`: the day's calls with their summaries and QA scores, by agent and campaign.
		- **Done when:** the auditor's form opens pre-filled for a transcribed call.
	- ### Step 8 - Analytics (M)
		- **Backend:** `likho-analytics` (Go): every CloudEvent into ClickHouse `events`; materialised views `calls_daily`, `minutes_daily`, `language_mix`, `agent_daily`, `insights_daily`; `AnalyticsService.GetOverview / GetTimeseries / GetBreakdown`; likho-api `analytics` queries.
		- **Frontend:** dashboards in `likho-mfe-insights`: calls and minutes a day, realtime factor, language mix, agents and campaigns, QA score trends.
		- **Done when:** the home page shows yesterday's numbers without a query being typed.
	- ### Step 9 - The transcript inside the reports portal (S)
		- **Frontend:** `TranscriptPanel` exposed by likho-mfe-transcript (`{ dialerCrtObjectId | recordingId, leg?, theme? }`); the company's reports portal loads it as a federated remote; the portal's backend exchanges its own session for a short-lived Likho token (likho-api `POST /api/v1/tokens/exchange` with an API key).
		- **Done when:** the audit screen of the reports portal shows the transcript beside the recording button.
	- ### Step 10 - Model registry and the training loop (L)
		- **Backend:** `likho-ml`: models (engine, version, languages, artifact in `likho-models`), evaluation runs on a gold set (word error rate, both layers), training examples from corrections, a fine-tuning job launcher for a GPU machine or a cloud job; likho-transcription reads the registry for the default model.
		- **Done when:** a new model is evaluated against the gold set and set as the default from the admin screen.
	- ### Step 11 - Scale and operations (M)
		- GPU worker image for likho-transcription (CUDA) and the cloud-engine toggle (off until allowed); KEDA scaling of the worker on queue depth, HPA for likho-api and the web apps; PodDisruptionBudgets and NetworkPolicies in the chart; backups of the volumes (snapshots) and a restore drill; alerts (queue age, failed jobs, disk).
	- ### Step 12 - Platform hygiene (S, needs the company's decisions)
		- Branch protection on `main`, release-please for changelogs and tags, Renovate for dependencies, CODEOWNERS, the licence, GHCR image visibility, a status page.
- ## 4. The frontend, as it grows
	- The shell stays small: sign-in, navigation, theme, home. Each group of screens is its own app, loaded at run time, released on its own: today `library` and `transcript`; next `admin` (step 2), `vocabulary` (step 5), `insights` (steps 7-8). The search page stays in the shell until it needs more than one screen.
	- Every app follows the same layout ([[Frontend]] §4): its stylesheet scoped under `[data-mfe="…"]`, shared singletons from the shell, the SDK for every call, a browser test of its main path, and the three states every screen handles (empty, loading, failed) with the mascot.
	- Across all steps: keyboard paths on every list and the player, screen-reader names on every control, a phone-width layout that keeps the player usable, and the design tokens of [[UI Design]] - no app-specific colours.
- ## 5. The order, with dependencies
	- | # | Step | Needs | Size |
	  | --- | --- | --- | --- |
	  | 1 | Corrections | - | M |
	  | 2 | Users, roles, admin app | - | M |
	  | 3 | Job robustness and metrics | - | S |
	  | 4 | The connector against the real dialer | credentials, campaigns, CRM permission | S |
	  | 5 | Vocabulary app with usage | - | S |
	  | 6 | Search and library by attributes | 4 (real attributes to filter) | S |
	  | 7 | Insights | the yes on a cloud model, or a local one | L |
	  | 8 | Analytics | 7 for QA numbers; the rest at once | M |
	  | 9 | Transcript panel in the reports portal | 2 (token exchange) | S |
	  | 10 | Model registry and training loop | 1 (corrections as data), a GPU | L |
	  | 11 | Scale and operations | a cluster with a domain | M |
	  | 12 | Platform hygiene | the licence, branch rules | S |
	- Steps 1, 2 and 3 can start now and in that order; 4 starts the day the credentials arrive; 5 and 6 fit between; 7 waits for one decision; 8 onwards follow.
