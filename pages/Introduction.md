- **Likho** (लिखो, "write it down") turns Hindi, Urdu and English call recordings into text a team can read, search and correct, and tells what happened in them.
- ![Likho mascot](../assets/mascot-idle.svg){:height 120, :width 120}
- ## What it does
	- A recording goes in, by upload, by script, or from the call centre's dialer. Two layers of text come out for every line:
		- **Layer 1**, the words in the script that was spoken (Devanagari for Hindi and Urdu calls).
		- **Layer 2**, the same words in **Hinglish**, the Roman-letter Hindi people type in chat.
	- The detected language and its probability are saved with every call, with the call's facts from the dialer: campaign, agent, disposition, time.
	- A person clicks any line to hear it, corrects a wrong word (every correction is kept, and is what the model learns from), and searches every word of every call, typos allowed.
	- A glossary tells the speech model the names and products to listen for; a spelling table decides how they are written. Changing a spelling rewrites the Hinglish of old transcripts in seconds, without running the speech model again.
	- A language model, when the company allows it, summarises each call, reads the customer's mood and pre-fills the auditor's form with a score.
	- The numbers: calls, minutes, speed, languages, moods and scores by day, by agent and by campaign; yesterday's on the home page.
	- Another system, such as a reports portal, shows a call's transcript beside its own recording button.
- ## Read next
	- **Using it**: [[User guide]], screen by screen; the words in [[Glossary]].
	- **Running it**: [[Setup and start]] (from nothing to the first call), [[Configuration]], [[Operations]].
	- **How it is built**: [[Services]] (each service, its data and its calls), [[Data model]] (where every kind of data lives), [[Platform]] (event bus, login, Kubernetes, CI/CD), [[Frontend]] and [[UI Design]] (the web app), [[API Documentation]] (REST, GraphQL, gRPC, events).
	- **Why**: [[Research]] (the speech models and projects studied) and the decision records [[adr/0001 Separate repositories]], [[adr/0002 NATS JetStream as the event bus]], [[adr/0003 Two-layer transcripts]], [[adr/0004 NestJS and GraphQL for the API]].
	- **What comes next**: [[Roadmap]].
- ## Repositories
	- | Repository | What it is |
	  | --- | --- |
	  | likho-infra | The local stack (PostgreSQL, MongoDB, Redis, NATS, the S3 store, Meilisearch, ClickHouse, the gateway, Grafana) and the shared CI |
	  | likho-contracts | gRPC definitions, event schemas, the stream layout; Python, Go and TypeScript packages |
	  | likho-api | Sign-in, people and roles, recordings, jobs, live lines, search, vocabulary, insights, analytics, the audit log; GraphQL and REST |
	  | likho-media | Uploads, playable copies, waveforms, signed links |
	  | likho-transcription | The speech engine and the job workers: both text layers, versions, corrections |
	  | likho-language | Hinglish, the spelling table, the glossary, the language policy, their usage counts |
	  | likho-search | Every line of every call, searchable in both layers |
	  | likho-insights | What a language model says about a call |
	  | likho-analytics | Every event in ClickHouse, and the numbers behind the calls |
	  | likho-connector-ameyo | Calls from the dialer, by id or by schedule; transcripts back to the CRM |
	  | likho-web-sdk, likho-ui | The typed client and React hooks; the design system |
	  | likho-web-shell | The web app's frame: sign-in, navigation, home, search, settings, the embed page |
	  | likho-mfe-library, -transcript, -vocabulary, -insights, -admin | The web app's parts, each released on its own |
	  | likho-deploy | Likho in Kubernetes: Helm charts, environments, Skaffold |
	  | likho-docs | This site |
- ## Status (6 October 2026)
	- Built and proven end to end on one machine: upload, record and dialer import; transcription with live lines; both layers; corrections and versions; search; the vocabulary with usage counts; people, roles, invitations, API keys and the audit log; job recovery, metrics and a Grafana dashboard; insights (with a local stand-in model until the company allows a real one); the numbers in ClickHouse with the dashboards; the transcript inside another system's page. Kubernetes charts are rehearsed on minikube.
	- Waiting for the company's decisions: whether transcript text may go to a language model, and the dialer schedule's policy (campaigns, daily budget) with the CRM write-back.
	- Next: the model registry and the training loop from corrections, then scale and operations. The details are on the [[Roadmap]].
