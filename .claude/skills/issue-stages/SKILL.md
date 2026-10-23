---
name: issue-stages
description: Drive a GitHub issue through the claude-tracker board stages (stage:define → investigate → spec → implement → test → review → deploy), one gated stage at a time. Use when asked to work on, pick up or take an issue through the process, or to move an issue's stage label.
---

Work an issue as a **gated pipeline**. Each stage has a gate: a checkable condition, usually David's explicit approval in the conversation. Do the stage's work, report it on the issue, and stop at the gate. Never do a later stage's work early: a fix that's already built makes define and spec theatre. This happened on german-story-generator#48, whose first PR was closed unmerged.

## Rules

- **Labels:** an issue carries exactly one `stage:*` label. Move it only when the current gate is passed. Moving it is your job; passing the gate is David's.
- **Gates need explicit approval.** Silence, a question, or "looks good so far" doesn't pass a gate. Ask.
- **Issues close at merge.** The PR says `Closes #N`. Deployment is tracked on the commit, by the deploy check (stage 7), not by keeping the issue open.
- **Never close an issue by hand**, and never merge. David merges.
- **Size the issue at define:**
  - **Small** (one module, no new behaviour surface): define and spec collapse into one issue comment plus a short `proposal.md` + `tasks.md`, with 3–4 grill questions and no `design.md`. Review is a code review only.
  - **Big** (cross-repo, new component or workflow, auth-adjacent, or new tooling): every stage in full.
  - If unsure, call it big.
- Mention `#N` in every commit and PR so the tracker links the work.

## Stages

### 1. define: what is needed

Read the issue and the repo's `CLAUDE.md`. Find what the issue missed. In #48 that was: the Docker image didn't ship the version file, and `Closes` would have closed the issue before deploy. Ask David every open requirement question, each with your recommended answer.

**Gate:** David agrees the requirements list. Post it on the issue as **"Define: agreed requirements"**, with numbered requirements and a definition of done. Then move to `stage:investigate`, or straight to `stage:spec` if nothing is unknown.

### 2. investigate (optional): find out what's going on

Use this stage for bugs, unknown behaviour or unconfirmed assumptions (which repos are affected, how a system really behaves). Use the `diagnosing-bugs` skill for bugs. Write the findings on the issue.

**Gate:** every unknown from define is answered, or recorded as an explicit assumption that David accepts.

### 3. spec: write down what will be built

1. **OpenSpec:**
   - Branch `openspec/<change-id>`.
   - If the repo has no `openspec/`, run `npm i -g @fission-ai/openspec@1.14.3` (if `openspec` is missing), then `openspec init --tools claude --no-animation --no-copilot-cloud`.
   - Then follow the `openspec-propose` skill to write `proposal.md`, the spec deltas, `design.md` (big issues only) and `tasks.md`.
   - Each task carries its own verification. Tests and docs go in the task group that needs them, not in a final group.
2. **Screenshots for any UI change.** Prototype it locally (never committed), screenshot light, dark and 390px wide with Playwright, and commit the images under `openspec/changes/<id>/screenshots/`. Send them to David as files. Never send a path that only resolves inside the change folder.
3. **Estimate the remaining cost** in `tasks.md`: model calls, tokens and dollars to deploy, from current model pricing (the `claude-api` skill). Compare finishing in the current session with a fresh one.
4. **Grill** David on every call you made yourself (use the `grilling` skill if present). Ask one round at a time, number the questions, and give a recommended answer for each. Fold each answer into the artifacts as a numbered decision ("Agreed at spec").
5. `openspec validate <id> --strict` passes. Push the branch.

**Gate:** the grill has no open questions and David approves the spec. Post **"Spec: approved"** on the issue, listing the decisions, then move to `stage:implement`.

### 4. implement: build it

- Follow the `openspec-apply-change` skill. Write tests first, and tick each task only when its stated verification passes.
- If the spec turns out wrong while implementing (in #48, the daily Watchtower schedule broke the planned post-build check), pause. Say so, propose the change, and update the artifacts before carrying on.
- Run the repo's own checks. Bump the version with the repo's tool and add a CHANGELOG entry.
- Open **one PR** carrying the spec, code and tests. It follows the repo's PR template, says `Closes #N`, and states the merge order of any cross-repo PRs.
- Subscribe to the PR and drive CI to green.

**Gate:** all tasks except the deploy group are ticked, and CI is green. Move to `stage:test`.

### 5. test: prove the tests prove it

Run an independent test review with the `test-critic` agent. Give it the spec paths and code, not your conclusions. Classify each finding as ACCEPT, REJECT or DEFER, with a reason. For each accepted gap, write the test, then break the behaviour on purpose and confirm the test fails (then restore it). Post the classification on the issue.

**Gate:** every accepted finding is fixed, CI is green, and David says to move on. Move to `stage:review`.

### 6. review: iterate until it passes

Run the reviews in [REVIEWS.md](REVIEWS.md) in parallel. Fix what you accept, and send each review that found issues back for another round. Repeat until every review says PASS. Log them with the `record-review` skill.

**Gate:** every review passes and is logged. Then:
- Create the **follow-up issues** the requirements call for, in this repo or others, each at `stage:define` and linking this issue.
- Post **"Review: passed"** on the issue with the follow-up links, and move to `stage:deploy`.

### 7. deploy: merge, then confirm live

1. **David merges the PR**, which closes the issue.
2. **Automatic deploys** (Watchtower and similar) are confirmed by the repo's `verify-deploy` workflow. It checks that the public `/health` reports `main`'s version, and records the result as a GitHub `production` deployment.
   - "Deployed to production" on a commit means it is live. Merged work after the last successful production deployment is not yet live.
   - Apps without the check yet get it from their follow-up issue.
3. **Deploys that need manual infra work** (compose, a new service, DNS or SSO, secrets) get their own issue in `docker-infra` or `azure-infra`, with an open / running / closed workflow.
4. **Archive** the OpenSpec change with `openspec-archive-change`. This is a new PR, so it needs the repo's version bump.
5. **Compare the actual cost** with the spec's estimate. Get session costs from `/cost` or the usage page; David supplies them if no tool can. Record the comparison in `docs/reviews/LOG.md`.

**Done when:** the production deployment for the merge commit is successful, the change is archived, and the cost comparison is recorded.

## Cost

A long session costs more per call, and the cost jumps whenever the 1-hour prompt cache expires. Hand waits of an hour or more (merges, overnight deploys) to a fresh session, primed with the spec's `tasks.md` and the issue thread. Spawn review agents only at their stage, never "to be safe".
