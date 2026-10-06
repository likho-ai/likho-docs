- Likho has four kinds of interface. Each is defined once, in code, and the references people read are generated from that definition and committed beside it.
- | Interface | Used by | Defined in | Reference |
  | --- | --- | --- | --- |
  | REST `/api/v1` | scripts, connectors, other internal apps | likho-api's controllers (NestJS + Swagger) | `/api/docs` on a running API; `likho-api/postman/likho-api.postman_collection.json` |
  | GraphQL `/graphql` | the web apps (through likho-web-sdk) | likho-api's resolvers (code first) | `likho-api/schema.graphql`; GraphiQL at `/graphql` in development |
  | gRPC | service to service | `likho-contracts/proto/likho/*/v1/*.proto` | the proto files; `buf lint` and `buf breaking` in CI |
  | Events | service to service | `likho-contracts/events/*.schema.json`, `streams.yaml` | the schemas and an example of each; validated in CI |
- ## Signing in
	- **People** sign in with email and password; the API sets an HTTP-only session cookie. The web apps send it with every request.
	- **Scripts and connectors** send `Authorization: Bearer lk_…`, an API key an admin makes in Admin → API keys. A key acts as a member of its workspace.
	- **Another system's page** sends `Authorization: Bearer lt_…`, a viewer token its server got from `POST /api/v1/tokens/exchange` with an API key. It reads, for up to an hour, and nothing more.
	- Errors always have one shape: `{"error": {"code": "not_found", "message": "The recording was not found."}}`. Codes: `unauthenticated`, `forbidden`, `not_found`, `invalid`, `conflict`, `service_unavailable`.
- ## REST `/api/v1`
	- | Method and path | Purpose |
	  | --- | --- |
	  | `POST /recordings` | Start an upload: `{ originalName, sizeBytes, externalId?, attributes? }` answers the recording and a link to `PUT` the file to |
	  | `GET /recordings` | List, newest first: `status`, `search`, `externalId`, `campaign`, `agent`, `disposition`, `source`, `since`, `until`, `first`, `after` |
	  | `GET /recordings/{id}`, `DELETE /recordings/{id}` | One recording; delete it with its audio, jobs and transcripts |
	  | `GET /recordings/{id}/audio` | A short-lived link to the playable audio |
	  | `GET /recordings/{id}/transcript` | The newest transcript: both layers, line by line, with the language detected |
	  | `POST /recordings/{id}/transcript/corrections`, `GET …/corrections` | Correct one line (a new version); the corrections made |
	  | `POST /recordings/{id}/jobs`, `GET /recordings/{id}/jobs` | Queue a transcription; the jobs of a recording |
	  | `GET /jobs/{id}`, `POST /jobs/{id}/cancel` | One job's progress; stop it |
	  | `GET /search?q=` | Every transcript line that matches, in either layer, with the facts filters |
	  | `POST /imports`, `GET /imports`, `GET /imports/{id}` | Fetch a call from the dialer by its id; what became of it |
	  | `GET/POST /vocabulary/glossary.csv`, `/vocabulary/spellings.csv` | The vocabulary as CSV, out and in |
	  | `GET /recordings/{id}/insights`, `POST …/insights`, `GET /insights/status` | What the model said about a call; ask again; whether a model is configured |
	  | `GET /analytics/overview`, `/timeseries`, `/breakdown` | The numbers of a window of call time, by day or hour, by agent, campaign, disposition, language, sentiment or source |
	  | `POST /tokens/exchange` | With an API key: a viewer token for another system's page |
	  | `GET /dialer/campaigns`, `/dialer/agents`, `/dialer/calls`, `/dialer/status` | The dialer's own campaigns and agents of a window with their counts, its calls a page at a time (each with its recording when Likho has it), and what the connector is doing |
	  | `GET /settings` | Every setting of the workspace; what a connector reads with its key |
	- Live updates are server-sent events: `GET /events/jobs/{id}` (one job's lines as they are written, and its end), `GET /events/recordings/{id}` (one call), `GET /events/recordings` (everything in the workspace).
	- Health and metrics: `GET /healthz`, `GET /readyz`, `GET /metrics`.
	- **A whole call by script**, with `KEY` an API key:
	- ```bash
	  curl -s -X POST -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
	    -d '{"originalName":"call.mp3","sizeBytes":123456}' http://localhost:8080/api/v1/recordings
	  # → { "recording": { "id": "rec_…" }, "uploadUrl": "…" }
	  curl -s -X PUT --data-binary @call.mp3 -H "Content-Type: audio/mpeg" "<uploadUrl>"
	  # the recording becomes ready and is transcribed (Admin → Workspace); then:
	  curl -s -H "Authorization: Bearer $KEY" http://localhost:8080/api/v1/recordings/rec_…/transcript
	  ```
- ## GraphQL `/graphql`
	- What the web apps use; everything the REST API does and more. The main queries: `me`, `recordings`, `recording`, `recordingFacets`, `recordingCounts`, `transcript`, `transcriptVersions`, `corrections`, `jobs`, `search`, `savedSearches`, `glossary`, `spellings`, `imports`, `insights`, `insightsStatus`, `analyticsOverview`, `analyticsTimeseries`, `analyticsBreakdown`, `dialerCampaigns`, `dialerAgents`, `dialerCalls`, `dialerStatus`, `systemStatus`, `users`, `invitations`, `apiKeys`, `auditLog`, `settings`, `engines`.
	- The main mutations: `login`, `logout`, `requestUpload`, `createJob`, `cancelJob`, `correctSegment`, `retransliterate`, `deleteRecording`, `requestImport`, `requestImports` (up to 200), `saveSearch`, `upsertGlossaryTerm`, `upsertSpelling` and their CSV imports, `analyseRecording`, `inviteUser`, `setUserRole`, `disableUser`, `createApiKey`, `revokeApiKey`, `updateSettings(input)`, password changes and resets.
	- Web apps do not write GraphQL by hand: likho-web-sdk holds the operations and typed React hooks (`useRecordings`, `useTranscript`, `useSearch`, `useAnalyticsOverview`, …).
- ## gRPC (between services)
	- | Service | Served by | Calls |
	  | --- | --- | --- |
	  | `likho.media.v1.MediaService` | likho-media | `GetMedia`, `CreateUpload`, `GetDownloadUrl`, `DeleteMedia` |
	  | `likho.transcription.v1.TranscriptionService` | likho-transcription | `GetTranscript`, `ListTranscripts`, `Transcribe` (streams lines), `Retransliterate`, `CorrectSegment`, `ListCorrections`, `ListEngines`, `CancelJob` |
	  | `likho.language.v1.LanguageService` | likho-language | `Transliterate`, `TransliterateBatch`, `GetHotwords`, `ResolveDecodePolicy`, glossary and spelling calls |
	  | `likho.search.v1.SearchService` | likho-search | `Search`, `Reindex`, `DeleteRecording` |
	  | `likho.insights.v1.InsightsService` | likho-insights | `GetInsights`, `Analyse`, `GetStatus` |
	  | `likho.analytics.v1.AnalyticsService` | likho-analytics | `GetOverview`, `GetTimeseries`, `GetBreakdown` |
	  | `likho.dialer.v1.DialerService` | likho-connector-ameyo | `ListCampaigns`, `ListAgents`, `ListCalls`, `GetCall`, `GetStatus` |
	- Every service also answers the standard gRPC health check.
- ## Events (NATS JetStream)
	- Every event is a CloudEvent with a versioned type (`likho.transcription.completed.v1`) on a fixed subject (`likho.transcription.completed`). The publisher sets the CloudEvent id as the message id, so a duplicate within two minutes is dropped; a consumer acknowledges only when it handled the event and acts on an id once.
	- | Event | From | To |
	  | --- | --- | --- |
	  | `likho.media.uploaded`, `.ready`, `.failed` | likho-media | likho-api (ready, failed) |
	  | `likho.transcription.requested` | likho-api | likho-transcription (the job queue) |
	  | `likho.transcription.started`, `.failed` | likho-transcription | likho-api |
	  | `likho.transcription.completed` | likho-transcription | likho-api, likho-search, likho-insights, the connector (write-back) |
	  | `likho.live.segment` | likho-transcription | likho-api (live lines to the browser), likho-language (heard and applied counts) |
	  | `likho.transcript.corrected` | likho-api | likho-search (kept for ever: training data) |
	  | `likho.vocabulary.updated` | likho-language | likho-transcription |
	  | `likho.import.requested` / `.completed`, `.failed` | likho-api / the connector | the connector / likho-api |
	  | `likho.recording.updated`, `.deleted` | likho-api | likho-search, the connector (deleted) |
	  | `likho.insights.completed`, `.failed` | likho-insights | likho-api |
	  | `likho.settings.changed` | likho-api | the services acting on workspace settings (the keys only) |
	- likho-analytics takes every event above, to count.
	- The full list, with the stream of each and its consumers, is `likho-contracts/streams.yaml`; the README there has the table.
- ## Versions
	- A released field is never removed or renumbered. A breaking change is a new package (`v2`) or a new event type (`….v2`); `buf breaking` enforces it for the protos. Events only gain optional fields.
	- likho-contracts is released with two tags: `vX.Y.Z` (Python, and the TypeScript tarball attached to the GitHub release) and `packages/go/vX.Y.Z` (the Go module).
