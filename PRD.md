# Livestream Analysis — Product Requirements Document

**Project:** livestream-analysis
**Status:** Draft v1
**Last updated:** 2026-06-28

---

## 1. Overview

Livestream Analysis identifies key moments in Twitch and YouTube live streams by analyzing
**audience activity in chat** rather than the video itself. It ingests chat in real time (live) or
from archives (VODs), computes reaction signals — chat velocity, regular-expression/keyword matches,
and emote bursts — and surfaces ranked, potentially clippable moments for a human to review and cut.

The guiding principle is **the machine proposes, the human disposes.** Detection finds candidate
spikes; the reviewer curates them — removing outliers, including/excluding noise words, and
selecting individual messages — before a clip is produced.

This is an open-source project. Chat is the cheapest, highest-signal, lowest-latency proxy for
"something just happened," which lets the tool run with near-zero platform API cost.

---

## 2. Problem & motivation

Streamers and clippers manually scrub hours of VOD or watch live to find the ~30-second moments worth
clipping. It is slow, subjective, and easy to miss moments in long sessions. Existing auto-clip tools
are mostly closed-source, opinionated, and tuned as black boxes.

We want an open, transparent tool where the detection logic is inspectable and the human stays in
control of the final cut.

---

## 3. Goals & non-goals

### Goals
- Accept a Twitch or YouTube stream/VOD link and return a ranked list of candidate clippable moments.
- Support both **live** monitoring and **VOD** review from one analysis engine.
- Detect moments from multiple chat signals: message velocity, regex/keyword matchers, and emote
  reaction classes.
- Let users **search the stream by reaction type** (e.g. funny vs. hype) as a first-class interaction.
- Give users normalization controls — outlier removal, a word bank, and message-level select/deselect.
- Distinguish **raids** (explained spikes) from genuine **anomalies** (unexplained spikes).
- Run at effectively $0 in platform API fees.

### Non-goals (v1)
- Video/audio content analysis (audio-hype fusion is a later phase).
- Fully automated, human-out-of-the-loop clipping.
- Hosting/serving the resulting clips, social publishing, or scheduling.
- Mobile apps. Web-first.

---

## 4. Users

- **Clippers** — comb VODs for shareable moments; want fast candidate surfacing + manual curation.
- **Streamers / their editors** — review their own VODs and live sessions for highlights.
- **Community managers / mods** — want searchable, normalized views of chat reactions.

---

## 5. Signals & detection

### 5.1 Message velocity (primary)
Messages per rolling window vs. a trailing baseline.
- Short window ≈ 8 s. **Window length is intentionally not user-tunable** — 5 s vs 10 s lands on the
  same event because clippers converge on the same moment. One sensible fixed default; no knob.
- Baseline: trailing 2–5 min median.
- Trigger: short-window rate exceeds baseline by a multiplier or MAD-based z-score threshold.

### 5.2 Matchers — regex/keyword as searchable facets ("POGs per second")
A **matcher** is a regex or keyword that tags every message containing it and produces a per-second
count plus its own spike timeline. This is the core search primitive.
- **User-defined search:** type a pattern (e.g. `\bclip(\s?it)?\b`, `W|dub|GG`) → instant count of
  matching messages over time. Ad-hoc searches re-scan the stored log; saved ones become facets.
- **Built-in presets are bundled matchers:** Hype / Funny / Shock / Sad ship as default regex/emote
  sets — no special-cased code. Editable and clonable.
- **Semantics:** case-insensitive + word-boundary by default; optional substring vs whole-message;
  optional per-user dedupe (count unique chatters, not raw hits).

### 5.3 Emote reaction classes
Twitch emotes arrive tagged in message metadata (no image parsing). 7TV/BTTV/FFZ emotes require
fetching each channel's set and matching by text. Reaction classes are just preset matchers over
these emotes.

### 5.4 Viewer count (normalizer, not trigger)
Concurrent viewers update slowly and lag the moment. Used to normalize ("50 messages in 8 s means
more on a 300-viewer stream than a 50k one"), never as a primary trigger.

### 5.5 Composite score
A weighted, normalized blend of the active signals produces a continuous "excitement curve," with
peaks marked as candidates. Weights are user-adjustable. The same code path runs live and on VODs so
results are reproducible.

---

## 6. Normalization & curation (the differentiator)

### 6.1 Automatic outlier handling
- **Median + MAD** baselines (robust to spam floods) instead of mean + standard deviation.
- **De-spam:** collapse runs of identical messages from one user; option to weight by unique chatters.
- **Per-user rate cap** within a window so one fast typer can't manufacture a spike.
- Outliers are flagged, not deleted — the human can override.

### 6.2 Word bank (frequency-based normalization)
Tokenize all chat into a ranked frequency list of most-used words/emotes. Users tag entries to:
- **Exclude** (stopwords/noise) — filler, greetings, bot-command prefixes, constant catchphrases that
  flatten the baseline. Dropped from velocity and matcher counts everywhere downstream.
- **Include** (signal) — promote genuine reaction words, optionally one-click into a saved matcher.

Auto-suggested defaults (built-in stopwords + auto-flagging high-frequency/low-variance noise vs.
bursty signal) plus manual curation. Lists are savable per channel, since each community has its own
slang. Recompute is live.

### 6.3 Message-level select/deselect
For any candidate window the reviewer can toggle individual messages in/out, exclude a user (mod/bot),
drag the clip in/out points, and adjust lead/lag offset (chat reacts late — pull clip start back a few
seconds). The score recomputes as messages are included/excluded.

---

## 7. Review UI

A full-length **activity timeline** is the centerpiece, with the player docked above it.

- **Activity line graph** spanning the entire runtime; clicking a peak seeks the player.
- **Signal switcher (facets):** chips swap what's plotted — overall velocity, a preset reaction class,
  or a live regex/keyword search. Chart, anomaly math, and candidate list recompute against it.
- **Raid indicator:** Twitch raids (`channel.raid` via EventSub) are discrete events drawn as distinct
  markers. Because raids flood chat with new viewers, their spikes are *explained* and
  **deprioritized** in ranking, never surfaced as clippable. (YouTube has no raid event; approximate
  from a sudden viewer-count jump.)
- **Anomaly highlighter:** regions exceeding the expected band (e.g. +2σ via MAD baselines, after
  removing raid/spam noise) are shaded as anomalies — the inverse of the raid marker: an *unexplained*
  spike = a likely real moment. Raid and anomaly spikes are styled distinctly so they're never confused.
- **Word bank panel:** ranked word/emote frequency with include/exclude toggles feeding normalization.
- **Candidate list:** ranked moments with timestamp, matched type (e.g. "Anomaly · Hype," "Raid"),
  σ above baseline, and a one-click Clip action.

Open Y-axis decision: default to **normalized-per-baseline** ("how unusual right now") so small spikes
stay legible next to large raids, rather than raw linear counts.

---

## 8. Architecture

```
Ingestion adapters → Normalize/enrich → Scoring engine → Storage → Review UI / Export
   Twitch IRC/         unify schema,      rolling windows,  event log   timeline, curation,
   EventSub            emote tagging,      MAD/z-score,      + scores    matcher search,
   YouTube live/VOD    viewer join,        composite curve   (SQLite/    clip export
                       word-bank filter                       Postgres)
```

- **Ingestion adapters** normalize to a common event: `{ts, user, text, emotes[], platform, channel}`.
  Both platform adapters target this schema in parallel.
- **Scoring engine:** pure functions over the event stream; identical for live and VOD.
- **Matcher engine:** precompiled regex set scanned per message in one pass; VOD ad-hoc search = re-scan.
- **Storage:** append-only event log + computed scores. SQLite default (zero-config self-host),
  Postgres for hosted.
- **Export:** Twitch → Create Clip API (captures ~last 30 s, no video handling). YouTube → timestamp
  deep-link, or optional local ffmpeg buffer/cut (off by default; heavier, ToS-sensitive).

### Stack
- **Language:** Python (scoring/stats core). Web review UI consumes the engine via API.
- **Twitch ingestion:** EventSub WebSocket (`channel.chat.message`).
- **YouTube ingestion:** `liveChatMessages` streaming for live; chat-replay JSON for VODs.
- **DB:** SQLite → Postgres.
- **License:** MIT or Apache-2.0 (TBD).

---

## 9. API cost & constraints

| Item | Cost |
|------|------|
| Twitch chat (EventSub/IRC) | Free — `channel.chat.message` subscription is cost-0 |
| Twitch Create Clip | Free (rate-limited) |
| YouTube live chat API | Free up to 10,000 quota units/day; quota-bound, not money-bound |
| YouTube VOD chat replay | Free (no live quota) |
| Self-host compute/storage | ~$0–20/mo, or free locally |
| YouTube video buffer + ffmpeg cut | The only meaningful cost; optional, ToS-sensitive |

**Twitch is effectively free at any self-host scale** (limits are operational: ~100 channels/account,
send-rate caps that don't affect a read-only tool). **YouTube costs $0 in fees but is quota-bound** —
the binding constraint is polling frequency, not unit price. Even at 1 unit/call, ~1 s polling exhausts
the daily 10k quota in under 3 hours of a single stream. Mitigations: use `streamList` streaming, parse
VOD replays (no quota), request a free quota increase, or shard across GCP projects. Because v1 targets
both platforms, the YouTube adapter must use streaming/replay parsing — never naive polling — from day one.

---

## 10. Roadmap

- **Phase 0 — Spike:** paste a Twitch *and* YouTube link → ingest chat → print rolling velocity +
  per-reaction-class rates to console. Proves the pipeline.
- **Phase 1 — MVP:** VOD ingestion (both platforms), velocity + matcher curves, timeline UI with
  candidate peaks, raid + anomaly markers.
- **Phase 2 — Curation:** message select/deselect, exclude users, MAD outlier flagging, word bank,
  live score recompute, adjustable lead/lag.
- **Phase 3 — Export + Live mode:** Twitch Create Clip integration; real-time candidate alerts.
- **Phase 4 — Parity & polish:** 7TV/BTTV/FFZ emotes, multi-stream dashboard, savable per-channel configs.
- **Phase 5 — Audio-hype fusion (optional):** volume/pitch/silence signals fused with chat.

---

## 11. Success metrics

- **Precision@N:** of the top N surfaced moments, how many a human keeps as real clips.
- **Recall vs. ground truth:** fraction of human-identified highlights the tool surfaces.
- **Time-to-clip:** median time from pasting a link to exporting a curated clip vs. manual scrubbing.
- **Curation retention:** how often users adjust vs. accept candidates as-is (calibrates thresholds).

---

## 12. Risks & open questions

- **YouTube quota ceiling** is the #1 scaling risk — design around streaming + VOD parsing, not polling.
- **ToS / rate-limit compliance** — read-only chat analysis is well within bounds; recording and
  re-cutting YouTube video is the gray area. Keep that path optional and clearly documented.
- **Emote sets are channel-specific** (7TV/BTTV/FFZ) and drift over time — needs periodic refresh.
- **Threshold tuning must be relative** (per-stream baseline + viewer-count normalization).
- **Privacy** — chat carries usernames; offer an option to anonymize/hash users in stored logs.
- **Open decisions:** license choice; default Y-axis scale (lean normalized-per-baseline).

---

## 13. Immediate next steps

1. Choose a license (MIT or Apache-2.0).
2. Define the shared message-event schema; both adapters target it.
3. Build Phase 0 spike (Twitch + YouTube link → console velocity + reaction rates).
4. Build the matcher engine + reaction-class presets.
5. Add the word bank tokenizer + include/exclude normalization.
