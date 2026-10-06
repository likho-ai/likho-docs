- The words used across Likho and its documentation.
- | Word | Meaning |
  | --- | --- |
  | Recording | One call's audio in Likho, with its facts and its transcripts |
  | Transcript | The text of a recording, line by line, in two layers; every correction makes a new **version** |
  | Layer | **Devanagari** (the script that was spoken) or **Hinglish** (the same words in Roman letters); both are kept for every line |
  | Segment, line | One phrase of a transcript, with its start and end time |
  | Job | One transcription of a recording by a worker; it is queued, runs, and is done, failed or cancelled |
  | Worker | A likho-transcription process that takes jobs from the queue, one at a time |
  | Realtime factor, speed | Seconds of work per second of audio; below 1 is faster than the call itself |
  | Glossary | Names and products the speech model listens for; **heard** counts how often a term appears |
  | Spelling | How a word is written in Hinglish; **applied** counts the lines it changed |
  | Language policy | Which language a call is decoded as, from the language detected and its probability |
  | Facts, attributes | What else is known about a call: campaign, agent, disposition, call time, and the dialer's other fields |
  | Dialer | The call-centre system the calls come from; Likho reaches it through a **connector** |
  | `crt_object_id` | The dialer's id of one interaction; one recording each; Likho stores it as the recording's **external id** |
  | Leg, `call_id` | One dial of an interaction; a transferred call has two legs under one `crt_object_id` |
  | Import | A call asked for from the dialer by its id; the connector fetches it |
  | Schedule | The connector fetching new calls by itself, within a policy and a daily budget |
  | Write-back | The transcript written into the CRM's own table when it is done |
  | Insights | What a language model says about a call: summary, products, mood, the auditor's checks and score |
  | Workspace | One team's recordings, people, vocabulary and settings |
  | API key (`lk_…`) | A key a script or connector uses instead of a person's sign-in |
  | Viewer token (`lt_…`) | A short-lived, read-only token an API key is exchanged for, for another system's page |
  | Event | A message on the bus saying something happened (`likho.transcription.completed`…) |
  | Micro-frontend, app | One part of the web app (library, transcript, admin…), loaded by the shell at run time |
