- What the open-source world offers, which models suit Hindi / Urdu / Hinglish phone calls, and what Likho does with that. Stars and licences were fetched from GitHub and Hugging Face on 2026-10-01; benchmark figures are quoted from the linked pages. Vendor figures come from different test sets and are **not comparable with each other**.
- ## 1. Open-source projects: what to take
	- | Feature | Project | Licence | Use |
	  | --- | --- | --- | --- |
	  | Speaker labels | [pyannote-audio](https://github.com/pyannote/pyannote-audio) (`community-1` model), [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) | MIT code + CC-BY-4.0 gated model; Apache-2.0 | Library, Wave 2. Check for stereo recordings first |
	  | Word timestamps, alignment | [WhisperX](https://github.com/m-bain/whisperX) | BSD-2 | Library / recipe, Wave 2 |
	  | Whisper + alignment + speakers recipe | [whisper-diarization](https://github.com/MahmoudAshraf97/whisper-diarization) | BSD-2 | Recipe only (its default aligner model is non-commercial) |
	  | Waveform player | [wavesurfer.js](https://github.com/katspaugh/wavesurfer.js) | BSD-3 | Dependency, Wave 1 |
	  | OpenAI-compatible API, model load/unload | [speaches](https://github.com/speaches-ai/speaches) | MIT | API shape for `/v1/audio/transcriptions`, registry ideas |
	  | Live streaming | [WhisperLiveKit](https://github.com/QuentinFuxa/WhisperLiveKit), [WhisperLive](https://github.com/collabora/WhisperLive) | Apache-2.0; MIT | Later; also its `bench` command idea |
	  | Self-hosted app (closest product) | [Scriberr](https://github.com/rishikanthc/Scriberr) | MIT | Feature ideas: chat with a transcript, notes |
	  | Subtitle editor UX | [whishper](https://github.com/pluja/whishper) | **AGPL-3.0** | Ideas only |
	  | Search + summary pipeline | [transcriptionstream](https://github.com/transcriptionstream/transcriptionstream) | **GPL-3.0** | Ideas only (Meilisearch, summary prompt) |
	  | Word confidence | [whisper-timestamped](https://github.com/linto-ai/whisper-timestamped) | **AGPL-3.0** | Idea only: flag doubtful words |
	  | Hinglish from the model | [Whisper-Hindi2Hinglish](https://github.com/OriserveAI/Whisper-Hindi2Hinglish) | Apache-2.0 | Benchmark (see §2) |
	- `timothypesi/Speech-to-Text-Converter`: a 40-line Streamlit demo that calls Google's recognizer through the SpeechRecognition library; no licence, no timestamps, no language setting. Nothing to reuse.
	- Gaps no project covers, which Likho already has or plans: deterministic Hinglish with an editable spelling table, corrections feeding the glossary, a no-dropped-seconds guarantee, agent/customer naming and per-leg segments for dialer calls, CPU-only operation.
- ## 2. Models
	- Best independent evidence: the **Voice of India** benchmark ([arXiv 2604.19151](https://arxiv.org/html/2604.19151v2)), 536 h of real phone conversations, spelling-tolerant error rate ("VoI"; lower is better).
	- ### Open models
		- | Model | Hindi / Urdu | Licence | CPU | Note |
		  | --- | --- | --- | --- | --- |
		  | [Whisper large-v3-turbo](https://huggingface.co/openai/whisper-large-v3-turbo) (faster-whisper) | yes / yes | MIT | 1.4x realtime measured here | Current default |
		  | [IndicConformer-600M](https://huggingface.co/ai4bharat/indic-conformer-600m-multilingual) | yes / yes | MIT, gated | ONNX; speed unpublished | VoI Hindi 8.2, Urdu 8.1: best open model. Use FP32 (an int8 community build degraded badly) |
		  | [Hindi2Hinglish Apex](https://huggingface.co/Oriserve/Whisper-Hindi2Hinglish-Apex) / Prime | yes / not claimed | Apache-2.0 | Apex is about turbo-sized; no CTranslate2 build | Outputs Hinglish directly; 700 h noisy Indian audio |
		  | [Vaani Hindi large-v3](https://huggingface.co/ARTPARK-IISc/whisper-large-v3-vaani-hindi), [Trelis Hinglish](https://huggingface.co/Trelis/whisper-hinglish-preview), [vasista22 large-v2](https://huggingface.co/vasista22/whisper-hindi-large-v2) | yes / no | Apache-2.0 | Slow (large size) | Candidates once a GPU exists |
		  | [Omnilingual ASR](https://github.com/facebookresearch/omnilingual-asr) | yes / yes | Apache-2.0 | GPU-oriented | VoI 13.7 / 16.0 (7B) |
		  | MMS, SeamlessM4T v2 | yes / yes | **CC-BY-NC** | — | Ruled out (non-commercial) |
		  | distil-whisper, Parakeet, Canary | no | — | — | No Hindi |
	- ### Cloud engines (optional, if audio may leave the company)
		- | API | Hindi / Urdu | Speakers | Published price | Note |
		  | --- | --- | --- | --- | --- |
		  | [Sarvam Saaras](https://docs.sarvam.ai/api-reference-docs/getting-started/models/saaras) | yes / yes, code-mix and translit modes, 8 kHz tuned | batch | [INR 30/h, INR 45/h with speakers](https://www.sarvam.ai/api-pricing) | Top of VoI (5.0–6.2). No word timestamps |
		  | [ElevenLabs Scribe v2](https://elevenlabs.io/docs/overview/capabilities/speech-to-text) | strong Hindi, weak Urdu | yes + word timestamps | [$0.22/h](https://elevenlabs.io/pricing/api) | VoI 7.7 / 25.5 |
		  | [Deepgram Nova-3](https://developers.deepgram.com/docs/models-languages-overview) | yes / yes | yes | [$0.0043/min](https://deepgram.com/pricing) | VoI Hindi 13.0 |
		  | OpenAI gpt-4o-transcribe | multilingual | separate model | $0.006/min | VoI Hindi 33.9: avoid |
		- Not verified online (third-party pages only): Azure and Google prices; data-residency statements; pyannote CPU speed.
	- ### Recommendation
		- Default now: keep `large-v3-turbo` int8.
		  logseq.order-list-type:: number
		- Build a **gold set**: about 60 minutes of our own calls, hand-corrected, split by call. Score WER and CER with an Indic normaliser (Whisper's normaliser understates Hindi errors) and spelling-tolerant scoring for Hinglish; record speed and cost.
		  logseq.order-list-type:: number
		- Benchmark in this order: IndicConformer-600M (ONNX FP32) → Oriserve Apex (converted to CTranslate2 int8) → Sarvam Saaras batch → ElevenLabs Scribe v2.
		  logseq.order-list-type:: number
		- Speakers: split channels if recordings are stereo; otherwise pyannote `community-1` with two speakers (about 0.86x realtime on CPU, so on demand only).
		  logseq.order-list-type:: number
		- Word timestamps: faster-whisper `word_timestamps=True`; WhisperX alignment if they drift.
		  logseq.order-list-type:: number
		- VAD: stay on Silero, move to v6 (native 8 kHz).
		  logseq.order-list-type:: number
		- Hinglish: rules + spelling table remain the deterministic layer 2. Optional LLM polish only with a glossary, minimal edits, numbers and names untouched, an edit-distance limit, raw layers always kept ([EACL 2026](https://aclanthology.org/2026.eacl-short.45/) found mid-sized LLMs can make Hindi transcripts worse).
		  logseq.order-list-type:: number
		- Fine-tuning: LoRA on turbo from corrected lines, merge, convert back; a rented 24 GB GPU; prove the gain on the gold set.
		  logseq.order-list-type:: number
- ## 3. Effect on the design
	- `likho-transcription/engines/` gets adapters: `faster_whisper` (now), `indic_conformer`, `sarvam`, `elevenlabs`; an engine declares its output layer (script or Hinglish).
	- `likho-ml` owns the registry and the benchmark job; the Models page shows each engine's gold-set score, speed and cost.
	- The transcript document gains optional `speaker`, `leg` and `words[]` per line.
