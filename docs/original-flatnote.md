> Verbatim copy of the Flatnotes note "Home Therapy Companion Project", as it
> stood on 2026-08-26 when this repo was scaffolded — kept here so the raw
> planning history survives independently of Flatnotes. See
> [`requirements.md`](./requirements.md) for the distilled, current spec, and
> [`adr/`](./adr/) for the decisions made in the grilling session that
> followed.

---

# Home Therapy Companion Project

**Created:** 2026-08-26

## Overview
A self-hosted, Ollama-based daily companion app — an "in-between" of Purpose.ai and Daylio. Interactive chat (not just a mood log), available anytime via GUI (no voice), designed to push back rather than just validate, track specific growth focus areas over time, and help review/draft WhatsApp thread responses (manual copy-paste). History stored as JSONL with periodic Parquet rollups, seeded from Daylio/Purpose.ai exports, and voiced by an external, versioned persona file (prompt-based for now, fine-tuning deferred). **WhatsApp seed data is limited to a one-time bulk import of the last 3 years (revised from "no bulk ingestion") — full 10-year history stays with Purpose.ai.**

---

## Updates

### 2026-08-26 (5)
**Revised WhatsApp scope: allow a one-time bulk import limited to the last 3 years**, rather than no bulk ingestion at all (previous decision). Full 10-year history still explicitly excluded — that stays with Purpose.ai. Rationale: 3 years is recent enough to be relevant to the current state of things without pulling in a much earlier, different chapter of the relationship with equal weight. Manual per-thread paste (for day-to-day tone-check/summary/draft help) remains separate and ongoing regardless of this one-time import.

### 2026-08-26 (4)
**Scope boundary decision (superseded above): no bulk WhatsApp historical ingestion (10 years of chat export) into this tool.** Considered exporting/parsing 10 years of WhatsApp chat history as seed data, but initially decided against any bulk import — that kind of large-scale historical analysis stays with Purpose.ai. Revised in (5) above to allow a bounded 3-year import.

### 2026-08-26 (3)
First live test: Qwen2.5:7b via Open WebUI gave no response (likely model/config issue, not a size problem — see Test Log). Switched to 14B run as an Open WebUI **workspace model** with the system prompt set in the model's own System Prompt field — this worked OK. Open question: whether 14B alone is fast enough to skip the 7-8B/14B split entirely.

### 2026-08-26 (2)
Added: conversation history storage format (JSONL + Parquet), Daylio/Purpose.ai historical data import plan, and an external persona spec file (prompt-based, not fine-tuned yet).

### 2026-08-26
Initial capture of the full concept from planning conversation: model tiering, GUI approach, focus-area tracking, and WhatsApp parsing. Ready to use as the base spec for a Claude Code build session.

---

## Core Concept

- Not a replacement for therapy or for talking to my wife (Lisa) directly — a thinking partner between those conversations and a long-term pattern tracker neither Daylio nor Purpose.ai provides.
- Core requirement: the model must **push back**, not just validate. Base instruction-tuned models default to agreeable — this has to be engineered deliberately via system prompt + few-shot examples, not just described.
- Available all day as an open GUI chat (like any chat app) — not a scheduled "session." No voice/audio capture (ruled out due to consent/legal issues around passive recording, especially of Lisa).
- **Deliberately scoped apart from Purpose.ai**: this tool is the daily interactive companion + a bounded (3-year) WhatsApp seed import; Purpose.ai remains the place for the full long-range historical/WhatsApp analysis. Not intended to fully merge or duplicate that role.

---

## Architecture

### Model tiering (via Ollama)
- **Daily/interactive tier:** 7-8B model (e.g. Qwen3 8B) — fast enough for real-time back-and-forth chat.
- **Periodic analysis tier:** 14B model (e.g. Qwen3 14B / Qwen2.5 14B) — runs on a schedule (weekly/monthly), not interactive, so speed doesn't matter. Reads logs + WhatsApp thread summaries and produces pattern analysis.
- Re-check current best Ollama-available models at build time — this space moves fast (as of Aug 2026: Qwen3 family and gpt-oss:20b are strong 8-16GB options).
- **Live test result (2026-08-26): 14B alone, run via Open WebUI workspace model, felt OK for interactive chat.** If this holds up under more use, may simplify to a single-tier setup (14B for both daily chat and periodic analysis) rather than the original 7-8B/14B split — decide after more real-conversation testing.

### GUI
- Lightweight local web app hitting the Ollama API — open anytime from phone/laptop, same chat window all day, not per-session.
- Consider Open WebUI as a base (already handles Ollama chat UI) vs. building custom frontend for tighter control over prompting/behavior.
- **In practice, using Open WebUI now.** System prompts must be set via Workspace → Models → [model] → System Prompt field — the CLI-only `/set system` command does not work inside the Open WebUI chat box.
- Optional: 1-2x/day gentle notification ("anything on your mind?") that opens straight into chat — not passive listening, just a nudge.

### Memory / continuity
- Vector store (Chroma or SQLite + embeddings) so the daily model can reference past conversations without me re-explaining context.
- Running daily log — messages throughout the day append to the same day's context so morning and evening chats feel continuous.

### Conversation history storage
- **Raw source of truth: JSONL**, not plain text — one JSON object per line: `{"timestamp": ..., "role": "user"|"assistant", "content": ..., "session_id": ...}`. Appendable, human-greppable, trivially loaded into pandas. Rolled by month, e.g. `conversations_2026-08.jsonl`.
- **Parquet: derived analytical layer, not raw storage.** A script periodically converts batches (daily/weekly) into Parquet with extracted structured fields (date, mood, focus_area_tags, word_count, etc.) — columnar format built for the 14B pattern analysis and any future dashboards/charts to query fast.
- Keep raw JSONL forever (source of truth, regenerable); Parquet snapshots can be rebuilt from it at any time.

### Historical data import (seed data)
- **Daylio:** has a well-documented CSV export — columns roughly `full_date, date, weekday, time, mood, activities, note`. A pandas script maps `mood` to the app's scale, splits `activities` into tags, and backfills the daily log table with correctly dated historical entries — becomes real seed data for the very first 14B pattern-analysis pass instead of starting cold.
- **Purpose.ai:** no confirmed bulk-export feature found (worth checking its own settings/data section directly to confirm one way or the other). Plan for a manual path similar to WhatsApp: copy-paste whatever summaries/insights it shows into a "legacy import" note, tagged with a rough date range, so it still slots into the timeline even without message-level granularity.
- **WhatsApp: bounded bulk import, last 3 years only** (revised — see Updates (5)). Full 10-year history explicitly excluded and stays with Purpose.ai.

### WhatsApp integration
- **Two separate paths, kept distinct:**
  1. **One-time bulk seed import — last 3 years only.** Use WhatsApp's own chat export (menu → More → Export chat → "without media") for the Lisa thread, not manual copy-paste, since 3 years of daily messages is too much to paste by hand. Parse into JSONL (`source: whatsapp_historical`), batch by quarter/year, run each batch through the 14B analysis prompt to produce period summaries — store both raw JSONL and the summaries. Retroactively apply focus-area tagging (defensiveness/withholding/withdrawal) across this window for a real historical baseline.
  2. **Ongoing manual copy-paste, no live integration.** For day-to-day thread review/tone-check/draft-response help going forward. Avoids ToS/consent issues entirely (no `whatsapp-web.js` or Business API needed for this path).
- Needs a parser recognizing WhatsApp's copy-paste/export format (varies slightly iOS vs Android, and export vs. copy-paste):
  ```
  [8/26/26, 9:14:32 AM] Lisa: message text
  8/26/26, 9:14 AM - Lisa: message text
  ```
- Regex-based parser (see build notes below) extracts date/time/sender/content, handles multi-line message continuations. Should work for both the bulk export and the manual paste path with minor tweaks.
- Dedicated "paste thread" input in the GUI (separate from normal chat) for the ongoing manual path → parses → hands structured messages to model for: (1) tone/summary, (2) optionally 2-3 draft response options if asked — never auto-send, I choose/edit.
- Open question / worth reflecting on: using the model to draft responses *to* Lisa is different from using it to think through my *own* feelings — worth keeping the drafting feature as an occasional tool, not the default way I respond to her.
- Worth being aware going into the 3-year bulk import: even 3 years can surface a lot at once — decide whether to weight more recent months more heavily than the start of that window, similar to the concern originally raised about the full 10-year read.

---

## Persona

A light, consistent personality — not a strong character, not a bare instruction-following tool.

- **Described externally, in its own versioned file** (`persona.md` or `persona.yaml`), not embedded loose in the app's system-prompt code. The daily-chat system prompt is assembled at session start by reading this file fresh — so the voice lives in one reviewable place and can't drift from ad hoc edits during a session.
- **Fine-tune, don't change on the fly.** Prompt-based for now (edit the file between sessions, not mid-conversation). Actual LoRA fine-tuning into the model weights is a real future option once the persona wording is stable and there are good example transcripts to train on — deliberately deferred for now, not a current task.

**Draft persona spec (v1):**
```yaml
name: persona_v1
voice: even-keeled, plainspoken, dry occasional humor
tone_rules:
  - no exclamation points unless genuinely funny
  - no therapy-speak or forced metaphors
  - same demeanor in good and bad moments
  - short sentences over long ones
pushback_style: asks the next question rather than performing empathy; has opinions but isn't contrarian for its own sake
created: 2026-08-26
```

**Condensed test version used in the live Open WebUI test (2026-08-26):**
```
You're a steady, direct thinking partner — not a cheerleader. Talk plainly, short sentences, no therapy-speak, no exclamation points unless something's genuinely funny. Never just validate: if I describe a conflict, ask about my role in it before commenting on anyone else's. Ask one direct question per response, not a list of gentle ones. If I'm venting without wanting to look at anything, ask directly whether I want to just vent or actually dig into it. Same even tone whether things are going well or badly.
```

Open decisions to settle before finalizing v1:
- Any humor at all, or fully serious given the subject matter?
- First-person casual ("I think...") vs. more observational/reflective ("Sounds like...") — first feels more like a person, second is safer but flatter.

---

## Focus Areas (living, editable — not hardcoded in the prompt)

Stored as a separate JSON/data store, not baked into the system prompt text, so they can be updated over time without editing prompts by hand.

**Current focus areas (starting set):**
1. **Defensiveness** — explaining/justifying instead of hearing feedback or criticism from Lisa.
2. **Withholding** — describing a feeling to the app but not actually saying it to Lisa.
3. **Pulling back / withdrawal** — going quiet, working late, avoiding conflict in ways that could make Lisa feel alone or unseen.

**Schema sketch:**
```json
{
  "id": "defensiveness",
  "label": "Not being defensive",
  "description": "...",
  "watch_for": ["justifying right after describing criticism", "..."],
  "status": "active",
  "created": "2026-08-26",
  "updated": "2026-08-26",
  "notes_history": [{"date": "2026-08-26", "note": "..."}]
}
```

**How focus areas plug into the system:**
- Daily chat: active focus areas loaded fresh into system prompt each session — editing the JSON changes behavior immediately, no prompt rewriting.
- Daily tagging: quick-select tags (defensive / withheld / pulled back / none) auto-generated from the focus_areas list.
- Weekly/monthly analysis: dedicated "growth pattern tracking" section counts/describes instances of each active focus area, notes trend direction, and can suggest (not auto-apply) retiring a resolved focus area or flag a new candidate pattern for me to approve.
- Small UI panel to view/add/edit/retire focus areas and see each one's mini-trend over time.
- The 3-year WhatsApp bulk import (see WhatsApp integration above) will retroactively apply this tagging to build a historical baseline, not just going forward.

---

## Prompting Approach

- System prompt must explicitly forbid pure validation and require pushback — includes few-shot examples of pushback exchanges, not just instructions (examples do more work than description).
- One direct question per response, not a list of gentle ones.
- Ask directly whether I want to be heard vs. actually examined, when I'm just venting.
- Avoid therapy-speak clichés — talk like a smart, honest friend (see Persona section above for the specific voice).
- Reference recall from prior summaries ("this is the third time this month...") rather than treating each day as isolated — this is a memory/RAG problem as much as a prompting one.
- Test for drift: 7-8B models can soften back into agreeableness over a long context even with a strong system prompt — periodic 14B summary re-injects the harder pattern-level view. Persona file also reloaded fresh each session for the same reason.

---

## Weekly/Monthly Analysis Prompt (draft)

Feeds the 14B model the period's logs + WhatsApp summaries, outputs:
1. Recurring themes (specific, not generic)
2. Contradictions or shifts in account/feelings
3. Avoidance patterns
4. Mood trajectory (cite specific entries)
5. Growth pattern tracking — per active focus area, count/describe instances, note trend direction, flag if a focus area looks resolved or a new one should be considered
6. One direct question to open the next period with

Instruction: be specific, cite dates, don't soften findings to be kind.

**Also used, batched by quarter/year, for the 3-year WhatsApp bulk import** — same structure, applied retroactively to produce historical period summaries as seed data (see WhatsApp integration above).

---

## Test Log

| Date | Setup | Result |
|---|---|---|
| 2026-08-26 | `qwen2.5:7b`, first attempt, system prompt via CLI `/set system` inside Open WebUI chat box | Created 41 spurious "tasks" instead of a normal reply — root cause suspected: `/set system` is CLI-only and doesn't apply inside Open WebUI's chat box, so the model was likely running with no persona/system prompt applied, possibly combined with thinking-mode output. |
| 2026-08-26 | `qwen2.5:7b`, retried after guidance | Gave no response at all — likely a model/config issue (interrupted pull, memory, or missing chat template), not a reflection of 7B quality generally. Not yet root-caused. |
| 2026-08-26 | `qwen3:14b` (or similar 14B), run as an Open WebUI **workspace model** with system prompt set correctly in the model's System Prompt field | Worked OK. Real test message used: a genuine, personal message about pre-holiday anxiety, a partnership-not-intimate marriage, sleeping separately for 12+ months, and dread about a week sharing a hotel room. Speed and pushback-quality specifics not yet captured in detail — worth a follow-up note on both once tested further. |

---

## Known Risks / Things to Test Early

- Regex WhatsApp parser needs testing against real pasted samples — iOS/Android/export formats vary.
- 14B model can overfit "recurring themes" to whatever's emotionally loudest in one bad entry rather than a true pattern — sanity-check its reads against my own sense of things before trusting it. Especially relevant for the 3-year bulk import — a rough patch earlier in that window could get equal weight to something recent unless deliberately handled.
- Keep drafting-responses-to-Lisa as a deliberate, occasional tool rather than a default, so the app doesn't become the actual voice in the relationship.
- This is a supplement to talking with Lisa and/or a therapist, not a replacement — worth keeping that framing built into how the app nudges me (e.g., surfacing "have you told her this?" rather than just logging it).
- Confirm whether Purpose.ai actually has a bulk-export option before committing to the manual copy-paste plan.
- `qwen2.5:7b` failed twice in initial testing — root-cause before relying on it (or drop it in favor of `qwen3:8b`/14B-only, pending further tests).
- Keep tool boundaries clean: full long-range WhatsApp history stays in Purpose.ai — this app's WhatsApp scope is capped at 3 years for the bulk import, plus ongoing manual thread review.

---

## Tasks

- [ ] Confirm final model choices at build time (check current Ollama library for best 7-8B / 14B options)
- [ ] Decide GUI base: Open WebUI vs custom frontend
- [ ] Set up Ollama + vector store (Chroma/SQLite) on home server
- [ ] Export last 3 years of WhatsApp thread with Lisa (via WhatsApp's own export feature) for one-time bulk seed import
- [ ] Build WhatsApp parser handling both the export format and manual copy-paste format; test against real samples of each
- [ ] Batch the 3-year export by quarter/year and run through the 14B analysis prompt to produce historical period summaries
- [ ] Write and test full system prompt with few-shot pushback examples
- [ ] Build focus_areas JSON store + UI panel
- [ ] Write weekly/monthly analysis prompt and scheduling (cron)
- [ ] Build daily log schema (structured tags + free text + WhatsApp thread refs)
- [ ] Test for agreeableness drift over multi-day context
- [ ] Set up JSONL logging + periodic Parquet rollup script
- [ ] Write Daylio CSV import script
- [ ] Check Purpose.ai for a real export option; fall back to manual copy-paste import if none
- [ ] Finalize persona_v1 spec (resolve humor and voice-perspective questions above)
- [ ] Root-cause qwen2.5:7b failures, or decide to drop it in favor of a single 14B tier
- [ ] Decide: keep 7-8B/14B split, or go single-tier with 14B based on further speed testing
- [ ] Decide whether/how to weight recency within the 3-year WhatsApp import
- [ ] Reconnect/verify Flatnotes MCP connector for future note syncing

---

#project #self-hosted #ollama #llm #mental-health #relationship #privacy #planning #build-spec #qwen
