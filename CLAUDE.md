# Therapy Companion — conventions for Claude Code

Self-hosted Ollama-based daily reflection companion that pushes back, tracks growth focus areas over time, and helps review WhatsApp thread responses.

## Workflow
- Default branch is `main`, protected, changes land via PR (§1).
- Feature branches: `claude/<description>-<id>` (agents) or `<type>/<description>` (humans).
- Every behaviour-changing PR bumps the version in its single source of truth (§2) — `pyproject.toml`, via `bump-my-version`.
- PR descriptions follow **Summary / Changes / Validation** (§3).

## Testing
`pytest`. Unit tests are the bar for any behaviour change; CI runs them on every PR (§4),
gated at **≥80% coverage** (`pytest --cov --cov-fail-under=80`).

## Linting & formatting (§10)
`ruff check .` (lint) and `ruff format .` (format). CI runs both in check mode on every PR —
an unformatted or lint-failing file fails the build. Run `ruff format .` before committing.

## Design system (§8)
This repo serves a web UI, so it uses the shared **winterbottom design system** —
vendor `winterbottom.css` / `winterbottom-theme.js` from `docker-infra/design-system`
and style through its tokens (see `LOOK-AND-FEEL.md`). Don't hand-roll colours,
spacing or type; use the `design-system-sync` skill to adopt/refresh the assets.

## AI engineering workflow
Claude is the **Lead Engineer** and owns each change end-to-end, with rigor
scaled to the task (canonical doc: devcontainer-sandbox
`standards/AI-WORKFLOW.md`): **trivial** → implement and verify; **normal** →
implement, verify, independent review if useful; **complex** → `architect`
plan first, then implement, verify, and have the relevant independent critics
(`correctness-critic`, `test-critic`, `architecture-critic`,
`security-critic`, `performance-critic`) challenge the result, classifying
each finding ACCEPT / REJECT / DEFER. Do not use multiple agents
automatically — the agents (vendored under `.claude/agents/`, synced from the
template; edit there, not here) advise and challenge, but the Lead Engineer
owns the final decision. A `PostToolUse` hook
(`.claude/hooks/require-tester.sh` — name is historical; registered in
`.claude/settings.json`) fires after every `Write`/`Edit` so verification is
never skipped.

## Reviews
Record every code / architecture review in `docs/reviews/LOG.md` (§9) — use the
`record-review` skill.

## Dev environment
This repo runs in a VS Code devcontainer (§7) and stays aligned with the
[`devcontainer-sandbox`](https://github.com/DavidWinterbottom-2/devcontainer-sandbox)
template via `template-sync`.
