- Every service keeps its own data and no service reads another's database: they talk over gRPC and events ([[Platform]]). This page lists where each kind of data lives and its main tables. The exact definitions are in each repository (migrations, models); ids are prefixed ULIDs (`rec_`, `med_`, `job_`, `trn_`, `wsp_`, `usr_`, `imp_`, `key_`), so they sort by creation time.
- ## Where data lives
	- | Service | Store | What it holds |
	  | --- | --- | --- |
	  | likho-api | PostgreSQL `likho_api` | people, sessions, workspaces, roles, API keys, recordings and their facts, jobs, imports, saved searches, settings, the audit log |
	  | likho-media | PostgreSQL `likho_media` + S3 store | the audio files (original, playable copy, waveform peaks) and what is known about each |
	  | likho-transcription | MongoDB `likho_transcription` | transcripts (every version, both layers, line by line) and the corrections people made |
	  | likho-language | PostgreSQL `likho_language` | the glossary, the spellings, the language policy, with their usage counts |
	  | likho-search | Meilisearch index `segments` | every transcript line, searchable in both layers |
	  | likho-insights | MongoDB `likho_insights` | what the model said about each call |
	  | likho-analytics | ClickHouse `likho_analytics` | every event of the platform, and the facts of each call, for the numbers |
	  | likho-connector-ameyo | PostgreSQL `likho_connector` | which dialer calls were fetched, the schedule's cursor |
	  | NATS JetStream | streams `LIKHO`, `LIKHO_LIVE`, `LIKHO_KEEP` | the events between services (30 days, 1 day, kept for ever) |
- ## likho-api (PostgreSQL)
	- | Table | One row per | Main columns |
	  | --- | --- | --- |
	  | `users` | person | email, name, password hash, role (admin, member, viewer), disabled at |
	  | `workspaces`, `workspace_members` | workspace; person in a workspace | name; role in the workspace (owner, member) |
	  | `sessions` | signed-in browser | hash of the cookie, expires at, revoked at, user agent |
	  | `invitations`, `password_resets` | invitation; reset link | email, role, hash of the token, expires at, accepted or used at |
	  | `api_keys` | key for a script or connector | name, hash of the key, last used at, revoked at |
	  | `exchanged_tokens` | short-lived viewer token for another system's page | hash, the key it came from, who it is for, expires at |
	  | `recordings` | call | original name, media id, size, duration, channels, source, external id (the dialer's id), **attributes** (campaign, agent, disposition and the dialer's other fields), call time, status, latest transcript id, detected language and probability |
	  | `jobs` | transcription | recording, model, language policy, status, progress, attempts, error, transcript id, started and finished at |
	  | `imports` | call asked for from the dialer | source, external id, status, recording id, reason when it failed |
	  | `saved_searches` | named search | name, the words, the filters |
	  | `settings` | setting of a workspace | key, value (JSON) |
	  | `audit_log` | change | who (person or key), what action, on what, details, IP, when |
	  | `handled_events` | event already acted on | event id (so an event delivered twice is acted on once) |
- ## likho-media (PostgreSQL and the S3 store)
	- `media`: one row per file: workspace, recording, original name and type, size, status, the object keys of the original, the playable copy and the waveform peaks, duration, channels, sample rate, codec.
	- Buckets: `likho-audio` (originals), `likho-normalized` (playable copies), `likho-peaks` (waveforms), `likho-models` (speech models). Browsers never get a bucket's address, only short-lived signed links.
- ## likho-transcription (MongoDB)
	- `transcripts`: one document per version of a call's transcript: recording id, version (1, 2, … unique per recording), job id, language detection (detected, probability, decoded as, policy), the model, timing stats, and **segments**: index, start and end seconds, the text in the spoken script and in Hinglish. A correction writes a new version.
	- `corrections`: one per corrected line: recording, transcript, line, layer, before, after, who, when. Kept as training data.
- ## likho-language (PostgreSQL)
	- | Table | Main columns |
	  | --- | --- |
	  | `glossary_terms` | term, language, enabled, note, heard, last heard at |
	  | `spellings` | source, target, phrase or word, enabled, applied, last applied at |
	  | `spelling_examples` | the last lines a spelling changed: recording, line, before, after |
	  | `language_policies` | in order: detected language, minimum probability, decode as, transliterate |
	  | `vocabulary_versions` | a version number per workspace, raised by every change, so workers know when to reload |
- ## likho-analytics (ClickHouse)
	- `events`: every CloudEvent on the bus, as it came (type, time, workspace, recording, the data), one copy per id.
	- `recordings`: a call's facts each time they change (source, call time, campaign, agent, disposition, deleted); the latest row wins.
	- Views `calls_daily`, `language_mix`, `agent_daily`, `insights_daily`: the same numbers by day, for people and dashboards that query ClickHouse directly.
- ## likho-connector-ameyo (PostgreSQL)
	- `calls`: one row per dialer call fetched: external id, workspace, status, recording id, failure code and reason, attempts, call time, campaign, who asked, when written back to the CRM.
	- `cursors`: where the schedule stands (the dialer's call time of the last call taken).
	- `handled_events`: events already acted on.
	- The dialer's own reporting database is only **read** (the call details, the schedule, and the campaign, agent and call lists people browse), with SQL kept in the installation (`queries/*.local.sql`, never in the repository), and the CRM is only written by the write-back query when it is switched on.
- ## What is never stored
	- Passwords, session cookies, API keys and tokens are stored only as hashes.
	- Phone numbers from the dialer are cut to the last few digits (four by default) before they reach Likho.
	- Transcript text goes to a language model only when one is configured and the company has allowed it.
