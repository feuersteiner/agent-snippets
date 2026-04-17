# TESTS.md

TESTS.md is a repo-local testing layer for coding agents.

It takes what agents and reviewers learn during real work and turns it into runnable, fetchable tests. Instead of storing more memory, prompts, or vague “knowledge,” TESTS.md captures lessons as checks the agent can run on the next commit or PR.

The goal is simple: **learn once, test forever.**

---

## Why this exists

Coding agents repeat the same mistakes:

- they violate local conventions
- they miss architecture boundaries
- they forget reviewer feedback from the last PR
- they reintroduce patterns that were already fixed before

Most “memory” solutions try to solve this by adding more context.

People already dislike that approach because it creates:

- memory bloat
- stale notes
- noisy prompts
- lower trust in the system

TESTS.md takes a different approach.

Instead of asking the agent to remember everything, it asks:

> what did we learn that can now be tested?

---

## What TESTS.md is

TESTS.md is a format and workflow for turning repo-specific knowledge into runnable checks.

It works like `SKILL.md`, but for verification instead of procedure.

- `SKILL.md` tells the agent **how to do something**
- `TESTS.md` tells the agent **how to verify it did it correctly in this repo**

Each test file is small, explicit, and fetchable on demand.

The agent loads the relevant test files for the code it changed, then runs them one by one against the new commit or PR.

---

## Core idea

After each agent session, review, or bugfix, some lessons are worth preserving.

But those lessons should not become more prose.

They should become tests.

Examples:

- “API handlers must validate input before business logic”
- “Do not format dates outside the shared helper”
- “Files under billing cannot import UI-only utilities”
- “Webhook handlers must preserve idempotency”
- “Retry logic belongs in the service layer, not route handlers”

If a rule matters, the system should try to express it as something runnable.

---

## How it works

### 1. Agent changes code

The agent opens a PR or produces a commit.

### 2. Relevant TESTS.md files are fetched

Based on the changed files, task type, or repo area, the system selects the relevant test files.

Examples:

- `api/TESTS.md`
- `billing/TESTS.md`
- `ui/forms/TESTS.md`

### 3. Sub-agents run tests one by one

Each test can be run by a dedicated sub-agent or a lightweight execution step.

A test may use:

- regex/static checks
- repo scripts
- existing test frameworks
- readonly sandboxed analysis
- an LLM-based evaluator with explicit reasoning

### 4. Results are scored

Each test returns one of four outcomes:

- **failed** — the test was relevant and the change failed it
- **passed** — the test was relevant and the change passed it
- **warning** — the test was relevant and mostly passed, but surfaced issues worth fixing
- **irrelevant** — the test was not useful for this change

### 5. New lessons become new tests

After review or after a bug is fixed, new repo knowledge can be promoted into new test files or new test cases.

That is the knowledge transfer loop.

---

## Output model

At the end of a run, the result should look like this:

### Failed
The test was useful to the commit and it failed because of a specific issue.

Example:

- `failed` — `api/input-validation`
  - useful because this PR edited API handlers
  - failed because `api/orders/create.ts` accessed request data before schema validation

### Passed
The test was useful to the commit and it succeeded.

Example:

- `passed` — `billing/service-boundary`
  - useful because this PR changed retry logic
  - passed because retry behavior stayed inside the billing service layer

### Warning
The test was useful and passed with warnings the agent can still fix.

Example:

- `warning` — `ui/form-consistency`
  - useful because this PR added a new form
  - passed structurally, but warning: error copy does not match existing form patterns

### Irrelevant
The test was not useful for this change.

Example:

- `irrelevant` — `webhook-idempotency`
  - not applicable because this PR only changed UI files

---

## What makes this different

Most agent memory systems try to preserve everything.

TESTS.md tries to preserve only what can be checked.

That makes it:

- smaller
- more explicit
- more durable
- easier to trust
- less likely to bloat prompt context

The point is not to create a smarter memory layer.

The point is to create a better feedback loop between work, review, and verification.

---

## Test file shape

A TESTS.md file should be small and local to a repo area.

Example:

```md
---
name: api-handler-tests
description: Checks for API handler changes
apply_when:
  paths:
    - "api/**"
    - "src/routes/**"
owner: team-platform
---

# TESTS.md

## Input validation must happen first

- type: script
- run: node .tests/check-api-validation.js
- fail_if: request data is used before schema validation
- warning_if: validation exists but does not use the shared pattern
- fix_hint: use the shared validator from `api/_shared/validation.ts`

## No direct date formatting

- type: regex
- run: rg "toLocaleDateString|Intl.DateTimeFormat|format\\(" api/ src/
- fail_if: date formatting appears outside shared utilities
- fix_hint: use `src/lib/date.ts`

## Shared error shape

- type: llm
- instruction: verify that API handlers return the repo-standard error shape
- fail_if: response contract differs from existing handler patterns
- warning_if: contract is valid but wording differs from repo norms
