# Architecture

This document walks through how one episode is produced, stage by stage, and the decisions behind each step.

## The stage machine

```
topic → outline → script → factcheck → metadata → voice → visuals → render → gates → upload
```

- An episode is a row in SQLite with a `stage`, a `status` and a JSON `data` blob. Each stage reads what it needs from `data`, writes its result back and moves the stage forward.
- **Errors are typed.**
  - A quota or overload error means "come back later". It does not count as a failure.
  - A quality-gate failure rejects the topic and starts a new one.
  - Any other exception is retried. After 4 attempts the episode is marked failed and a Telegram alert goes out.
- **Scheduling** picks the next free slot (Wed / Fri / Sun, 15:00 local time) and keeps a configurable buffer of finished videos in the queue. A new episode starts only when the buffer runs low.
- **Re-doing an episode** from any stage clears the data owned by that stage and everything after it. The old YouTube upload stays scheduled until the replacement has finished uploading.

## 1. Topic

1. The LLM receives the channel niche, every past topic (rejected ones included, so a failed topic does not come back) and a short performance table (views per day per video).
2. It proposes candidates, each with an angle and search queries.
3. Candidates too similar to past topics (embedding cosine) are dropped.
4. The rest are scored by expected interest and by **footage availability**: real queries against NASA and Commons are run before committing to a topic.

## 2. Outline and script

- **Outline:** a narrative structure that differs from the last three episodes, 6-9 chapters, length derived from the target duration. Chapter titles follow explicit rules for natural Polish (with good and bad examples in the prompt).
- **Script:** written chapter by chapter as "beats" (sentences with ids), which later carry timing and visuals.
- **Anti-template gate:** the script embedding is compared against all finished episodes. If it is too similar, the episode is rejected.

## 3. Fact-check

1. Up to 40 checkable claims are extracted (numbers, dates, records, names, causes), each mapped to the beats it came from.
2. Claims are verified in batches with Google Search grounding, or with Wikipedia when search is unavailable.
3. Wrong claims are rewritten in the affected beats, and unsure claims are softened or removed.
4. If the share of doubtful claims passes a threshold (stricter with Search than with Wikipedia), the topic is dropped.

## 4. Metadata and proofreading

- The LLM writes the title, thumbnail text, description and tags.
- A **proofreading pass** then fixes Polish grammar and naturalness in the title, thumbnail text, chapter titles and every beat.
- Two safety checks guard the rewrite: the number of chapters must stay the same, and each rewritten beat must stay within 75-130% of its original length, so the meaning cannot drift.

## 5. Voice

**Recording**
- The script is synthesized in chunks of about 1,400 characters for natural intonation.
- The voice is chosen by episode number from a rotation of three.
- Chapter announcements ("Chapter two. *Title*.") are recorded separately, with short pauses around them. The chapter card stays on screen for the whole announcement.

**Cleaning**
- Gemini TTS sometimes appends a short burst of noise at the end of a chunk.
- Each chunk is trimmed to its first and last speech window.
- Bursts in the middle are detected and muted. A burst is a frame with very flat spectrum and speech-level loudness, which separates it from sibilants.

**Timing**
- faster-whisper produces word timestamps for the full narration.
- `difflib` aligns the recognized words to the script, which gives exact start and end times for every sentence and every number word.

**Cost**
- Each chunk is cached on disk under a hash of its text and voice.
- The free and paid keys produce identical audio, so if the free quota runs out mid-film, the paid key only records what is missing.

## 6. Visuals

**Direction.** The LLM splits sentences into shots: 3-6 s in the hook, 5-14 s afterwards. Each shot gets an intent:
- specific subject
- general atmosphere
- number card on a given word
- chapter card

**Library.** For each topic a pool of candidates is built from:
- NASA images and video
- Wikimedia Commons (public domain / CC0 / CC BY only, with attribution in the description)
- Pexels and Pixabay space footage

**Ranking.** Candidate descriptions and shot text are embedded with a **local multilingual model** (fastembed). The Gemini embedding API was dropped because its free tier caps at 1,000 requests a day.

**Selection.** A vision model looks at contact sheets of the top candidates and picks the best match for each sentence. "Hero" shots for general moments are chosen as a set sized to the film's needs.

**Checks.**
- Video clips are sampled frame by frame. Clips with burned-in text, talking heads or a different mission are rejected.
- Perceptual hashing (dHash) removes duplicates across sources.
- Adjacent generic shots are merged into longer ones.

**Strict mode.** If the vision model does not answer, the stage raises a quota error and waits. It never falls back to unchecked footage.

## 7. Render

**Segments**
- One segment per shot at 1080p25.
- Images get Ken Burns motion computed in OpenCV with sub-pixel interpolation, which avoids the jitter of ffmpeg's integer `zoompan`.
- Segments are cached by a hash of their content, so re-renders only redo what changed.

**Look**
- Four on-screen frame styles (observatory, control room, archive, orbit), chosen by episode id.
- A chapter label bar, plus chapter and number cards rendered with Pillow.

**Transitions.** Crossfades are applied pairwise. One large `xfade` graph over a whole film ran out of memory on a 4 GB server.

**Audio mix**
```
narration ─┬────────────────────────────────┐
           └─ sidechain key ─┐              ├─ amix ─ loudnorm (-14 LUFS) ─ output
music ── volume ── sidechaincompress ───────┘
```
- The music track is generated for the exact film length.
- Under speech it is compressed gently (ratio 2.5, slow release), so it stays audible but never competes with the voice.

**Subtitles.** An SRT file is built from the aligned sentences and chapter titles.

## 8. Music synthesis

- Every film gets its own track, generated in NumPy from a seed (the episode id). The seed picks:
  - the key
  - the chord progression (minor and Phrygian, dark but not menacing)
  - the chord length
  - the melody
- Four styles rotate between films:
  - **Bells:** pads with a slow, wandering bell melody and echo, which rests every fourth chord.
  - **Pulsar:** dark pads, a soft low heartbeat-like pulse that swells and fades, sparse echoing pings.
  - **Crystal:** pads with a gentle plucked arpeggio that drifts in and out.
  - **Nebula:** a wordless choir-like timbre, an octave shimmer, occasional low gong strikes.
- Each track is normalized to the same RMS, so the mix level is identical across styles.
- A 16-minute track takes 12-50 s to generate and stays under about 650 MB of peak memory.

## 9. Thumbnail

1. Candidates come from NASA and Commons searches for the specific subject, plus the best images from the film.
2. A punchiness heuristic (contrast and saturation, with a penalty for white-background diagrams) narrows them to 10.
3. A vision model picks the final image.
4. The image edges are cropped (agencies put credits in the corners), and the subject is placed on the right, mirrored if needed.
5. The text layout tries every split into 1-3 lines and keeps the one with the largest font that fits.

## 10. Gates and upload

**Gates** check:
- duration against the narration (±1.5 s)
- that both video and audio streams exist
- 1920×1080 resolution
- the share of empty placeholder cards

**Upload**
- Resumable upload with `publishAt`, then the thumbnail and Polish captions.
- The description contains the chapters (YouTube timestamps), the sources and photo credits.
- If the process restarts mid-upload, it looks the video up instead of uploading a duplicate.

## Operations

- `/health` exposes a fingerprint of the running code, so after a deploy it is clear whether the new container is live before anything is triggered.
- Manual controls:
  - `/redo?from_stage=`
  - `/rethumb`
  - `/reupload`
  - `/reschedule`
  - `/retry`
  - `/diag`, which checks API keys without revealing them
- Telegram messages:
  - new episode started
  - uploaded
  - rejected or failed
  - out of credits on the paid key
- A disk guard and automatic cleanup keep only the latest renders.
- Local development runs in **dry-run mode**: fake LLM and silent narrator, with real footage, real music and a real render, no API cost.
