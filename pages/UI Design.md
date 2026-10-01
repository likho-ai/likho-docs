- Visual version: the **Likho UI Design** canvas (12 artboards: brand, home, home on a phone, dashboard, recordings, transcript light + dark, jobs, search, vocabulary, models, settings); rendered images of every screen are in `images/screens/`. This file is the written spec that the web apps implement. Screens on the canvas use sample data built from the four recordings in this repo.
- Direction: the look of goask.me (its light and dark colours, DM Sans, pill navigation, soft rounded cards, a mascot) with Likho's own logo, mascot and screens. The brand gradient is reserved for the headline, logo and mascot; data marks use one solid accent. Implementation is split across the micro-frontends described in [[Frontend]]; the components below live in `@likho/ui` or in the app that owns the screen.
- ## 1. Tokens
	- The palette follows goask.me in both modes (its CSS was read on 2026-10-01): the violet to indigo page, indigo text, violet-to-indigo buttons and the orange → red → lavender brand gradient. `@likho/ui` defines these as CSS variables; Tailwind v4 reads them with `@theme`.
	- | Token | Light | Dark | Use |
	  | --- | --- | --- | --- |
	  | `--ground` | `#F5F3FF` → `#EDE9FE` → `#E0E7FF` (gradient) | `#0A0B14` | page background |
	  | `--surface` | `#FFFFFF` | `#11142A` | cards, nav, inputs |
	  | `--surface-2` | `#EEF2FF` | `#1A1F3A` | active nav item, example blocks |
	  | `--line` | `#E0E7FF` | `#2B3158` | borders, gridlines |
	  | `--line-strong` | `#A4B3FF` | `#6D5FA8` | input borders, dropzone dashes |
	  | `--ink` | `#1E1A4D` | `#F8FBFF` | primary text |
	  | `--ink-2` | `#4B4792` | `#D7DEF2` | secondary text |
	  | `--ink-3` | `#5F5AA6` | `#9AA8C7` | captions, table headers (≥ 4.5:1) |
	  | `--accent` | `#7C3AED` | `#8B5CF6` | progress, bars, played waveform |
	  | `--accent-strong` | `#5D0EC0` | `#A78BFA` | icons on soft background |
	  | `--accent-soft` | `#EDE9FE` | `#232950` | meter track, banners |
	  | `--active-row` | `#DDD6FF` | `#232950` | the transcript line being played |
	  | `--btn` | `#7C3AED` → `#5047E3` | same | primary buttons (white text, ≥ 5.7:1) |
	  | `--link` | `#4F39F6` | `#A78BFA` | links, timestamps |
	  | `--tag` | `#E0E7FF` / `#432DD7` | `#232950` / `#D7DEF2` | neutral chips |
	  | `--eyebrow` | `#C20039` | `#FF667F` | small upper-case labels |
	  | brand gradient | `#FF7F3F` → `#FF2D58` → `#B872D1` | same | logo, mascot, hero headline |
	- In the light theme the headline gradient starts at coral (`#FF5A4A`), because orange text on the pale page falls below 3:1 contrast; the full gradient is used on graphics and in dark mode.
	- Status chips (dot + label, never colour alone):
	- | State | Light bg / text | Dark bg / text |
	  | --- | --- | --- |
	  | New | `#E0E7FF` / `#372AAC` | `#232950` / `#D7DEF2` |
	  | Queued | `#FEF3C6` / `#7B3306` | `#45330A` / `#FFD236` |
	  | Transcribing | `#EDE9FE` / `#5D0EC0` | `#2A1F5C` / `#C4B4FF` |
	  | Done | `#DCFCE7` / `#016630` | `#0D3B24` / `#7BF1A8` |
	  | Failed | `#FFE2E2` / `#9F0712` | `#4A1620` / `#FFA3A3` |
	- Shape and space: radius 12 (inputs), 20 (cards), 28 (dropzone), full (buttons, chips, nav); card padding 24; page container 1240 px with 24 px gutters (16 on phones); shadow `0 10px 28px rgba(30,26,77,.08)`; every control at least 44 px tall.
- ## 2. Type
	- | Role | Font | Notes |
	  | --- | --- | --- |
	  | Everything Latin: headlines, interface, Hinglish | DM Sans 400–800 (as on goask.me) | hero `clamp(44px, 6.2vw, 84px)` at 800, tracking −0.035em; page titles 32 px; body 16 px; transcript lines 17 px / 1.55 |
	  | Script layer | Noto Sans Devanagari 400/500 | 16 px / 1.7, always with `lang="hi"` |
	  | Timestamps, file names | JetBrains Mono 400/500 | 14 px |
	- Numbers in tables and axes use `tabular-nums`; large stat values use normal figures.
- ## 3. Brand assets (in `docs/brand/`, packaged in `@likho/ui`)
	- | File | What |
	  | --- | --- |
	  | `logo-mark.svg` | rounded square in the gradient with a sound wave above a line of writing; wordmark "Likho" is live text beside it |
	  | `mascot-idle.svg` | Likho: round robot with a headset, used in the hero and empty states |
	  | `mascot-listening.svg` | same, with sound waves — shown while a job runs |
	  | `mascot-done.svg` | happy eyes and a green check — finished states, training-data card |
	  | `empty-recordings.svg` | window with a waveform turning into text lines — empty library, drop hint |
	- All are hand-authored SVG (no raster image generator is available in this workflow); they stay sharp at any size and total under 8 KB. Still to draw in Wave 1: favicon (the mark at 32 px), Open Graph image, empty-search illustration.
- ## 4. Components
	- | Component | Behaviour |
	  | --- | --- |
	  | `NavBar` | white pill, logo left; Dashboard, Recordings, Search, Vocabulary, Models as icon + label; right: Jobs, Settings, theme toggle, **Upload call**. Active item has the `surface-2` pill and `aria-current="page"`. Under 900 px: logo + Upload + menu button opening a sheet. |
	  | `Dropzone` | dashed rounded bar with "Browse files"; the whole page also accepts drops (overlay appears on drag). Accepts mp3, wav, m4a, mp4; several files at once; shows per-file progress in `UploadQueue`. |
	  | `StatusChip`, `Tag`, `LanguageBadge` | as in §1; `LanguageBadge` shows language + probability. |
	  | `Meter` | label, value, 8 px bar; track is `accent-soft`, fill `accent`. Used for language probabilities and progress. |
	  | `WaveformPlayer` | wavesurfer.js over precomputed peaks from likho-media; play/pause, ±5 s, speed 0.75–2×, current / total time. Keys: space play/pause, ←/→ 5 s. |
	  | `LayerToggle` | segmented control **Hinglish / Devanagari / Both**; choice is remembered per user. For English calls it is hidden (one layer). |
	  | `SegmentLine` | timestamp button (seeks), text for the chosen layer(s), pencil to edit. The line being played gets the `accent-soft` background and scrolls into view. J / K move to next / previous line. |
	  | `EditSegment` | inline textarea on the Hinglish layer; Save stores a correction (original kept), shows "Edited by you", emits `transcript.corrected`. Esc cancels. |
	  | `LiveBanner` | while a job runs: listening mascot, "Transcribing, 02:25 of 03:11 written", progress bar, Cancel. New lines fade in at the bottom as they arrive over SSE; the view follows unless the user has scrolled up. |
	  | `DataTable` | real `<table>`, sticky header, row checkbox, horizontal scroll inside the card on narrow screens. |
	  | `StatTile` | label, value, one-line note. No decorative icons. |
	  | `BarChart` | single-hue bars ≤ 24 px wide, 4 px rounded tops, hairline gridlines, 0/2/4-style ticks, value label on the hovered (or latest) bar, hover + keyboard focus readout. |
- ## 5. Screens
	- **Home `/`** — eyebrow "Hindi, Urdu and English calls"; headline "Every call. / Every word. / Written in Hinglish." (first two lines in the gradient); one-sentence promise; "For call teams" card linking to the watched-folder setting; "Meet Likho" with the mascot and a speech bubble ("Aap boliye, main likh leta hoon."); the dropzone; "From sound to Hinglish in four steps" (upload → detect language with probabilities → script layer → Hinglish layer, each with a live example); three facts (every second decoded, both layers kept, search every call); footer.
	- **Dashboard `/dashboard`** — four stat tiles (calls transcribed, audio written down, speed, waiting in queue); "Calls transcribed per day, last 14 days" bar chart; language of calls; "Working on now" with live progress and the next file; latest transcripts.
	- **Recordings `/recordings`** — title + Scan folder / Upload; status filter pills (All, Done, Transcribing, Queued, Failed) and file-name search; table: select, recording, status (with progress bar while transcribing, reason when failed), language, length, added, action (Open / Watch live / View queue / Try again); drop hint with the empty-state art. Selecting rows shows a bulk bar: Transcribe, Delete.
	- **Transcript `/recordings/:id`** — breadcrumb; file name; chips (status, language + probability, length, model); Transcribe again / Download (txt, srt, json); live banner when running; waveform player; layer toggle, find-in-transcript, copy; segment lines. Side column: Language (probability of each candidate + the decode decision), Details (model, audio length, processing time and speed, silence skipped, lines), Versions (each re-transliteration or re-run, newest first).
	- **Jobs `/jobs`** — the running job with the listening mascot, progress, time left and the line just written; "Up next" in order with remove buttons; "Finished" table (result, audio, took, speed, finished at; failure reason in plain words).
	- **Search `/search`** — one large search box, language and date filters, a result count that says which spelling variants were included, results grouped by call with highlighted snippets; each snippet opens the transcript at that second.
	- **Vocabulary `/vocabulary`** — banner explaining that a new spelling rewrites the Hinglish layer from the saved Devanagari without running the model, with "Re-apply to all transcripts"; Glossary (names as removable chips + add field, Devanagari input); Spellings table (heard as → written as, word or phrase) + add row.
	- **Models `/models`** — model cards (name, engine, size, honest note on speed and accuracy on this machine, Default / Available / Not downloaded, action); Language policy table (detected language, condition, decode as, written as); Training data card (corrections collected, Export dataset, note that fine-tuning needs a GPU machine).
	- **Settings `/settings`** — Transcription (model, language, beam size), Chunking (longest chunk, speech sensitivity, skip-silence), Watched folder (path, auto-transcribe), Appearance (System / Light / Dark). One Save button; each field has a one-line explanation.
- ## 6. States every screen handles
	- Loading (skeleton rows, not spinners), empty (illustration + one action), error (what happened + Try again), offline API (banner at the top, cached data stays visible), and the phone layout (cards stack, tables scroll inside their card, nav collapses).
- ## 7. Accessibility
	- Real buttons, links, labels and tables; visible focus rings; 4.5:1 text contrast in both themes (3:1 for the large gradient headline); status never by colour alone; `lang="hi"` on Devanagari; `prefers-reduced-motion` turns off the line fade-in and the listening pulse; all player and transcript actions reachable from the keyboard.
