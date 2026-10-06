- Where Likho's settings live and the ones that matter most. Each repository's README lists all of its own.
- ## Two kinds of settings
	- **Installation settings** say how a service runs: addresses, ports, passwords, model files. They come from environment variables and `.env` files, and change with a restart.
	- **Workspace settings** say what Likho does for a team: whether every call is transcribed, and so on. An admin changes them in **Admin → Workspace**; they are stored in likho-api and take effect at once.
- ## The .env files
	- Every repository reads, in order, each overriding the one before: `.env`, `.env.local`, `.env.<LIKHO_ENV>`, `.env.<LIKHO_ENV>.local`. A real environment variable wins over all of them. `LIKHO_ENV` is `development` unless set.
	- `.env.development`, `.env.staging`, `.env.production` are committed and hold **no secrets**. The `.local` files hold the secrets of that environment on that machine and are ignored by git.
	- `python likho-infra/scripts/make-env-secrets.py staging` (or `production`) writes every repository's `.env.<environment>.local` with fresh passwords and keys; fill in the public domain and the first admin's email.
	- In Kubernetes the same values come from ConfigMaps (the committed files, synced by `likho-deploy/scripts/sync-env.py`) and Secrets (the `.local` files, applied by `likho-deploy/scripts/secrets.py`). A value is changed in one place: the repository's own file.
- ## The settings that matter most
	- | Setting | Service | What it does |
	  | --- | --- | --- |
	  | `PUBLIC_ORIGIN` | likho-api | the address people open; links in mails point there |
	  | `BOOTSTRAP_ADMIN_EMAIL`, `BOOTSTRAP_ADMIN_PASSWORD` | likho-api | the first admin, made on the first start |
	  | `SMTP_URL`, `MAIL_FROM` | likho-api | where invitation and password mails go; empty = mails are only logged |
	  | `TZ` | likho-api | the company's zone: how a dialer's bare call time is read |
	  | `JOB_*` | likho-api | how stuck jobs are found and tried again ([[Operations]]) |
	  | `DEFAULT_MODEL`, `DEVICE`, `COMPUTE_TYPE`, `CPU_THREADS`, `WORKER_ENABLED` | likho-transcription | which speech model, CPU or GPU, int8 on a CPU, how many threads, whether this process takes jobs |
	  | `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL` | likho-insights | the language model for insights; empty = nothing is analysed and no text leaves |
	  | `QA_FORM_FILE` | likho-insights | the auditor's form, its checks and their points: `config/qa.example.json` ships with the repository; the company's own form goes in `config/qa.local.json` |
	  | `TIMEZONE`, `RETENTION_DAYS` | likho-analytics | where a day begins in the numbers; how long events are kept |
	  | `AMEYO_*`, `DIALER_DATABASE_URL`, `queries/*.local.sql` | likho-connector-ameyo | the dialer's API for the audio, its reporting database for the details |
	  | `SCHEDULE_ENABLED`, `CAMPAIGNS`, `MIN_TALK_SECONDS`, `DAILY_LIMIT` | likho-connector-ameyo | which calls are fetched by themselves, and how many a day |
	  | `WRITEBACK_ENABLED`, `CRM_DATABASE_URL` | likho-connector-ameyo | transcripts written back to the CRM |
	  | `PHONE_DIGITS` | likho-connector-ameyo | how many digits of a phone number are kept (4; 0 = none) |
	  | `*_GRPC_ADDR` | likho-api | where the other services are |
	  | `OTEL_EXPORTER_OTLP_ENDPOINT`, `LOG_LEVEL` | every service | metrics pushed to Grafana; how much is logged |
- ## Workspace settings (Admin)
	- **Transcribe every recording as soon as it is ready** (on by default). Off: a call waits until someone presses Transcribe.
	- The models the workers can run are listed beside it; which is the default is an installation setting of likho-transcription today, and an admin choice from Roadmap step 10.
- ## Before going live
	- Set real secrets (the `.local` files), `PUBLIC_ORIGIN`, the first admin, `SMTP_URL`, `TZ`.
	- Decide, with whoever owns the data, whether transcript text may go to a language model; set `ANTHROPIC_API_KEY` only after a yes.
	- Give the dialer connector its API values, a policy (campaigns, shortest talk time, daily budget) and only then `SCHEDULE_ENABLED=true`.
