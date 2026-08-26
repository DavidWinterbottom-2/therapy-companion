# Requirements

Living product spec for the Therapy Companion, as settled after the initial
grilling session (2026-08-26). This is the current, intended shape of the
build — see [`docs/original-flatnote.md`](./original-flatnote.md) for the raw
planning note it was distilled from, and [`docs/adr/`](./adr/) for the
reasoning behind the architectural calls.

## Purpose

A self-hosted daily companion — an "in-between" of Purpose.ai and Daylio.
Interactive chat (not a mood log), available all day like any chat app (not a
scheduled session), that pushes back rather than validates, tracks specific
growth focus areas over time, and helps review/draft WhatsApp thread
responses (manual copy-paste, never auto-sent).

Not a replacement for therapy or for talking to Lisa directly — a thinking
partner between those conversations, and a long-term pattern tracker that
neither Daylio nor Purpose.ai provides. Deliberately scoped apart from
Purpose.ai, which remains the place for full long-range historical analysis.

## Core behaviour

- **Must push back, not validate.** Base instruction-tuned models default to
  agreeable — this is engineered deliberately via system prompt + few-shot
  pushback examples, not just described.
- One direct question per response, not a list of gentle ones.
- When the user is venting without wanting to look at anything, ask directly
  whether they want to be heard or actually examined.
- No therapy-speak clichés — talk like a smart, honest friend.
- Reference recall from prior summaries ("this is the third time this
  month...") rather than treating each day as isolated.
- No voice/audio capture — ruled out on consent/legal grounds, especially
  regarding Lisa.

## Model

Single Ollama model tier — a 14B model (e.g. `qwen3:14b`) used for both
interactive daily chat and periodic pattern analysis. (Originally scoped as a
7-8B/14B split; collapsed to single-tier after a live test showed 14B alone
performs fine interactively — see ADR 0001.) Re-check the best available
Ollama model at build time; this space moves fast.

## Persona

Described externally in a versioned `persona.md`/`persona.yaml`, not embedded
in application code. Read fresh at the start of each session so the voice
lives in one reviewable place and can't drift mid-session. Prompt-based for
now; fine-tuning is a deliberately deferred future option.

**Voice**: even-keeled, plainspoken, dry occasional humour. No exclamation
points unless genuinely funny. Same demeanour in good and bad moments. Short
sentences over long ones.

**Perspective**: first-person ("I think...") — chosen over observational
("Sounds like...") because it reads more like a person, which is the point.

**Pushback style**: asks the next question rather than performing empathy;
has opinions but isn't contrarian for its own sake.

## Focus areas

Living, editable data — not hardcoded into the prompt. Starting set:

1. **Defensiveness** — explaining/justifying instead of hearing feedback or
   criticism from Lisa.
2. **Withholding** — describing a feeling to the app but not actually saying
   it to Lisa.
3. **Pulling back / withdrawal** — going quiet, working late, avoiding
   conflict in ways that could make Lisa feel alone or unseen.

Loaded fresh into the system prompt each session. Daily quick-select tags are
auto-generated from the active list. Weekly/monthly analysis counts/describes
instances per active area, notes trend direction, and can *suggest* (never
auto-apply) retiring a resolved area or flagging a new candidate.

## Memory

Two layers, deliberately separate (ADR 0004):

- **Conversation history (source of truth)**: JSONL, one object per line —
  `{"timestamp", "role", "content", "session_id"}` — rolled monthly. Kept
  forever; regenerable.
- **Analytical layer (derived)**: Parquet, built periodically from the JSONL
  with structured fields (date, mood, focus_area_tags, word_count, ...) for
  fast querying by the pattern-analysis pass and any future dashboards.
- **Semantic recall**: the app's own SQLite + embeddings store (not Open
  WebUI's native Memory — see ADR 0004), so the daily chat can reference past
  conversations without the user re-explaining context.

## Historical seed data

- **Daylio**: CSV export, mapped to the app's mood scale, activities split
  into tags, backfilled as dated historical entries.
- **Purpose.ai**: no confirmed bulk-export feature. Check its own
  settings/data section at build time; if none exists, fall back to a manual
  "legacy import" note — copy-pasted summaries tagged with a rough date
  range.
- **WhatsApp**: bounded one-time bulk import, **last 3 years only** (full
  10-year history stays with Purpose.ai). Ongoing manual paste-thread review
  is separate and permanent.

## WhatsApp integration

Two distinct, permanently separate paths:

1. **One-time bulk seed import (last 3 years)** — via WhatsApp's own chat
   export ("without media"), not manual paste. Parsed into JSONL
   (`source: whatsapp_historical`), batched by quarter/year, run through the
   analysis prompt to produce period summaries. Retroactively tags focus
   areas across the window for a historical baseline. Recency-weighted: older
   quarters are framed to the model as "historical context" rather than
   "current pattern," so a rough patch from years back doesn't get equal
   diagnostic weight to something recent.
2. **Ongoing manual copy-paste** — for day-to-day tone-check/summary/draft
   help, going forward indefinitely. No live integration, no `whatsapp-web.js`
   or Business API — avoids ToS/consent issues entirely.

Parser must handle both the export format and the copy-paste format
(iOS/Android/export vary slightly), including multi-line message
continuations. Drafting responses to Lisa is a deliberate, occasional tool,
not the default way the user responds to her — never auto-sent.

**Open, deliberately deferred**: whether/how to tell Lisa this tool exists
and processes her side of the WhatsApp thread. Not a build blocker; being
handled outside the technical scope of this repo.

## Weekly/monthly analysis

Feeds the model the period's logs + WhatsApp summaries. Outputs: recurring
themes (specific, not generic), contradictions/shifts, avoidance patterns,
mood trajectory (cited), growth pattern tracking per focus area, and one
direct question to open the next period. Instruction: be specific, cite
dates, don't soften findings to be kind. The same structure, applied
retroactively per quarter/year, produces the historical WhatsApp baseline.

## Interface

Custom Flask web app (this repo) — not Open WebUI itself, though it talks to
models *through* the existing Open WebUI gateway (see ADR 0002/0003). Needs:

- Always-open chat, not a per-session flow.
- A dedicated "paste WhatsApp thread" input, separate from normal chat,
  producing tone/summary and optionally 2-3 draft response options.
- A small panel to view/add/edit/retire focus areas and see each one's
  mini-trend.
- Styled with the shared winterbottom design system (§8).

Out of scope for v1: the "anything on your mind?" notification nudge — no
delivery mechanism decided, deferred to a later version.

## Hosting

- `desktop-docker` (always-on, GPU host — already runs Ollama + Open WebUI).
- Exposed publicly via Azure Entra OIDC, restricted to the user's own email
  only — chosen over Tailscale/LAN-only so it's reachable away from home too.
- Nightly backup of the JSONL/SQLite journal data to NAS + Wasabi, built in
  from day one — not git-based, given the sensitivity of the content.

## Known risks / things to test early

- WhatsApp parser needs testing against real pasted samples — iOS/Android/
  export formats vary.
- The model can overfit "recurring themes" to whatever's emotionally
  loudest in one bad entry; sanity-check its reads, especially for the 3-year
  bulk import where an old rough patch could otherwise dominate.
- Keep drafting-responses-to-Lisa an occasional tool, not the default voice
  in the relationship.
- This is a supplement to Lisa/therapy, not a replacement — the app should
  nudge toward "have you told her this?" rather than just logging it.
- Confirm whether Purpose.ai has a real bulk-export option before committing
  to the manual copy-paste fallback.
- Test for agreeableness drift over long multi-day context; the periodic
  analysis pass and persona reload both exist partly to counter this.

## Open tasks (carried from the flatnote, still unresolved)

- [ ] Confirm final model choice at build time (best current 14B option).
- [ ] Set up Ollama-via-Open-WebUI wiring + SQLite/embeddings store on
      `desktop-docker`.
- [ ] Export last 3 years of the WhatsApp thread with Lisa for the bulk seed
      import.
- [ ] Build and test the WhatsApp parser against real samples of both
      formats.
- [ ] Batch the 3-year export by quarter/year and run the analysis prompt to
      produce historical period summaries.
- [ ] Write and test the full system prompt with few-shot pushback examples.
- [ ] Build the focus-areas JSON store + UI panel.
- [ ] Write the weekly/monthly analysis prompt and scheduling (cron).
- [ ] Build the daily log schema (structured tags + free text + WhatsApp
      thread refs).
- [ ] Set up JSONL logging + periodic Parquet rollup script.
- [ ] Write the Daylio CSV import script.
- [ ] Check Purpose.ai for a real export option.
- [ ] Wire up Azure Entra OIDC (single-user `ALLOWED_EMAILS`).
- [ ] Build the nightly NAS/Wasabi backup job.
- [ ] Have the deferred conversation with Lisa about the WhatsApp import
      (outside this repo's scope, but tracked here so it isn't lost).
