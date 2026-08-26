# Therapy Companion

A self-hosted daily reflection companion: a chat app that pushes back rather
than validates, tracks personal growth patterns over time, and helps review
WhatsApp thread responses — distinct from Purpose.ai (full long-range
historical analysis) and Daylio (mood logging).

## Language

**Focus area**:
A specific, named behaviour pattern being tracked over time (e.g.
"defensiveness"), stored as living/editable data rather than hardcoded into
the prompt. Has a status (`active`/retired) and a trend.
_Avoid_: tag, category, trait.

**Persona**:
The externally versioned file (`persona.md`/`persona.yaml`) describing the
companion's voice and pushback style, read fresh at the start of every
session. Prompt-based, not a fine-tuned model.
_Avoid_: system prompt (the persona is the source; the system prompt is
assembled from it plus focus areas each session), character.

**Pushback**:
The model's core required behaviour — asking the next direct question
instead of validating or performing empathy. The opposite of the base
model's default agreeableness.
_Avoid_: challenge, confrontation.

**Bulk seed import**:
The one-time, bounded historical WhatsApp import (last 3 years, via
WhatsApp's own export feature) used to backfill a historical baseline.
Permanently separate from ongoing manual review.
_Avoid_: WhatsApp integration (too broad — covers both paths), sync.

**Ongoing manual review**:
The day-to-day path where the user pastes a WhatsApp thread into a dedicated
input for tone-check/summary/draft-response help. No live integration, no
API access to WhatsApp itself.
_Avoid_: paste thread (the UI element, not the practice), import.

**Growth pattern tracking**:
The section of the periodic (weekly/monthly) analysis output that counts and
describes instances of each active focus area and notes trend direction.
_Avoid_: analytics, reporting.

**Period summary**:
A structured output of the analysis prompt (recurring themes, contradictions,
avoidance patterns, mood trajectory, growth pattern tracking, one opening
question) for a given time window — used both for the regular weekly/monthly
cadence and, retroactively per quarter/year, for the WhatsApp bulk import.
_Avoid_: report, digest.

**Draft response**:
A model-suggested reply to a message from Lisa, offered only on request as
an occasional tool — never auto-sent; the user always chooses/edits.
_Avoid_: auto-reply, suggestion (too generic — this is specifically about
messages to Lisa).
