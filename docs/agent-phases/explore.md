# Explore phase

Argument: `<issue>`, the number of the issue being worked.

Investigate the issue and post the findings as a comment, for the design phase (a later, separate run) to consume. Investigation only — propose no approach, no plan, and change no code.

## 1. Check for existing research

Find any comment whose body starts with `<!-- agent:research -->` (`gh issue view <issue> --json comments -q '.comments[] | select(.body | startswith("<!-- agent:research -->")) | {id, body}'`). At most one should exist.

If found, this phase already ran for this issue — report `READY` with summary "research already exists" and stop without posting anything.

## 2. Research

Classify the issue as `Bug` or `Feature` first. If it's neither (e.g. a question, a discussion, an out-of-scope request), report `BLOCKED` with a one-line reason and stop — do not research further.

Otherwise investigate as deep as the issue warrants:

- Read the issue body and every comment.
- Read `CONTEXT.md` and `docs/adr/` at the repo root if they exist — they hold this project's domain vocabulary and past decisions; don't contradict a settled ADR without flagging it.
- Read the actual code the issue touches. If the repo has no code yet in the relevant area, say so plainly rather than inventing what "currently happens."
- Note anything that looks unsafe to build without a human call — a genuine product/business decision, not a technical detail you could reasonably decide yourself. That gets flagged in the design phase, not here; explore only surfaces the fact, it doesn't decide whether it's blocking.

If you hit something that makes the issue itself unworkable (contradicts a settled ADR with no reconciliation path, depends on infrastructure that doesn't exist and isn't this issue's job to create, etc.), report `BLOCKED` with a one-line reason and stop.

## 3. Post the findings

Comment on the issue:

```
<!-- agent:research -->
## Research

**Type:** Bug | Feature
**Summary:** ...
**Reproduction:** ...                    (Bug — steps to reproduce, expected vs. actual)
**Current vs. desired behavior:** ...    (Feature — use instead of Reproduction)
**Relevant code/docs:** ... (files, ADRs, CONTEXT.md terms this touches)
**Risks:** ... (what could break; "None identified" if none found)
```

Include exactly one of **Reproduction** or **Current vs. desired behavior**, matching the type. Never include a proposed approach or effort estimate — that's the design phase's job.

## Outcome

Report `READY` or `BLOCKED` plus a one-line summary. End your final message with exactly one line, verbatim:

`AGENT_OUTCOME: READY` or `AGENT_OUTCOME: BLOCKED`
