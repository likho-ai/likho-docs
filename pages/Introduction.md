- **Likho** (लिखो, "write it down") turns Hindi, Urdu and English call recordings into text a team can read, search and correct.
- ![Likho mascot](../assets/mascot-idle.svg){:height 120, :width 120}
- ## What it does
	- A recording goes in. Two layers of text come out for every line:
		- **Layer 1**, the words in the script that was spoken (Devanagari for Hindi and Urdu calls).
		- **Layer 2**, the same words in **Hinglish**, the Roman-letter Hindi people type in chat.
	- The detected language and its probability are saved with every call.
	- A person can click any line to hear it, correct a wrong word, and search every call.
	- A spelling table decides how names and products are written. Changing a spelling rewrites the Hinglish of old transcripts in seconds, without running the speech model again.
- ## How it is built
	- Small services, each in its own repository with its own data: [[Services]].
	- One event bus, a gateway, login, Kubernetes and CI/CD: [[Platform]].
	- A web app made of micro-frontends: [[Frontend]], with the screens in [[UI Design]].
	- Every interface defined once, with generated references and a Postman collection: [[API Documentation]].
	- Which speech models and open-source projects were studied, and what was chosen: [[Research]].
	- Why things are the way they are: [[adr/0001 Separate repositories]], [[adr/0002 NATS JetStream as the event bus]], [[adr/0003 Two-layer transcripts]], [[adr/0004 NestJS and GraphQL for the API]].
- ## Repositories
	- | Repository | What it is |
	  | --- | --- |
	  | likho-infra | The local stack (databases, event bus, object store, search, gateway) and the shared CI |
	  | likho-contracts | gRPC definitions, event schemas, the stream layout |
	  | likho-transcription | The speech engine: chunking, language detection, both text layers |
	  | likho-language | Hinglish rules, spelling table, glossary, language policy |
	  | likho-media | Uploads, conversion, waveforms, playback |
	  | likho-api | Login, recordings, jobs, live lines; GraphQL and REST |
	  | likho-search | Search across every line of every call |
	  | likho-web-shell and likho-mfe-* | The web app |
	  | likho-ui, likho-web-sdk | Shared design system and web client |
	  | likho-docs | This site |
- ## Status
	- Wave 0 (foundation) is done: the local stack, the contracts, the design system and this site.
	- Wave 1 is done: **likho-api**, **likho-media**, **likho-transcription**, **likho-language**, and the web app (**likho-web-sdk**, **likho-web-shell**, **likho-mfe-library**, **likho-mfe-transcript**). A person signs in, uploads a call (or records one from the microphone), watches the lines arrive, reads both layers, plays the audio from any line, and downloads the transcript. A browser test drives the whole product through the gateway.
	- Wave 2 has begun with **likho-deploy**: Likho in Kubernetes (two Helm charts, the local, staging and production environments, Skaffold), rehearsed on a local minikube cluster. The settings keep their one source, the `.env` files of each repository.
	- The work list from here, step by step with the database designs: [[Roadmap]].
	- Also built in Wave 2: **likho-search** (every line of every call, found by a few words in either layer, typos allowed; the Search page) and **likho-connector-ameyo** (calls from the dialer by schedule or by id, the transcript back to the CRM). Next: corrections, user management, the dialer credentials and policy with the company.
