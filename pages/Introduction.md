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
	- Wave 0 (foundation) is in progress: the local stack and the contracts exist and are tested. The services and the web app follow in Wave 1.
