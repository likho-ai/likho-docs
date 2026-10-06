- Running Likho day to day: knowing it is healthy, finding out why it is not, and keeping its data. Starting it is [[Setup and start]]; settings are in [[Configuration]].
- ## Health
	- Every service answers on its HTTP port: `GET /healthz` (the process is alive) and `GET /readyz` (its databases and the bus answer). Kubernetes uses both; locally, `curl http://localhost:4000/readyz` and so on.
	- | Service | HTTP port | gRPC port |
	  | --- | --- | --- |
	  | likho-api | 4000 (also behind the gateway on 8080) | - |
	  | likho-media | 4010 | 5010 |
	  | likho-transcription | 4020 | 5020 |
	  | likho-language | 4030 | 5030 |
	  | likho-search | 4040 | 5040 |
	  | likho-insights | 4050 | 5050 |
	  | likho-connector-ameyo | 4060 | - |
	  | likho-analytics | 4070 | 5070 |
	- The stack: `.\stack.ps1 ps` in likho-infra shows every container, its health and its memory; `.\stack.ps1 smoke` runs the checks that prove it works.
- ## Metrics, logs and the dashboard
	- Every service serves Prometheus metrics at `GET /metrics`: requests and their duration, events handled, and its own counts (jobs, lines written, transcription speed, imports, model calls, events stored).
	- Logs are JSON, one object per line, with the same field names in every service; `LOG_LEVEL` sets how much.
	- `.\stack.ps1 obs` starts Grafana (http://localhost:3000) with Loki, Tempo and Prometheus. Services push metrics there when `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318`. The dashboard **Likho - jobs and services** is provisioned from likho-infra: jobs waiting and running, the oldest waiting job, the realtime factor, failed jobs and sweeps, lines transcribed per minute, media by status, events handled.
- ## Jobs that go wrong
	- likho-api sweeps the jobs every minute (`JOB_SWEEP_SECONDS`): a job queued longer than `JOB_QUEUED_MAX_MINUTES` or silent longer than `JOB_STALL_MAX_MINUTES` is tried again, up to `JOB_MAX_ATTEMPTS`, then marked failed with the reason. A job that keeps failing ends on the `likho.dead` subject, which is kept for a look.
	- A job that stays **queued** means no worker took it: likho-transcription is not running, or is still downloading its model (its log says).
	- **Cancel** on the call's page stops a waiting or running job.
- ## Events
	- Each consumer is a durable consumer named `<service>-<subject>`; an event is acknowledged only when it was handled, delivered again up to five times otherwise, and acted on once even when delivered twice (`handled_events`).
	- To see what is on the bus: `docker run --rm --network likho natsio/nats-box:0.20.0 nats --server nats://nats:4222 stream ls` (or `consumer report LIKHO`).
	- A service started for the first time takes the events the stream still holds (30 days); `CONSUMER_START=new` starts from now on.
- ## Data and backups
	- Locally the data is in Docker volumes (`.\stack.ps1 down` keeps it, `.\stack.ps1 reset` removes it). In Kubernetes each backing service is a StatefulSet with its own volume.
	- What to back up, and what can be rebuilt:
	- | Data | Back up? | Why |
	  | --- | --- | --- |
	  | PostgreSQL (all databases) | yes | people, recordings, jobs, settings, vocabulary, the connector's state |
	  | MongoDB | yes | transcripts and corrections: the work of the speech model and of people |
	  | S3 store, bucket `likho-audio` | yes | the original recordings |
	  | S3 buckets `likho-normalized`, `likho-peaks` | no | made again from the originals |
	  | Meilisearch | no | rebuilt from the transcripts (`Reindex`) |
	  | ClickHouse | optional | rebuilt from the events the stream still holds; older events only from a backup |
	  | NATS `LIKHO_KEEP` | yes | the corrections stream, kept for ever, training data |
	- Scheduled snapshots and a restore drill are Roadmap step 11.
- ## When something is off
	- **A page says an app "could not be loaded"**: that app's server is not running (locally: its Vite dev server); the rest of Likho keeps working. Start it and press **Try again**.
	- **"The … service is not answering right now"**: the named service is down or still starting; its `/readyz` and its log say why.
	- **A service stops at start-up**: it could not reach NATS within `NATS_CONNECT_TIMEOUT_SECONDS`, or its database. Start the stack first.
	- **Docker does not start on Windows**: Docker Desktop keeps its data disk (a large `.vhdx` file) on the system drive by default, and a nearly full drive hangs the engine. Keep the disk on a drive with room, and change its location only from Docker Desktop's settings, with Docker otherwise idle; a half-finished move leaves the disk locked ("being used by another process") until whatever holds it lets go.
	- **After a change to the web SDK, an app shows "… is not a function"**: the shell still serves the old package to the apps. Restart the shell's dev server with `--force`.
	- **Tests take jobs that are not theirs**: stop your own likho-transcription before running a service's tests; a running worker takes the test's jobs.
