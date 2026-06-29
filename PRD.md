# Livestream Analysis — Product Requirements Document

**Project:** livestream-analysis
**Status:** Draft v1
**Last updated:** 2026-06-28

## 1. Overview

Livestream Analysis identifies key moments in Twitch and YouTube live streams by analyzing
audience activity in chat rather than the video itself. It ingests chat in real time (live) or
from archives (VODs), computes reaction signals — chat velocity, regex/keyword matches, and emote
bursts — and surfaces ranked, potentially clippable moments for a human to review and cut.

The guiding principle is the machine proposes, the human disposes. Detection finds candidate
spikes; the reviewer curates them — removing outliers, including/excluding noise words, and
selecting individual messages — before a clip is produced.

This is an open-source project. Chat is the cheapest, highest-signal, lowest-latency proxy for
"something just happened," which lets the tool run with near-zero platform API cost.

## 2. Problem & motivation

Streamers and clippers manually scrub hours of VOD or watch live to find the ~30-second moments
worth clipping. It is slow, subjective, and easy to miss moments in long sessions. Existing
auto-clip tools are mostly closed-source, opinionated, and tuned as black boxes. We want an open,
transparent tool where the detection logic is inspectable and the human stays in control.

## 3. Goals & non-goals

Goals: accept a Twitch or YouTube link and return ranked candidate moments; support both live and
VOD from one engine; detect from multiple signals (velocity, regex/keyword matchers, emote
classes); let users search the stream by reaction type; provide normalization controls (outlier
removal, word bank, message-level select/deselect); distinguish raids (explained spikes) from
genuine anomalies (unexplained spikes); run at effectively $0 in platform API fees.

Non-goals (v1): video/audio content analysis; fully automated human-out-of-the-loop clipping;
hosting/publishing clips; mobile apps (web-first).

## 4. Users

Clippers comb VODs for shareable moments. Streamers and their editors review their own sessions.
Community managers and mods want searchable, normalized views of chat reactions.

## 5. Signals & detection

5.1 Message velocity (primary): messages per rolling window vs a trailing baseline. Short window
~8s; window length is intentionally not user-tunable (5s vs 10s lands on the same event). Baseline
is the trailing 2–5 min median. Trigger when the short-window rate exceeds baseline by a multiplier
or a MAD-based z-score threshold.

5.2 Matchers (regex/keyword as searchable facets, "POGs per second"): a matcher is a regex or
keyword that tags every message containing it and produces a per-second count plus its own spike
timeline. User-defined search (e.g. \bclip(\s?it)?\b, W|dub|GG) gives an instant count over time;
ad-hoc searches re-scan the stored log, saved ones become facets. Built-in Hype/Funny/Shock/Sad
are bundled preset matchers, not special-cased code. Case-insensitive + word-boundary by default;
optional per-user dedupe (count unique chatters, not raw hits).

5.3 Emote reaction classes: Twitch emotes arrive tagged in message metadata (no image parsing).
7TV/BTTV/FFZ emotes require fetching each channel's set and matching by text. Reaction classes are
just preset matchers over these emotes.

5.4 Viewer count (normalizer, not trigger): concurrent viewers update slowly and lag the moment;
used to normalize for audience size, never as a primary trigger.

5.5 Composite score: a weighted, normalized blend of active signals produces a continuous
excitement curve with peaks marked as candidates. Weights are user-adjustable. The same code path
runs live and on VODs so results are reproducible.

## 6. Normalization & curation (the differentiator)

6.1 Automatic outlier handling: median + MAD baselines (robust to spam floods); de-spam by
collapsing identical-message runs; option to weight by unique chatters; per-user rate cap within a
window. Outliers are flagged, not deleted — the human can override.

6.2 Word bank (frequency-based normalization): tokenize all chat into a ranked frequency list of
most-used words/emotes. Users exclude entries (stopwords/noise — filler, greetings, bot-command
prefixes, constant catchphrases) so they stop polluting the baseline, or include entries (genuine
reaction words), optionally one-click promoted into a saved matcher. Auto-suggested defaults
(built-in stopwords plus auto-flagging high-frequency/low-variance noise vs bursty signal) plus
manual curation. Lists are savable per channel. Recompute is live.

6.3 Message-level select/deselect: for any candidate window the reviewer toggles individual
messages in/out, excludes a user (mod/bot), drags clip in/out points, and adjusts lead/lag offset
(chat reacts late). The score recomputes as messages are included/excluded.

## 7. Review UI

A full-length activity timeline is the centerpiece, with the player docked above it. An activity
line graph spans the entire runtime; clicking a peak seeks the player. A signal switcher (chips)
swaps what is plotted — overall velocity, a preset reaction class, or a live regex/keyword search —
and the chart, anomaly math, and candidate list recompute against it. A raid indicator marks Twitch
raids (channel.raid via EventSub) as discrete events; because raids flood chat with new viewers,
their spikes are explained and deprioritized in ranking, never surfaced as clippable (YouTube has no
raid event; approximate from a sudden viewer-count jump). An anomaly highlighter shades regions
exceeding the expected band (e.g. +2 sigma via MAD baselines, after removing raid/spam noise) — the
inverse of the raid marker: an unexplained spike is a likely real moment. A word bank panel exposes
ranked word/emote frequency with include/exclude toggles. A candidate list ranks moments with
timestamp, matched type, sigma above baseline, and a one-click Clip action. Default Y-axis is
normalized-per-baseline so small spikes stay legible next to large raids.

## 8. Architecture

Ingestion adapters normalize to a common event {ts, user, text, emotes[], platform, channel}; both
platform adapters target this schema in parallel. The scoring engine is pure functions over the
event stream, identical for live and VOD. The matcher engine scans a precompiled regex set per
message in one pass; VOD ad-hoc search is a re-scan. Storage is an append-only event log plus
computed scores (SQLite default for zero-config self-host, Postgres for hosted). Export: Twitch via
Create Clip API (captures ~last 30s, no video handling); YouTube via timestamp deep-link or optional
local ffmpeg buffer/cut (off by default; heavier, ToS-sensitive).

Stack: Python (scoring/stats core) with a web review UI consuming the engine via API. Twitch
ingestion via EventSub WebSocket (channel.chat.message). YouTube ingestion via liveChatMessages
streaming for live and chat-replay JSON for VODs. DB SQLite to Postgres. License MIT or Apache-2.0
(TBD).

## 9. API cost & constraints

Twitch chat (EventSub/IRC) is free — the channel.chat.message subscription is cost-0; limits are
operational (~100 channels/account, send-rate caps irrelevant to a read-only tool). Twitch Create
Clip is free (rate-limited). YouTube live chat API is free up to 10,000 quota units/day but
quota-bound, not money-bound: the binding constraint is polling frequency, not unit price — even at
1 unit/call, ~1s polling exhausts the daily quota in under 3 hours of one stream. Mitigations: use
streamList streaming, parse VOD replays (no quota), request a free quota increase, or shard across
GCP projects. Because v1 targets both platforms, the YouTube adapter must use streaming/replay
parsing — never naive polling — from day one. Self-host compute/storage runs ~$0–20/mo or free
locally. The only meaningful money cost is the optional YouTube video buffer + ffmpeg cut path.

## 10. Roadmap

Phase 0 (spike): paste a Twitch and YouTube link, ingest chat, print rolling velocity + reaction
rates to console. Phase 1 (MVP): VOD ingestion both platforms, velocity + matcher curves, timeline
UI with raid + anomaly markers. Phase 2 (curation): message select/deselect, exclude users, MAD
outlier flagging, word bank, live recompute, adjustable lead/lag. Phase 3 (export + live): Twitch
Create Clip, real-time alerts. Phase 4 (parity & polish): 7TV/BTTV/FFZ emotes, multi-stream
dashboard, savable per-channel configs. Phase 5 (optional): audio-hype fusion.

## 11. Success metrics

Precision@N of surfaced moments a human keeps; recall vs human-identified highlights; time-to-clip
vs manual scrubbing; curation retention (accept vs adjust) to calibrate thresholds.

## 12. Risks & open questions

YouTube quota ceiling is the top scaling risk. ToS/rate-limit compliance: read-only chat analysis is
fine; recording and re-cutting YouTube video is the gray area (optional, documented). Emote sets are
channel-specific and drift. Thresholds must be relative. Chat carries usernames — offer anonymize/
hash. Open: license choice; default Y-axis scale (lean normalized-per-baseline).

## 13. Immediate next steps

Choose a license; define the shared message-event schema; build the Phase 0 spike; build the matcher
engine + reaction presets; add the word bank tokenizer + include/exclude normalization.
