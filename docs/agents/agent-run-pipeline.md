# Agent run pipeline

How labeling an issue/PR here kicks off an agent run, and where each part of
that pipeline actually lives.

## Label state machine

`agent:explore` → `agent:design` → `agent:implement` → `agent:review`, each
gated by `agent:in-progress` (busy) and falling out to `agent:needs-human` on
`AGENT_OUTCOME: BLOCKED`. `implement` hands tracking from the issue to the PR
it opens; `review` reports `APPROVED`, `CHANGES_REQUESTED`, or
`CHANGES_REQUESTED_CAPPED`. The agent applies its own label transitions at
the end of each run (see `.github/workflows/agent-run-trigger.yml` in
`ajorquera/agent-runner` for the exact commands per outcome).

## Where the pieces live

- **This repo** (`ajorquera/new`): `.github/workflows/agent-run-trigger.yml`
  — just the `issues`/`pull_request` `labeled` trigger and the phase-label
  gate, calling into `ajorquera/agent-runner`'s reusable workflow.
- **`ajorquera/agent-runner`**: the actual decide/dispatch logic
  (`.github/workflows/agent-run-trigger.yml`, called via `workflow_call`) and
  the runner itself (`.github/workflows/agent-run.yml`) that clones this
  repo, runs Claude Code, and reports back. See its `docs/triggering.md` for
  the full calling contract.
- **This repo**: `docs/agent-phases/{explore,design,implement,review}.md` —
  the actual per-phase instructions the agent follows for this repo. If
  these ever went missing, the agent falls back to `agent-runner`'s generic
  `docs/default-phases/` stubs and flags that in its outcome comment.

## Why the trigger is split out

The decide/dispatch logic (parse the label, check for `agent:in-progress`,
build the task prompt, call `agent-runner`) used to be duplicated inline in
this repo's own trigger workflow. It moved to `agent-runner` as a reusable
`workflow_call` workflow so a second caller repo can pick it up by adding a
few lines rather than copy-pasting the whole thing — see
`ajorquera/agent-runner`'s PR history for the migration.
